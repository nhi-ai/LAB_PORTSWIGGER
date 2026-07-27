# WEB CACHE POISONING — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Web cache poisoning là gì?](#1-web-cache-poisoning-là-gì)
2. [Cache key & unkeyed input](#2-cache-key--unkeyed-input)
3. [Quy trình khai thác chung](#3-quy-trình-khai-thác-chung)
4. [Các nguồn unkeyed input thường gặp](#4-các-nguồn-unkeyed-input-thường-gặp)
5. [Kỹ thuật nâng cao](#5-kỹ-thuật-nâng-cao)
6. [Cách phòng chống](#6-cách-phòng-chống)
7. [Danh sách lab liên quan](#7-danh-sách-lab-liên-quan)

## 1. Web cache poisoning là gì?

Xảy ra khi attacker khiến cache lưu lại 1 **response độc hại** (do attacker gây ra) dưới 1 cache key mà nhiều user khác cũng sẽ truy cập trúng — khiến toàn bộ user sau đó nhận được response độc hại thay vì response hợp lệ, mà không cần tương tác gì thêm với attacker.

## 2. Cache key & unkeyed input

Cache quyết định 2 request có "giống nhau" hay không dựa trên **cache key** — thường chỉ gồm URL (path) + 1 số header nhất định (KHÔNG phải toàn bộ request). Mọi phần của request **không nằm trong cache key** gọi là **unkeyed input** — dù giá trị của nó ảnh hưởng tới nội dung response, cache vẫn coi các request khác nhau ở phần unkeyed input là "giống hệt nhau" và phục vụ chung 1 response đã lưu.

**Vấn đề:** nếu unkeyed input ảnh hưởng tới nội dung response (ví dụ header `X-Forwarded-Host` được dùng để build URL trong trang) → attacker chỉ cần gửi 1 request với unkeyed input độc hại, cache lưu lại response đó, và **mọi user khác** (dù gửi request khác hẳn ở phần unkeyed input) đều nhận response độc hại đã lưu.

## 3. Quy trình khai thác chung

1. **Xác định unkeyed input:** thử thêm/sửa từng header nghi ngờ (`X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Original-URL`, `User-Agent`...), quan sát response có thay đổi theo giá trị header đó không (mà giá trị này không ảnh hưởng tới cache key — kiểm tra bằng cách gửi lại request y hệt trừ header đó, xem cache có trả về response cũ hay tính lại).
2. **Xây payload độc hại** dựa trên unkeyed input tìm được (thường là XSS qua URL bị phản chiếu vào response).
3. **Đầu độc cache:** gửi request chứa payload, xác nhận response độc hại đã được cache (dựa vào header `X-Cache: hit`, hoặc gửi lại request sạch và thấy vẫn nhận response độc hại).
4. **Xác nhận ảnh hưởng tới user khác:** gửi request hoàn toàn "sạch" (không có bất kỳ input độc hại nào) và kiểm tra vẫn nhận được response độc hại từ cache.

## 4. Các nguồn unkeyed input thường gặp

| Nguồn | Ví dụ khai thác |
|---|---|
| `X-Forwarded-Host` | Server dùng header này build URL tuyệt đối (link import script/canonical) → chèn domain attacker |
| Cookie không nằm trong cache key | Nếu cookie ảnh hưởng nội dung (ví dụ theo dõi A/B testing) nhưng cache không tính cookie vào key → nội dung cá nhân hoá theo cookie bị cache chung cho mọi người |
| Tham số query string "thừa" | Cache chỉ tính 1 số tham số cố định vào key, tham số khác dù ảnh hưởng response vẫn bị bỏ qua khi tính key |
| `User-Agent` | 1 số cache/CDN loại trừ `User-Agent` khỏi key nhưng response vẫn có thể phản ánh giá trị này (hiếm gặp hơn) |
| Header tùy chỉnh nội bộ (`X-Original-URL`, `X-Rewrite-URL`) | Dùng để override routing nội bộ — nếu unkeyed, có thể đầu độc cache cho 1 path hoàn toàn khác path thật của request |

## 5. Kỹ thuật nâng cao

- **Cache probing để xác định chính xác cache key:** gửi 2 request giống hệt nhau ngoại trừ 1 phần cụ thể (ví dụ query string thừa `?cachebuster=random`), quan sát `X-Cache` để xác định phần nào được tính vào cache key.
- **Fat GET request:** gửi GET request kèm body (không chuẩn nhưng nhiều server vẫn xử lý) để chèn tham số ảnh hưởng response mà không đổi cache key (vốn chỉ dựa vào path/query).
- **Parameter cloaking:** dùng ký tự phân cách tham số không chuẩn (`;` thay vì `&`) để "giấu" 1 tham số khỏi bộ phân tích cache key nhưng vẫn được backend xử lý, ví dụ `/?utm_content=123;discount=90`.
- **Cache poisoning qua fragment/internal header discrepancy:** kết hợp với HTTP Request Smuggling (xem topic tương ứng) để đầu độc cache mà không cần lỗ hổng unkeyed input truyền thống.
- **Nhắm nhiều cache layer:** hệ thống có thể có nhiều tầng cache (CDN + reverse proxy nội bộ), cần xác định chính xác tầng nào đang lưu response bị đầu độc để verify đúng cách.

## 6. Cách phòng chống

- Đảm bảo mọi input ảnh hưởng tới nội dung response đều được đưa vào cache key (hoặc loại bỏ hoàn toàn ảnh hưởng của input đó nếu không cần thiết).
- Không dùng header client-controlled (`X-Forwarded-Host`, `X-Forwarded-Scheme`...) để build nội dung response mà không validate whitelist.
- Với response cá nhân hoá (theo cookie/session), đặt `Cache-Control: private, no-store` rõ ràng.
- Định kỳ audit cấu hình cache (CDN, reverse proxy) để xác nhận cache key bao phủ đúng, đủ các yếu tố ảnh hưởng response.
- Chuẩn hoá cách xử lý tham số trùng lặp/ký tự phân cách giữa cache và backend để tránh parameter cloaking.

## 7. Danh sách lab liên quan

1. Web cache poisoning with an unkeyed header
2. Web cache poisoning with an unkeyed cookie
3. Web cache poisoning with multiple headers
4. Targeted web cache poisoning using an unknown header
5. Web cache poisoning via an unkeyed query string
6. Web cache poisoning via an unkeyed query parameter
7. Parameter cloaking
8. Web cache poisoning via a fat GET request
9. URL normalization
10. Web cache poisoning to exploit a DOM vulnerability via a cache with strict cacheability criteria
11. Combining web cache poisoning vulnerabilities
12. Cache key injection
13. Internal cache poisoning

