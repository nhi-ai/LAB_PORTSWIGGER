# SSRF (SERVER-SIDE REQUEST FORGERY) — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [SSRF là gì?](#1-ssrf-là-gì)
2. [SSRF nhắm tới hệ thống nội bộ (không whitelist)](#2-ssrf-nhắm-tới-hệ-thống-nội-bộ-không-whitelist)
3. [Bypass whitelist/blacklist domain](#3-bypass-whitelistblacklist-domain)
4. [SSRF qua chuỗi lỗ hổng khác (chained)](#4-ssrf-qua-chuỗi-lỗ-hổng-khác-chained)
5. [Blind SSRF](#5-blind-ssrf)
6. [Cách phòng chống](#6-cách-phòng-chống)
7. [Lab step-by-step](#7-lab-step-by-step)

## 1. SSRF là gì?

Xảy ra khi ứng dụng cho phép attacker điều khiển URL/địa chỉ mà **server tự gửi request tới** (ví dụ chức năng "check stock từ URL", "tải ảnh từ link", webhook...), cho phép attacker khiến server gửi request tới nơi mình không được phép truy cập trực tiếp — bao gồm dịch vụ nội bộ (không public), metadata endpoint của cloud provider, hoặc chính localhost.

## 2. SSRF nhắm tới hệ thống nội bộ (không whitelist)

```
POST /product/stock
stockApi=http://192.168.0.1:8080/admin
```
Nếu không có bất kỳ kiểm soát nào, server tự động gửi request tới địa chỉ nội bộ này và trả kết quả về cho attacker — lộ thông tin/hành động trên hệ thống nội bộ mà bình thường không truy cập được từ internet.

**Mục tiêu phổ biến:** cloud metadata endpoint `http://169.254.169.254/latest/meta-data/` (AWS) — thường chứa credential tạm thời của instance.

## 3. Bypass whitelist/blacklist domain

| Cơ chế chặn | Cách bypass |
|---|---|
| Blacklist `localhost`/`127.0.0.1` | Dùng biến thể: `http://127.1`, `http://0.0.0.0`, `http://[::1]`, hoặc số nguyên tương đương IP (`http://2130706433/` = `127.0.0.1` dạng decimal) |
| Whitelist chỉ cho phép domain cụ thể | Dùng credential trong URL để "đánh lừa" parser: `https://expected-host@attacker.com` (nhiều parser hiểu domain là phần sau `@`, nhưng số khác hiểu nhầm là trước `@`) |
| Whitelist theo domain nhưng không theo path/subdomain | Dùng subdomain do attacker đăng ký trỏ về IP nội bộ, hoặc dùng open redirect trên chính domain whitelist để redirect sang địa chỉ nội bộ |
| Blacklist IP nội bộ theo chuỗi số | Dùng DNS rebinding: đăng ký domain có TTL thấp, ban đầu trỏ về IP hợp lệ (qua được validate), sau đó đổi DNS trỏ về IP nội bộ (server request thực tế xảy ra sau khi DNS đã đổi) |
| Chặn theo scheme `http/https` | Thử scheme khác nếu thư viện HTTP client hỗ trợ (`file://`, `gopher://`, `dict://`) để mở rộng khả năng khai thác (đọc file local, giao tiếp thô với service khác) |

## 4. SSRF qua chuỗi lỗ hổng khác (chained)

- **Kết hợp open redirect:** nếu whitelist chỉ validate URL ban đầu (domain hợp lệ), nhưng chức năng tự động **follow redirect**, dùng 1 open redirect trên domain hợp lệ đó để redirect request thực tế sang địa chỉ nội bộ:
  ```
  stockApi=https://weliketoshop.net/product/nextProduct?path=http://192.168.0.1/admin
  ```
- **Kết hợp XSS/HTML injection để dụ chính server** (ít gặp hơn, thường áp dụng khi có chức năng "render PDF từ HTML"/screenshot server-side sử dụng chính nội dung do user cung cấp).

## 5. Blind SSRF

Response không trả về nội dung request nội bộ (không thấy trực tiếp), nhưng vẫn xác nhận được SSRF tồn tại qua **out-of-band interaction** (Burp Collaborator) tương tự các loại blind injection khác — ví dụ chèn URL trỏ về Collaborator vào 1 header như `Referer` hoặc `X-Forwarded-For` mà server có thể dùng để tự gọi request theo dõi/logging nội bộ.

## 6. Cách phòng chống

- Whitelist domain/IP đích được phép cho mọi chức năng server tự gửi request (deny-by-default).
- Không tự động follow redirect, hoặc nếu follow thì validate lại URL đích sau mỗi lần redirect.
- Disable hỗ trợ scheme không cần thiết (`file://`, `gopher://`) trong HTTP client dùng nội bộ.
- Với dịch vụ cloud, hạn chế IMDSv1 (metadata endpoint không cần token), chuyển sang IMDSv2 (yêu cầu token qua PUT request trước, khó khai thác qua SSRF GET đơn thuần).
- Tách mạng: service xử lý request bên ngoài không nên có quyền truy cập trực tiếp vào mạng nội bộ nhạy cảm (network segmentation).

## 7. Lab step-by-step

### Lab 1: Basic SSRF against the local server
**Mục tiêu:** Truy cập trang admin nội bộ chạy trên `localhost` của chính server.
- **B1:** Vào chức năng "Check stock", dùng Burp bắt request `POST /product/stock` chứa tham số `stockApi=http://stock.weliketoshop.net:8080/...`.
- **B2:** Sửa `stockApi=http://localhost/admin`.
- **B3:** Response trả về nội dung trang admin nội bộ → tìm chức năng xoá user trong đó, gọi tiếp `stockApi=http://localhost/admin/delete?username=carlos` → lab solved.

### Lab 2: Basic SSRF against another back-end system
**Mục tiêu:** Dò dải mạng nội bộ để tìm server admin khác (không phải localhost).
- **B1:** Sửa `stockApi=http://192.168.0.1:8080/admin`, thử lần lượt các IP trong dải `192.168.0.X` (dùng Burp Intruder brute-force octet cuối).
- **B2:** Tìm được IP trả về trang admin hợp lệ (ví dụ `192.168.0.5:8080`).
- **B3:** Gọi tiếp chức năng xoá `carlos` qua địa chỉ đó → lab solved.

### Lab 3: SSRF with blacklist-based input filter
**Mục tiêu:** Server chặn chuỗi `localhost` — bypass bằng biến thể địa chỉ.
- **B1:** Thử `stockApi=http://localhost/admin` → bị chặn.
- **B2:** Thử các biến thể: `http://127.0.0.1/admin`, `http://127.1/admin`, `http://2130706433/admin` (decimal), `http://0177.0.0.1/admin` (octal), hoặc thêm ký tự thừa `http://localhost.` (dấu chấm cuối) — server chỉ so khớp chuỗi chính xác "localhost" nên các biến thể trỏ cùng địa chỉ nhưng khác chuỗi sẽ bypass.
- **B3:** Tìm biến thể hoạt động, tiếp tục gọi chức năng xoá `carlos` → lab solved.

### Lab 4: SSRF with filter bypass via open redirection vulnerability
**Mục tiêu:** Whitelist chỉ chấp nhận domain hợp lệ — dùng open redirect trên chính domain đó.
- **B1:** Tìm 1 endpoint open redirect trên domain whitelist, ví dụ `/product/nextProduct?path=`.
- **B2:** Sửa `stockApi=http://weliketoshop.net/product/nextProduct?path=http://192.168.0.12:8080/admin`.
- **B3:** Server validate domain ban đầu hợp lệ (weliketoshop.net), nhưng tự động follow redirect tới địa chỉ nội bộ → truy cập được trang admin.
- **B4:** Gọi tiếp chức năng xoá `carlos` qua địa chỉ nội bộ đã lộ → lab solved.

### Lab 5: Blind SSRF with out-of-band detection
**Mục tiêu:** Xác nhận blind SSRF qua header `Referer` mà server dùng để gọi analytics nội bộ.
- **B1:** Lấy payload Collaborator.
- **B2:** Gửi bất kỳ request nào tới ứng dụng kèm header:
  ```
  Referer: https://abcxyz.oastify.com
  ```
- **B3:** Poll Collaborator → nếu ghi nhận HTTP interaction từ server ứng dụng (do backend tự gọi request theo dõi referrer tới URL này) → xác nhận blind SSRF tồn tại → lab solved.

### Lab 6: SSRF with whitelist-based input filter
**Mục tiêu:** Whitelist chỉ cho phép domain cụ thể — bypass bằng kỹ thuật `@` trong URL.
- **B1:** Thử `stockApi=http://192.168.0.1/admin` → bị chặn (không khớp whitelist domain hợp lệ `stock.weliketoshop.net`).
- **B2:** Dùng cú pháp userinfo để đánh lừa parser (server chỉ check có chứa domain hợp lệ, không parse đúng chuẩn URL):
  ```
  stockApi=http://stock.weliketoshop.net@192.168.0.1/admin
  ```
- **B3:** Parser HTTP client thực tế coi phần sau `@` (`192.168.0.1`) mới là host thật, trong khi bộ lọc chỉ nhìn thấy `stock.weliketoshop.net` ở đầu chuỗi → bypass thành công.
- **B4:** Gọi tiếp chức năng xoá `carlos` → lab solved.

### Lab 7: SSRF with filter bypass via DNS rebinding
**Mục tiêu:** Server validate DNS resolve ra IP hợp lệ tại thời điểm check, nhưng request thực tế xảy ra sau đó — dùng DNS rebinding.
- **B1:** Dùng dịch vụ hỗ trợ DNS rebinding có sẵn của PortSwigger (thường tích hợp trong giao diện lab, cho phép đăng ký domain rebinding tự động đổi giữa 2 IP: 1 IP hợp lệ để pass validate, 1 IP nội bộ để request thật trỏ tới).
- **B2:** Cấu hình domain rebinding: IP thứ nhất = địa chỉnh public hợp lệ (để qua bước validate DNS của server, nếu có), IP thứ hai = `192.168.0.X` (địa chỉ admin nội bộ cần truy cập).
- **B3:** Gửi `stockApi=http://<rebinding-domain>/admin` — do TTL DNS thấp và có độ trễ giữa bước validate và bước request thật, lần resolve DNS thứ 2 (khi server thực sự gửi request) trả về IP nội bộ.
- **B4:** Request thật được gửi tới địa chỉ nội bộ, response trả về trang admin → gọi tiếp chức năng xoá `carlos` → lab solved.

## Tổng kết
- Luôn thử: localhost, các biến thể encode IP loopback, dò dải mạng nội bộ phổ biến (`192.168.0.X`, `10.0.0.X`), và kỹ thuật `@`/redirect/DNS rebinding khi gặp validate domain.
- Blind SSRF luôn cần Burp Collaborator để xác nhận khi không có phản hồi trực tiếp.
- SSRF thường là bước đệm để truy cập hệ thống nội bộ — kết hợp tốt với kiến thức Access Control và Host Header Attacks.

