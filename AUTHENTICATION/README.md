# AUTHENTICATION — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Authentication là gì & phân loại lỗ hổng](#1-authentication-là-gì--phân-loại-lỗ-hổng)
2. [Username enumeration](#2-username-enumeration)
3. [Brute-force & cách phòng chống bị bypass](#3-brute-force--cách-phòng-chống-bị-bypass)
4. [Lỗ hổng logic trong 2FA](#4-lỗ-hổng-logic-trong-2fa)
5. [Password reset / thay đổi mật khẩu bị lỗi logic](#5-password-reset--thay-đổi-mật-khẩu-bị-lỗi-logic)
6. [Stay-logged-in / "remember me" cookie](#6-stay-logged-in--remember-me-cookie)
7. [Offline password cracking](#7-offline-password-cracking)
8. [Cách phòng chống chung](#8-cách-phòng-chống-chung)
9. [Danh sách lab liên quan](#9-danh-sách-lab-liên-quan)

## 1. Authentication là gì & phân loại lỗ hổng

Authentication (xác thực) xác minh danh tính người dùng. Lỗ hổng authentication cho phép attacker bypass hoàn toàn bước xác thực hoặc đoán/brute-force được thông tin đăng nhập của người dùng khác. 3 nhóm chính:

- **Lỗi trong cơ chế xác thực** (logic flaw trong login, 2FA, "remember me"...).
- **Cơ chế brute-force protection yếu hoặc thiếu.**
- **Lỗi logic trong luồng phụ trợ** (quên mật khẩu, đổi mật khẩu).

## 2. Username enumeration

Xảy ra khi ứng dụng vô tình tiết lộ **username có tồn tại hay không** thông qua sự khác biệt trong response, dù không cố ý:

| Kỹ thuật phát hiện | Ví dụ |
|---|---|
| Thông báo lỗi khác nhau | "Invalid username" vs "Invalid password" |
| Thông báo lỗi khác nhau tinh vi (subtle) | khác 1 dấu chấm, khác độ dài response vài byte |
| Thời gian phản hồi khác nhau | username tồn tại → server tốn thời gian hash password → chậm hơn |
| Hành vi khoá tài khoản | chỉ tài khoản tồn tại mới có thể bị "lock" sau nhiều lần sai |
| HTTP status code khác nhau | 200 vs 302, hoặc số lượng response header khác nhau |

## 3. Brute-force & cách phòng chống bị bypass

Cơ chế phòng chống brute-force phổ biến: khoá theo IP, khoá theo tài khoản sau N lần sai. Các lỗi logic khiến cơ chế này bị vô hiệu hoá:

- **Khoá theo IP nhưng có "danh sách trắng"** (ví dụ bỏ qua khoá nếu request có header `X-Forwarded-For` hợp lệ) → attacker tự set header đó để "reset" bộ đếm.
- **Gửi nhiều cặp credential trong 1 request** (ví dụ submit array JSON `["pass1","pass2",...]` cho field password) — nếu backend xử lý toàn bộ mảng nhưng cơ chế đếm số lần thử chỉ tính là "1 request" → bypass rate limit.
- **Reset bộ đếm bằng cách "đăng nhập thành công" 1 lần bất kỳ** giữa các lần thử sai, nếu logic đếm sai reset theo login event thay vì theo tài khoản mục tiêu.

## 4. Lỗ hổng logic trong 2FA

- **2FA simple bypass:** sau khi qua bước 1 (đúng username/password), người dùng được redirect tới trang nhập mã 2FA — nhưng nếu server đã tạo **session đăng nhập đầy đủ quyền** ngay ở bước 1 (trước khi verify mã 2FA), attacker chỉ cần **điều hướng thẳng tới trang sau đăng nhập** (bỏ qua bước nhập mã) để vào tài khoản.
- **2FA broken logic:** endpoint verify mã 2FA nhận tham số username từ **request** (thay vì lấy từ session hiện tại) → attacker tự đổi tham số username thành nạn nhân, chỉ cần brute-force mã 2FA của họ (mã ngắn 4 số) mà không cần biết password nạn nhân.
- **2FA bypass using brute-force:** mã 2FA ngắn (4 chữ số) và không có rate-limit đúng cách → brute-force toàn bộ 10000 khả năng bằng Burp Intruder.

## 5. Password reset / thay đổi mật khẩu bị lỗi logic

- **Password reset broken logic:** luồng reset gồm nhiều bước (nhập email → xác nhận token → đặt mật khẩu mới); nếu bước đặt mật khẩu mới **không kiểm tra lại token có khớp với username** đang đặt hay không, attacker có thể tự đổi username trong request cuối để đổi mật khẩu tài khoản khác.
- **Password reset poisoning via middleware / Host header:** link reset password được server sinh ra dựa vào `Host` header của request → nếu attacker giả mạo `Host` (hoặc chèn `X-Forwarded-Host`), link reset gửi cho nạn nhân qua email sẽ trỏ về domain do attacker kiểm soát → khi nạn nhân click, token lộ cho attacker.
- **Password brute-force via password change:** chức năng đổi mật khẩu yêu cầu nhập password cũ — nếu endpoint này không có rate-limit như trang login, có thể brute-force password hiện tại qua đây.

## 6. Stay-logged-in / "remember me" cookie

Nếu giá trị cookie "remember me" được sinh ra theo công thức đoán được (ví dụ `md5(username:password)`, hoặc `base64(username:hardcoded_salt)`) → attacker có thể tự tính/brute-force giá trị cookie hợp lệ cho tài khoản nạn nhân mà không cần biết password thật.

## 7. Offline password cracking

Nếu attacker lấy được **hash** password (qua 1 lỗ hổng khác như XSS đọc localStorage, hay thông tin bị leak), có thể crack offline bằng công cụ như **hashcat**/John the Ripper với wordlist, không bị giới hạn bởi rate-limit của server.

## 8. Cách phòng chống chung

- Thông báo lỗi login **giống hệt nhau** dù sai username hay sai password, thời gian phản hồi đồng nhất (dummy hash nếu username không tồn tại).
- Rate-limit theo cả IP **và** tài khoản, không tin bất kỳ header client-controlled nào (`X-Forwarded-For`).
- Không tạo session đầy đủ quyền trước khi hoàn tất toàn bộ bước xác thực (kể cả 2FA).
- Token reset password phải gắn chặt với đúng user, có thời hạn ngắn, single-use, và không dựa vào `Host` header để build link — dùng domain cấu hình cứng phía server.
- Cookie "remember me" phải dùng giá trị ngẫu nhiên không đoán được (không tự sinh theo công thức từ username/password), lưu trạng thái phía server (session token), có thể thu hồi.

## 9. Danh sách lab liên quan

1. Username enumeration via different responses
2. 2FA simple bypass
3. Password reset broken logic
4. Username enumeration via subtly different responses
5. Username enumeration via response timing
6. Broken brute-force protection, IP block
7. Username enumeration via account lock
8. 2FA broken logic
9. Brute-forcing a stay-logged-in cookie
10. Offline password cracking
11. Password reset poisoning via middleware
12. Password brute-force via password change
13. Broken brute-force protection, multiple credentials per request
14. 2FA bypass using a brute-force attack

