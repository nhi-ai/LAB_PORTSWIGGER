# OAUTH AUTHENTICATION — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [OAuth 2.0 là gì?](#1-oauth-20-là-gì)
2. [Luồng Authorization Code & Implicit](#2-luồng-authorization-code--implicit)
3. [Các lỗ hổng OAuth thường gặp](#3-các-lỗ-hổng-oauth-thường-gặp)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Danh sách lab liên quan](#5-danh-sách-lab-liên-quan)

## 1. OAuth 2.0 là gì?

OAuth 2.0 là chuẩn uỷ quyền (authorization) cho phép ứng dụng bên thứ 3 (client) truy cập tài nguyên của người dùng trên 1 dịch vụ khác (ví dụ "Login with Google") mà không cần biết mật khẩu người dùng. Các thành phần chính: **Resource Owner** (người dùng), **Client** (ứng dụng bên thứ 3), **Authorization Server** (nơi cấp token, ví dụ Google), **Resource Server** (API chứa dữ liệu người dùng).

## 2. Luồng Authorization Code & Implicit

**Authorization Code flow (phổ biến, an toàn hơn):**
1. Client redirect user tới Authorization Server: `GET /authorize?client_id=...&redirect_uri=...&response_type=code&scope=...`
2. User đăng nhập & đồng ý cấp quyền.
3. Authorization Server redirect về `redirect_uri` kèm `code`.
4. Client (server-side) đổi `code` lấy `access_token` qua request `POST /token` (kèm `client_secret`).

**Implicit flow (kém an toàn hơn, ít dùng hiện nay):** Authorization Server trả thẳng `access_token` qua URL fragment ngay ở bước redirect, không qua bước đổi code — token lộ trực tiếp trong URL (dễ bị log/leak qua Referer/history).

## 3. Các lỗ hổng OAuth thường gặp

| Lỗ hổng | Mô tả |
|---|---|
| **`redirect_uri` không validate chặt** | Nếu Authorization Server chỉ kiểm tra `redirect_uri` "chứa" domain hợp lệ (không kiểm tra chính xác/whitelist đầy đủ), attacker đổi `redirect_uri` trỏ về domain mình → `code`/`token` bị gửi thẳng cho attacker |
| **Thiếu `state` parameter (CSRF trong OAuth)** | `state` dùng để chống CSRF trong luồng OAuth; nếu thiếu, attacker có thể ép nạn nhân hoàn tất luồng OAuth bằng **tài khoản của attacker**, khiến tài khoản nạn nhân trên client app bị "liên kết" (linked) với tài khoản OAuth của attacker — dẫn tới account takeover khi hệ thống login bằng cách match theo email từ token |
| **Force OAuth linking** | Tương tự trên — dùng CSRF ép nạn nhân tự liên kết tài khoản SSO của attacker vào tài khoản đang đăng nhập của nạn nhân, sau đó attacker dùng chính tài khoản SSO của mình để đăng nhập vào tài khoản nạn nhân |
| **Leo thang từ `code` bị lộ (qua Referer/log)** | Nếu trang sau khi nhận `code` có load thêm resource từ domain thứ 3 (ảnh, script), `code` trong URL có thể bị lộ qua header `Referer` gửi kèm request đó |
| **SSRF qua `redirect_uri` do OAuth server tự fetch metadata** | 1 số OAuth Server hỗ trợ "dynamic client registration", tự fetch thông tin metadata client từ URL do client cung cấp → có thể lợi dụng để SSRF |
| **Access token không giới hạn scope đúng cách** | Token cấp quyền rộng hơn cần thiết, hoặc không kiểm tra `aud` (audience) khiến token cấp cho ứng dụng A dùng được cho ứng dụng B |
| **Password reset/login qua broken logic khi kết hợp SSO và login thường** | Ứng dụng hỗ trợ cả login thường (password) và SSO cho cùng tài khoản (match theo email) — nếu không kiểm tra kỹ, attacker có thể tạo tài khoản SSO trùng email với nạn nhân (nếu OAuth provider không xác thực email) để chiếm quyền |

## 4. Cách phòng chống

- Validate `redirect_uri` theo **whitelist chính xác tuyệt đối** (exact match), không dùng kiểm tra "contains"/prefix lỏng lẻo.
- Luôn dùng `state` parameter ngẫu nhiên, gắn với session, verify khi nhận callback.
- Không tin `email` từ token OAuth để tự động match/merge tài khoản trừ khi provider xác nhận email đã verified (`email_verified: true`).
- access_token nên có `aud` rõ ràng, resource server phải verify đúng audience trước khi chấp nhận.
- Đổi `code` lấy token nên yêu cầu xác thực `client_secret` (chỉ làm được ở server-side), tránh dùng Implicit flow khi có thể dùng Authorization Code + PKCE.

## 5. Danh sách lab liên quan

1. Authentication bypass via OAuth implicit flow
2. Forced OAuth profile linking
3. OAuth account hijacking via redirect_uri
4. Stealing OAuth access tokens via an open redirect
5. SSRF via OpenID dynamic client registration
6. Stealing OAuth access tokens via a proxy page

