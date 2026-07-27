# RACE CONDITIONS — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [Race condition là gì?](#1-race-condition-là-gì)
2. [Kỹ thuật gửi request đồng thời (single-packet attack)](#2-kỹ-thuật-gửi-request-đồng-thời-single-packet-attack)
3. [Các dạng race condition thường gặp](#3-các-dạng-race-condition-thường-gặp)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Lab step-by-step](#5-lab-step-by-step)

## 1. Race condition là gì?

Xảy ra khi 2 hay nhiều luồng xử lý (request) truy cập/thay đổi cùng 1 tài nguyên **gần như đồng thời**, và logic ứng dụng không xử lý đúng trường hợp này (thiếu khoá/transaction) — dẫn tới kết quả không nhất quán mà attacker có thể lợi dụng (ví dụ áp mã giảm giá 2 lần, redeem voucher nhiều lần, vượt giới hạn số lần thử).

## 2. Kỹ thuật gửi request đồng thời (single-packet attack)

Gửi request tuần tự thông thường (dù rất nhanh) vẫn có độ trễ đủ để server xử lý xong request trước khi nhận request sau — không tạo được race window thực sự. Kỹ thuật hiệu quả:

- **Burp Repeater "Send group in parallel":** nhóm nhiều tab request, chọn gửi song song — Burp cố gắng đồng bộ hoá thời điểm gửi.
- **Single-packet attack (HTTP/2):** kỹ thuật nâng cao dùng đặc tính HTTP/2 cho phép gửi nhiều request trong **cùng 1 gói TCP** (single packet) tới server, loại bỏ hoàn toàn độ trễ network jitter giữa các request — tối đa hoá khả năng trúng đúng race window cực hẹp. Burp Repeater hỗ trợ sẵn tính năng này khi gửi group request qua HTTP/2.
- **Burp Turbo Intruder:** viết script Python nhỏ, dùng `engine=Engine.THREADED` hoặc kỹ thuật gửi đồng thời tương tự để tự động hoá race condition attack với số lượng request lớn.

## 3. Các dạng race condition thường gặp

| Loại | Mô tả |
|---|---|
| **Limit overrun** | Gửi đồng thời nhiều request cùng thực hiện 1 hành động có giới hạn (redeem coupon, rút tiền, đổi điểm thưởng) — do server check-rồi-update không atomic, nhiều request cùng "đọc" thấy điều kiện còn hợp lệ trước khi bất kỳ request nào kịp ghi nhận đã dùng → vượt giới hạn dự kiến |
| **Bypass rate limiting** | Tương tự Authentication topic — gửi nhiều lần thử password đồng thời để bộ đếm rate-limit (vốn cũng có race condition riêng) không kịp tăng đúng trước khi toàn bộ request đã được xử lý |
| **Multi-endpoint race condition** | Race condition xảy ra giữa 2 **endpoint khác nhau** cùng thao tác 1 tài nguyên (ví dụ endpoint A kiểm tra số dư, endpoint B trừ tiền) — gửi đồng thời request tới cả 2 endpoint để lợi dụng khoảng hở giữa chúng |
| **Single-endpoint race condition với session/state phức tạp** | Session hoặc trạng thái nhiều bước (ví dụ luồng đổi email cần xác thực OTP) bị race giữa bước cấp OTP và bước xác minh, cho phép bypass logic tưởng như tuần tự chặt chẽ |
| **Time-sensitive attack (partial construction)** | Lợi dụng khoảng thời gian rất ngắn khi 1 tài nguyên đang được **khởi tạo dần** (ví dụ session mới tạo nhưng chưa gán đầy đủ quyền) để can thiệp đúng lúc "nửa vời" đó |

## 4. Cách phòng chống

- Dùng **transaction có khoá (lock)** ở tầng database cho mọi thao tác đọc-rồi-ghi trên tài nguyên có giới hạn (`SELECT ... FOR UPDATE`, hoặc optimistic locking với version field).
- Thiết kế logic nghiệp vụ theo hướng atomic (kiểm tra và cập nhật trong cùng 1 câu lệnh DB duy nhất, ví dụ `UPDATE wallet SET balance = balance - amount WHERE balance >= amount`).
- Với luồng nhiều bước/nhiều endpoint, đảm bảo trạng thái được đồng bộ hoá đúng cách (không tin tưởng thứ tự request client gửi lên).
- Rate-limit và cơ chế khoá tài khoản cũng cần atomic tương tự.

## 5. Lab step-by-step

### Lab 1: Limit overrun race conditions
**Mục tiêu:** Áp dụng mã giảm giá "chỉ dùng 1 lần" nhiều lần bằng cách gửi đồng thời.
- **B1:** Thêm sản phẩm vào giỏ, áp mã giảm giá hợp lệ 1 lần bình thường để xác nhận cơ chế hoạt động.
- **B2:** Dùng Burp Repeater, tạo tab group gồm **nhiều bản sao** (10-20) của cùng request `POST /cart/coupon` áp mã giảm giá.
- **B3:** Chọn **"Send group in parallel"** (single-packet attack) để gửi toàn bộ đồng thời.
- **B4:** Do server không khoá đúng cách, nhiều request cùng "thấy" mã chưa được dùng → mã bị áp nhiều lần → tổng tiền giảm xuống 0 hoặc âm.
- **B5:** Checkout với giá đã giảm sâu bất thường → lab solved.

### Lab 2: Bypassing rate limits via race conditions
**Mục tiêu:** Brute-force password mà không bị khoá do rate-limit có race condition riêng.
- **B1:** Xác nhận: login sai quá N lần liên tiếp (tuần tự) → bị khoá tài khoản.
- **B2:** Dùng Burp Repeater group, gửi **đồng thời** (parallel) nhiều request login với các password khác nhau (bao gồm cả password đúng nằm trong danh sách) — vượt quá giới hạn N request nhưng gửi cùng lúc.
- **B3:** Vì bộ đếm rate-limit cũng có race condition (đọc số lần thử hiện tại trước khi bất kỳ request nào kịp tăng số đếm) → toàn bộ request được xử lý trước khi tài khoản bị khoá.
- **B4:** 1 trong các request (password đúng) trả về đăng nhập thành công → lab solved.

### Lab 3: Multi-endpoint race conditions
**Mục tiêu:** Mua hàng vượt số dư bằng cách khai thác race condition giữa 2 endpoint khác nhau (giỏ hàng & áp gift card).
- **B1:** Xác định 2 endpoint riêng biệt cùng ảnh hưởng tới số dư: ví dụ `POST /cart/checkout` (trừ tiền) và `POST /cart/coupon` (áp voucher/gift card một lần).
- **B2:** Tạo group gồm nhiều request xen kẽ giữa 2 endpoint này, gửi đồng thời qua "Send group in parallel".
- **B3:** Do race condition giữa việc kiểm tra số dư ở endpoint A và cập nhật trạng thái đã dùng voucher ở endpoint B, có thể checkout thành công nhiều lần với cùng 1 voucher hoặc vượt số dư cho phép.
- **B4:** Xác nhận mua được sản phẩm đắt tiền vượt số dư thực tế → lab solved.

### Lab 4: Single-endpoint race conditions
**Mục tiêu:** Race condition ngay trong 1 endpoint duy nhất xử lý đổi email — gửi đồng thời nhiều địa chỉ email khác nhau để bypass giới hạn domain.
- **B1:** Xác định endpoint đổi email `POST /my-account/change-email` chỉ cho phép domain nội bộ (`@ginandjuice.shop`) là hợp lệ, và có 2 bước: đổi email tạm + xác nhận qua click link email.
- **B2:** Gửi đồng thời (parallel) 2 request đổi email khác nhau tới cùng endpoint trong 1 group: 1 request đổi sang email hợp lệ nội bộ, 1 request đổi sang email attacker kiểm soát — do race condition trong việc kiểm tra & lưu email tạm, request thứ 2 "ghi đè" lên trạng thái mà request thứ 1 đã qua được validate.
- **B3:** Click link xác nhận gửi tới email hợp lệ ban đầu nhưng thực tế đã áp dụng cho địa chỉ email attacker do bị ghi đè → chiếm được quyền xác nhận với domain nội bộ dù dùng email ngoài.
- **B4:** Xác nhận email attacker được liên kết với tài khoản có quyền domain nội bộ → lab solved.

### Lab 5: Exploiting time-sensitive vulnerabilities
**Mục tiêu:** Lợi dụng khoảng thời gian rất ngắn giữa lúc password reset token được tạo và lúc nó được gán đầy đủ thuộc tính bảo mật.
- **B1:** Kích hoạt luồng "Forgot password", dùng Burp Repeater gửi đồng thời (parallel) nhiều request **request token mới** liên tiếp trong khoảng thời gian cực ngắn.
- **B2:** Do race condition, có thể có 2 token hợp lệ tồn tại song song trong khoảng thời gian ngắn (thay vì token cũ bị vô hiệu hoá ngay lập tức khi token mới được tạo) — hoặc token được tạo ra với entropy/thời hạn kiểm tra chưa đầy đủ ở giai đoạn "nửa vời" này.
- **B3:** Khai thác đúng thời điểm đó để dùng lại token tưởng đã hết hạn, hoặc dò được token dễ đoán hơn trong khoảnh khắc bị race → đặt lại password tài khoản mục tiêu.
- **B4:** Đăng nhập bằng password mới đặt → lab solved.

### Lab 6: Partial construction race conditions
**Mục tiêu:** Lợi dụng khoảnh khắc 1 object/session đang được "khởi tạo dần" (chưa đầy đủ) trên server để can thiệp.
- **B1:** Xác định luồng tạo session/tài khoản gồm nhiều bước ghi dữ liệu tuần tự phía server (ví dụ: tạo bản ghi user cơ bản trước, sau đó mới gán quyền/role trong 1 câu lệnh riêng).
- **B2:** Gửi đồng thời request kích hoạt luồng tạo tài khoản đó VÀ 1 request khác cố tình thao tác lên chính tài khoản đang trong quá trình khởi tạo (ví dụ request "get profile" hoặc "assign role") ngay trong khoảng thời gian cực ngắn giữa 2 bước ghi dữ liệu.
- **B3:** Do dữ liệu đang ở trạng thái "nửa vời" (đã tồn tại record nhưng thiếu field quan trọng, ví dụ role mặc định `NULL` thay vì giá trị an toàn) → request thứ 2 có thể set giá trị role tuỳ ý cho chính tài khoản này trước khi bước khởi tạo bình thường kịp hoàn tất và ghi đè.
- **B4:** Xác nhận tài khoản mới tạo có quyền cao hơn dự kiến (ví dụ admin) → lab solved.

## Tổng kết
- Luôn dùng "Send group in parallel" (single-packet attack) trong Burp Repeater thay vì gửi tuần tự khi test race condition — độ trễ dù nhỏ cũng đủ làm mất race window.
- Race condition không chỉ giới hạn ở 1 endpoint — luôn cân nhắc khả năng race condition xảy ra **giữa nhiều endpoint khác nhau** hoặc trong **các bước của 1 luồng nhiều giai đoạn**.
- Đây là lớp lỗ hổng đòi hỏi hiểu rõ luồng nghiệp vụ backend — kết hợp tốt với kiến thức Business Logic Vulnerabilities.

