# CLICKJACKING — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Clickjacking là gì?](#1-clickjacking-là-gì)
2. [Cơ chế tấn công cơ bản](#2-cơ-chế-tấn-công-cơ-bản)
3. [Các biến thể nâng cao](#3-các-biến-thể-nâng-cao)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Danh sách lab liên quan](#5-danh-sách-lab-liên-quan)

## 1. Clickjacking là gì?

Clickjacking (UI redress attack) là kỹ thuật lừa nạn nhân **click vào 1 phần tử trên trang mục tiêu mà không hề hay biết**, bằng cách nhúng trang đó trong 1 `<iframe>` vô hình (opacity gần 0) đặt chồng lên 1 giao diện mồi hấp dẫn (ví dụ "Click to win prize"). Click của nạn nhân thực chất rơi vào nút/link thật trên trang bị nhúng.

## 2. Cơ chế tấn công cơ bản

```html
<style>
  iframe {
    position: relative;
    width: 500px;
    height: 700px;
    opacity: 0.0001;
    z-index: 2;
  }
  div {
    position: absolute;
    top: 300px;
    left: 60px;
    z-index: 1;
  }
</style>
<div>Click me</div>
<iframe src="https://victim-website.com/account"></iframe>
```

Nạn nhân thấy chữ "Click me" nhưng thực tế click xuyên qua vào nút thật (ví dụ "Delete account") nằm ngay bên dưới trong iframe vô hình, đã được căn chỉnh trùng vị trí.

## 3. Các biến thể nâng cao

| Kỹ thuật | Mô tả |
|---|---|
| **CSRF token vẫn hợp lệ** | Vì request click thật gửi từ chính trang victim (trong iframe), CSRF token hợp lệ vẫn được gửi kèm → clickjacking **bypass được CSRF token protection thông thường** |
| **Prefilled input từ URL param** | Nếu trang victim cho phép prefill giá trị form qua query string, attacker vừa kiểm soát nội dung vừa kiểm soát click → không cần nạn nhân tự nhập gì |
| **Frame buster script bị bypass** | JS chống nhúng iframe (`if (top != self) top.location = self.location`) có thể bị vô hiệu hoá bằng thuộc tính `sandbox` trên iframe (chặn JS bên trong thực thi điều hướng) |
| **Kết hợp DOM XSS** | Dụ nạn nhân click vào vị trí kích hoạt 1 DOM XSS sẵn có (ví dụ link chứa `javascript:` hoặc phần tử có `onclick`) mà nạn nhân không biết mình đang tương tác với trang đó |
| **Multistep clickjacking** | Dàn dựng chuỗi nhiều bước click liên tiếp (mỗi bước là 1 iframe/overlay khác nhau) để hoàn tất 1 luồng nhiều bước (ví dụ xác nhận 2 lần) mà nạn nhân tưởng đang chơi game/click 1 nút duy nhất |

## 4. Cách phòng chống

- **X-Frame-Options** header:
  ```
  X-Frame-Options: DENY
  X-Frame-Options: SAMEORIGIN
  ```
- **Content-Security-Policy** hiện đại hơn (override được `X-Frame-Options` trên trình duyệt hỗ trợ):
  ```
  Content-Security-Policy: frame-ancestors 'self';
  ```
- Frame-busting script chỉ nên là lớp phòng thủ bổ sung, **không thay thế** cho header trên (vì dễ bypass bằng `sandbox` attribute hoặc disable JS).
- Với hành động nhạy cảm, có thể yêu cầu xác nhận bổ sung (ví dụ nhập lại mật khẩu) để giảm thiệt hại nếu bị clickjack.

## 5. Danh sách lab liên quan

1. Basic clickjacking with CSRF token protection
2. Clickjacking with form input data prefilled from a URL parameter
3. Clickjacking with a frame buster script
4. Exploiting clickjacking vulnerability to trigger DOM-based XSS
5. Multistep clickjacking

