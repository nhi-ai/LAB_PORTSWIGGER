# NOSQL INJECTION — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [NoSQL injection là gì?](#1-nosql-injection-là-gì)
2. [Kỹ thuật phát hiện & khai thác](#2-kỹ-thuật-phát-hiện--khai-thác)
3. [Cách phòng chống](#3-cách-phòng-chống)
4. [Lab step-by-step](#4-lab-step-by-step)

## 1. NoSQL injection là gì?

Tương tự SQL injection nhưng nhắm vào cơ sở dữ liệu NoSQL (MongoDB là phổ biến nhất trong các lab). Do NoSQL thường dùng cú pháp truy vấn dạng object/JSON (không phải chuỗi SQL), kỹ thuật injection cũng khác: thay vì thoát chuỗi bằng `'`, attacker chèn **toán tử NoSQL** (`$where`, `$ne`, `$gt`, `$regex`...) vào tham số để thay đổi logic truy vấn.

## 2. Kỹ thuật phát hiện & khai thác

**Phát hiện cơ bản:** thử ký tự đặc biệt của MongoDB (`'`, `"`, `\`, `;`, `{`, `}`) xem có gây lỗi cú pháp không, tương tự bước đầu test SQLi.

**Bypass login bằng toán tử `$ne` (not equal) qua JSON body:**
```json
{"username": {"$ne": ""}, "password": {"$ne": ""}}
```
Nếu backend query MongoDB kiểu `db.users.find({username: username, password: password})` với input JSON được nhận trực tiếp từ client (không ép kiểu string), gửi object `{"$ne":""}` khiến điều kiện luôn đúng với **bất kỳ user nào có password khác rỗng** → login bypass.

**Bypass qua query string (khi backend parse `param[$ne]=` thành object):**
```
POST /login
username=administrator&password[$ne]=x
```
Nhiều framework tự động parse cú pháp `password[$ne]=x` trong body `application/x-www-form-urlencoded` thành object `{"$ne":"x"}` → tương tự trên.

**Trích xuất dữ liệu từng ký tự bằng `$regex` (boolean-based blind):**
```json
{"username":"administrator","password":{"$regex":"^a"}}
```
Nếu response khác nhau tuỳ regex khớp hay không → dò từng ký tự password của `administrator` bằng cách thử lần lượt regex `^a`, `^b`, ... rồi mở rộng `^ab`, `^ac`... (tương tự binary/linear search ký tự như blind SQLi).

**`$where` injection (thực thi JavaScript phía DB):** MongoDB hỗ trợ toán tử `$where` nhận 1 đoạn code JavaScript chạy trực tiếp trong ngữ cảnh DB — nếu input được chèn vào đây, attacker có thể chạy code JS tuỳ ý:
```
username[$where]=this.username == this.username && (function(){ /* code */ return true;}())
```
Dùng kỹ thuật tương tự conditional/time-based blind (như blind SQLi) khi kết hợp `$where` với hàm `sleep()`của JS để suy luận dữ liệu nếu không có phản hồi trực tiếp.

**Operator injection để bypass danh sách trắng validate đơn giản:** nếu server chỉ kiểm tra input "không chứa các ký tự nguy hiểm cơ bản" nhưng chấp nhận cấu trúc object lồng qua tham số dạng mảng (`param[$operator]=value`), có thể chèn hầu hết mọi toán tử MongoDB mà không cần bất kỳ ký tự đặc biệt nào (`{`, `}`, `'`) — do cấu trúc được framework tự dựng thành object từ cú pháp key ngoặc vuông.

## 3. Cách phòng chống

- Luôn ép kiểu dữ liệu input (string) trước khi đưa vào truy vấn, không cho phép client gửi object/array cho các field lẽ ra chỉ nhận string đơn giản.
- Whitelist cấu trúc input mong đợi (dùng schema validation như JSON Schema) thay vì chỉ lọc ký tự.
- Tắt/hạn chế toán tử `$where` (thực thi JS) trong MongoDB nếu không thực sự cần.
- Áp dụng nguyên tắc least privilege cho tài khoản DB, giới hạn field/collection truy cập được.

## 4. Lab step-by-step

### Lab 1: Detecting NoSQL injection
**Mục tiêu:** Xác nhận điểm injection NoSQL tồn tại trên tham số tìm kiếm sản phẩm.
- **B1:** Gửi ký tự đặc biệt `'` vào tham số `category`, quan sát lỗi cú pháp MongoDB xuất hiện trong response (khác với lỗi 404 thông thường).
- **B2:** Gửi thêm `[$ne]=invalid` vào tham số để xác nhận server parse được cú pháp operator: `category[$ne]=abc` → nếu trả về toàn bộ sản phẩm (không lọc theo category cụ thể) → xác nhận NoSQL injection.
- **B3:** Ghi nhận điểm injection đã xác nhận → lab solved.

### Lab 2: Exploiting NoSQL operator injection to bypass authentication
**Mục tiêu:** Đăng nhập vào tài khoản `administrator` không cần biết password.
- **B1:** Quan sát request login dạng `application/x-www-form-urlencoded`: `username=administrator&password=x`.
- **B2:** Sửa tham số password thành cú pháp operator: `username=administrator&password[$ne]=invalidpassword`.
- **B3:** Server parse thành `{"$ne":"invalidpassword"}`, điều kiện `password != "invalidpassword"` luôn đúng với password thật của administrator → login thành công mà không cần biết password → lab solved.

### Lab 3: Exploiting NoSQL operator injection to extract unknown fields
**Mục tiêu:** Dùng `$regex` để trích xuất giá trị field không biết trước (ví dụ mã giảm giá bí mật) từng ký tự một.
- **B1:** Xác định endpoint áp dụng mã giảm giá, có field `couponCode` được kiểm tra khớp trong DB.
- **B2:** Gửi payload dùng `$regex` để dò ký tự đầu tiên:
  ```
  couponCode[$regex]=^a.*
  ```
  Lặp qua bảng chữ cái + số, quan sát response (mã hợp lệ hay không) để xác định ký tự đúng.
- **B3:** Mở rộng dần: `^ab.*`, `^ac.*`... cho tới khi xác định đủ toàn bộ chuỗi coupon code.
- **B4:** Dùng mã coupon đầy đủ vừa trích xuất được để áp dụng giảm giá → lab solved.

### Lab 4: Exploiting NoSQL injection to extract data
**Mục tiêu:** Trích xuất password của user `administrator` bằng kỹ thuật blind injection kết hợp `$where`/`$regex`.
- **B1:** Xác định điểm injection trong request tìm kiếm hoặc login chấp nhận operator NoSQL.
- **B2:** Dò độ dài password bằng payload dạng biểu thức so sánh độ dài chuỗi (nếu hỗ trợ `$where`):
  ```
  username=administrator&password[$where]=this.password.length == 8
  ```
  Tăng dần giá trị cho tới khi response cho thấy điều kiện đúng (ví dụ login thành công hoặc thông báo khác biệt) → xác định đúng độ dài.
- **B3:** Dò từng ký tự bằng `$regex` (tương tự Lab 3) trên field `password`:
  ```
  username=administrator&password[$regex]=^s.*
  ```
  Lặp lại cho từng vị trí ký tự cho tới khi ghép đủ toàn bộ password.
- **B4:** Đăng nhập bằng `administrator` + password vừa trích xuất được → lab solved.

## Tổng kết
- Luôn thử cú pháp `param[$operator]=value` (đặc biệt `$ne`, `$gt`, `$regex`, `$where`) khi nghi ngờ backend dùng MongoDB.
- `$regex` là công cụ mạnh nhất để trích xuất dữ liệu blind theo từng ký tự, tương tự `SUBSTRING` trong blind SQLi.
- Không có ký tự đặc biệt nào bắt buộc phải gửi (`'`, `"`) nếu framework tự dựng object từ cú pháp `param[key]=value` — đây là điểm khác biệt lớn so với SQL injection truyền thống.

