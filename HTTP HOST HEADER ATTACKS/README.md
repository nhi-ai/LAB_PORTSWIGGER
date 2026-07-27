# HTTP HOST HEADER ATTACKS — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Host header là gì & vì sao ứng dụng tin tưởng nó](#1-host-header-là-gì--vì-sao-ứng-dụng-tin-tưởng-nó)
2. [Password reset poisoning](#2-password-reset-poisoning)
3. [Host header authentication bypass](#3-host-header-authentication-bypass)
4. [Web cache poisoning qua Host header](#4-web-cache-poisoning-qua-host-header)
5. [SSRF qua Host header (routing-based & parsing)](#5-ssrf-qua-host-header-routing-based--parsing)
6. [Bypass validate Host qua connection state](#6-bypass-validate-host-qua-connection-state)
7. [Cách phòng chống](#7-cách-phòng-chống)
8. [Danh sách lab liên quan](#8-danh-sách-lab-liên-quan)

## 1. Host header là gì & vì sao ứng dụng tin tưởng nó

Header `Host` cho server biết tên miền client muốn truy cập (dùng cho virtual hosting nhiều domain trên 1 IP). Nhiều ứng dụng **tin tưởng header này** để tự build absolute URL (link reset password, redirect...) hoặc dùng nó để routing nội bộ — nhưng đây là header hoàn toàn do **client kiểm soát**, dễ giả mạo.

## 2. Password reset poisoning

Đã đề cập ở topic Authentication — server dùng `Host` (hoặc `X-Forwarded-Host`) để build link reset password gửi qua email. Giả mạo header này khiến link trỏ về domain attacker, đánh cắp token khi nạn nhân click.

## 3. Host header authentication bypass

Một số ứng dụng có logic "nội bộ tin tưởng" dựa trên `Host`, ví dụ: nếu `Host: localhost` hoặc `Host: internal-admin.company.local` thì tự động coi request đến từ mạng nội bộ, cấp quyền admin mà không cần xác thực thêm. Attacker chỉ cần gửi request từ ngoài internet nhưng đặt `Host` header giả thành giá trị "nội bộ" đó để bypass.

## 4. Web cache poisoning qua Host header

Nếu cache dùng `Host` làm 1 phần cache key nhưng ứng dụng backend xử lý `Host` không nhất quán (ví dụ có 1 header khác override `Host` thực tế dùng để build nội dung trang, như `X-Forwarded-Host`) → attacker có thể đầu độc cache bằng response chứa nội dung độc hại (link script trỏ domain attacker) được lưu và phục vụ cho user khác. (Chi tiết sâu hơn ở topic Web Cache Poisoning.)

## 5. SSRF qua Host header (routing-based & parsing)

- **Routing-based SSRF:** Trong kiến trúc có front-end + back-end server, front-end định tuyến request theo path nhưng back-end dùng `Host` header để xác định server đích khi tự gọi request nội bộ khác (ví dụ microservice) → giả mạo `Host` để chuyển hướng request nội bộ tới địa chỉ do attacker chọn (bao gồm cả địa chỉ nội bộ như `192.168.0.1`, `localhost`).
- **SSRF via flawed request parsing:** Front-end và back-end **phân tích (parse)** dòng đầu request/`Host` header khác nhau (ví dụ có khoảng trắng thừa, ký tự lạ) dẫn đến hiểu sai đích đến khác nhau giữa 2 tầng — lợi dụng sự khác biệt này để "tuồn" 1 request nội bộ đi nơi khác so với những gì front-end tưởng đang gửi.

## 6. Bypass validate Host qua connection state

Một số server dùng **kết nối TCP đã xác thực Host hợp lệ ở request đầu** để "tin tưởng" mọi request tiếp theo trên cùng kết nối đó (connection reuse/keep-alive), bất kể `Host` header của các request sau có hợp lệ hay không. Attacker gửi request đầu tiên với `Host` hợp lệ, sau đó **trên cùng kết nối TCP** (không mở kết nối mới), gửi tiếp request thứ 2 với `Host` giả mạo — server không re-validate vì tin tưởng theo trạng thái kết nối.

## 7. Cách phòng chống

- Không dùng `Host` header để build absolute URL — cấu hình cứng domain chính thức phía server (whitelist).
- Nếu bắt buộc phải dùng `Host` cho mục đích routing, validate chặt theo whitelist domain hợp lệ, từ chối mọi giá trị lạ.
- Không tin `X-Forwarded-Host` trừ khi được set bởi proxy tin cậy nội bộ (và tầng ngoài cùng phải loại bỏ header này nếu client tự gửi).
- Validate `Host` header cho **từng request** trên kết nối, không tin tưởng theo trạng thái kết nối TCP đã xác thực trước đó.
- Đảm bảo front-end và back-end parse request theo cùng 1 chuẩn nhất quán (tránh discrepancy dẫn tới request smuggling/SSRF).

## 8. Danh sách lab liên quan

1. Basic password reset poisoning
2. Host header authentication bypass
3. Web cache poisoning via ambiguous requests
4. Routing-based SSRF
5. SSRF via flawed request parsing
6. Host validation bypass via connection state attack
7. Password reset poisoning via dangling markup

