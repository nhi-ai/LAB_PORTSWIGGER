# WEB CACHE DECEPTION — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [Web cache deception là gì?](#1-web-cache-deception-là-gì)
2. [Cơ chế gây nhầm lẫn cache](#2-cơ-chế-gây-nhầm-lẫn-cache)
3. [Phân biệt với Web Cache Poisoning](#3-phân-biệt-với-web-cache-poisoning)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Lab step-by-step](#5-lab-step-by-step)

## 1. Web cache deception là gì?

Là kỹ thuật lừa cache lưu lại **response chứa dữ liệu cá nhân/nhạy cảm của 1 user** dưới 1 URL mà cache tưởng là tài nguyên tĩnh dùng chung (do đó lưu công khai) — khiến dữ liệu riêng tư đó bị lộ cho bất kỳ ai truy cập cùng URL đó sau này.

## 2. Cơ chế gây nhầm lẫn cache

Cache thường quyết định có lưu response hay không dựa vào **phần đuôi/path** của URL (ví dụ mọi request kết thúc bằng `.js`, `.css`, `.jpg` được coi là static, mặc định cache), mà không xem xét kỹ **path đầy đủ hoặc logic xử lý thật sự phía server**.

**Path mapping discrepancy:**
```
GET /my-account/random-nonexistent-name.js
```
Nếu server (do cấu hình routing) **vẫn trả về nội dung trang `/my-account`** (bỏ qua phần cuối `.js` không tồn tại, coi như route catch-all), nhưng cache chỉ nhìn thấy đuôi `.js` và quyết định **lưu response này công khai** → dữ liệu cá nhân trong trang `/my-account` (API key, token) bị cache lộ ra cho mọi người dùng URL trên.

**Path/query string confusion:**
```
GET /my-account?x=y.js
```
hoặc
```
GET /my-account%3Fx=y.js
```
Một số cấu hình cache/server khác nhau trong cách hiểu query string/path — có thể dẫn tới cùng loại nhầm lẫn.

## 3. Phân biệt với Web Cache Poisoning

| | Web Cache Deception | Web Cache Poisoning |
|---|---|---|
| Mục tiêu | Lừa cache lưu **dữ liệu riêng tư của chính nạn nhân** dưới URL công khai | Đầu độc cache bằng **nội dung độc hại của attacker** để phục vụ cho mọi user khác |
| Ai bị hại trực tiếp | Chính nạn nhân bị lộ dữ liệu | Toàn bộ user truy cập URL bị đầu độc |
| Cách attacker khai thác | Tự dụ nạn nhân truy cập URL đặc biệt, sau đó tự mình truy cập lại URL đó để đọc cache | Tự gửi request đầu độc, sau đó bất kỳ ai truy cập URL đó đều nhận nội dung độc hại |

## 4. Cách phòng chống

- Cấu hình cache chỉ dựa vào path/content-type **thực sự chính xác** khớp với tài nguyên tĩnh có thật, không suy đoán theo đuôi file trong URL.
- Server backend nên trả **404 rõ ràng** cho path không tồn tại thay vì catch-all route render nhầm nội dung trang khác.
- Không cache bất kỳ response nào có dữ liệu cá nhân hoá (dựa vào cookie/session) — set header `Cache-Control: private, no-store` cho các trang này.
- Kiểm tra kỹ cấu hình cache key — nên bao gồm đầy đủ path + query string chính xác, tránh chuẩn hoá quá mức khiến nhiều URL khác nhau trỏ chung 1 cache entry.

## 5. Lab step-by-step

### Lab 1: Exploiting path mapping for web cache deception
**Mục tiêu:** Đánh cắp API key của nạn nhân qua path mapping discrepancy.
- **B1:** Xác nhận trang `/my-account` hiển thị API key cá nhân của user đăng nhập.
- **B2:** Thử truy cập `/my-account/nonexistent.js` bằng chính tài khoản mình — nếu vẫn trả về đúng nội dung trang tài khoản (do server catch-all route) và response có header cho thấy đã bị cache (`X-Cache: miss` lần đầu, `hit` lần sau) → xác nhận lỗ hổng.
- **B3:** Chuẩn bị link `https://LAB-ID.web-security-academy.net/my-account/tracking.js`, gửi cho nạn nhân (qua Exploit Server hoặc submit trực tiếp nếu lab yêu cầu).
- **B4:** Sau khi nạn nhân (bot) truy cập link (khiến response cá nhân của họ bị cache dưới URL này), tự mình truy cập lại đúng URL đó (không cần đăng nhập) → nhận về API key của nạn nhân từ cache → lab solved.

### Lab 2: Exploiting path delimiter for web cache deception
**Mục tiêu:** Tương tự Lab 1 nhưng dùng ký tự phân cách path khác (ví dụ `;`, `%2f`) để gây nhầm lẫn giữa cache và server thay vì đuôi `.js` thông thường.
- **B1:** Thử các biến thể URL khác nhau để tìm delimiter khiến cache và server hiểu path khác nhau, ví dụ:
  ```
  /my-account;foo.css
  /my-account%2Ftest.css
  ```
- **B2:** Xác định biến thể mà server vẫn trả về đúng nội dung trang cá nhân (bỏ qua phần dư sau delimiter) trong khi cache coi đây là tài nguyên tĩnh (dựa theo đuôi `.css`) và lưu lại.
- **B3:** Gửi link chứa payload đã xác định cho nạn nhân, đợi họ truy cập.
- **B4:** Tự truy cập lại URL đó → nhận API key/dữ liệu cá nhân của nạn nhân từ cache → lab solved.

## Tổng kết
- Web cache deception là "mặt trái" của web cache poisoning — thay vì đầu độc cache bằng nội dung độc hại, ta lừa cache lưu nhầm dữ liệu riêng tư của nạn nhân.
- Luôn thử thêm các đuôi file tĩnh phổ biến (`.js`, `.css`, `.jpg`) hoặc ký tự phân cách path lạ vào cuối URL của trang chứa dữ liệu cá nhân để kiểm tra lỗ hổng này.
- Header `X-Cache`/`Age` trong response là dấu hiệu quan trọng để xác nhận response có đang được cache hay không.

