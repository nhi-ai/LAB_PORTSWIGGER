# HTTP REQUEST SMUGGLING — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [HTTP Request Smuggling là gì?](#1-http-request-smuggling-là-gì)
2. [Nguyên nhân gốc rễ: CL vs TE](#2-nguyên-nhân-gốc-rễ-cl-vs-te)
3. [Phân loại: CL.TE, TE.CL, TE.TE](#3-phân-loại-clte-tecl-tete)
4. [Cách phát hiện](#4-cách-phát-hiện)
5. [Khai thác thực tế](#5-khai-thác-thực-tế)
6. [Request smuggling trong HTTP/2](#6-request-smuggling-trong-http2)
7. [Các biến thể nâng cao (0.CL, CL.0, desync tinh vi)](#7-các-biến-thể-nâng-cao-0cl-cl0-desync-tinh-vi)
8. [Cách phòng chống](#8-cách-phòng-chống)
9. [Danh sách lab liên quan](#9-danh-sách-lab-liên-quan)

## 1. HTTP Request Smuggling là gì?

Xảy ra khi có **2 tầng server** xử lý request tuần tự (front-end/proxy/load-balancer → back-end) và **2 tầng này hiểu ranh giới giữa các request khác nhau** (bất đồng bộ/desync). Attacker lợi dụng sự khác biệt này để "buôn lậu" (smuggle) 1 phần request vào hàng đợi xử lý của back-end, khiến nó bị ghép nối/ảnh hưởng tới request của **nạn nhân tiếp theo** trên cùng kết nối.

## 2. Nguyên nhân gốc rễ: CL vs TE

Có 2 cách khai báo độ dài body request:
- **`Content-Length` (CL):** khai báo số byte chính xác của body.
- **`Transfer-Encoding: chunked` (TE):** body được chia thành các "chunk", mỗi chunk có kích thước riêng, kết thúc bằng chunk kích thước `0`.

Nếu request gửi **cả 2 header CL và TE cùng lúc**, và front-end / back-end **ưu tiên xử lý header khác nhau** (1 bên đọc theo CL, bên kia đọc theo TE) → 2 tầng sẽ hiểu ranh giới request khác nhau → desync.

## 3. Phân loại: CL.TE, TE.CL, TE.TE

| Loại | Front-end dùng | Back-end dùng | Cách khai thác |
|---|---|---|---|
| **CL.TE** | Content-Length | Transfer-Encoding | Front-end gửi đúng theo CL (coi phần dư là request tiếp theo cùng kết nối), back-end đọc theo TE, dừng sớm hơn ở chunk `0` → phần dư front-end tưởng đã gửi hết bị back-end coi là **phần đầu của request tiếp theo** |
| **TE.CL** | Transfer-Encoding | Content-Length | Ngược lại — back-end đọc đúng CL byte, nhưng front-end xử lý theo chunk, có thể để lại phần dữ liệu chưa gửi ứng với payload gây desync ở phía sau |
| **TE.TE** | Cả 2 đều dùng TE nhưng 1 bên bị "lừa" bỏ qua TE (do obfuscate header) | | Obfuscate header `Transfer-Encoding` (thêm khoảng trắng, tab, viết hoa/thường khác nhau, giá trị không chuẩn) khiến 1 trong 2 tầng không nhận diện được là chunked, quay về xử lý theo CL → biến thành CL.TE hoặc TE.CL |

## 4. Cách phát hiện

- **Differential responses:** gửi 2 request khác nhau tùy giả thuyết (CL.TE hoặc TE.CL), so sánh timeout/response để xác nhận desync xảy ra (không cần ảnh hưởng tới request thật của ai, chỉ tự gây desync với chính request tiếp theo của mình bằng cách gửi 2 request liền nhau trên cùng kết nối và quan sát response thứ 2 có bất thường không).
- **Timing-based:** gửi request với payload cố tình khiến back-end "chờ" thêm dữ liệu (do đếm sai độ dài) → response bị delay bất thường → dấu hiệu desync tồn tại, an toàn để test trên production vì không cần request thứ 2 thật.

**Payload thăm dò CL.TE (timing-based):**
```
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```
Nếu back-end xử lý theo TE (đọc chunk `1` byte `A`, dừng), còn "X" dư ra bị treo lại chờ tiếp → nếu front-end xử lý theo CL (gửi đủ 4 byte) → request kết thúc ở front-end nhưng back-end vẫn đang chờ thêm dữ liệu để hoàn thành request → gây timeout → xác nhận CL.TE.

## 5. Khai thác thực tế

- **Bypass front-end security control:** front-end chặn truy cập `/admin` cho request trực tiếp, nhưng nếu smuggle được request `/admin` ẩn trong phần "dư" của 1 request khác đã qua được front-end (front-end không soi lại phần dư này) → back-end xử lý `/admin` như thể nó hợp lệ.
- **Reveal front-end request rewriting:** front-end thường tự thêm header khi forward request (`X-Forwarded-For`, `X-Forwarded-Proto`...) — smuggle 1 request "rỗng" để buộc front-end append các header đó vào **request tiếp theo** rồi phản chiếu ngược lại nội dung, giúp attacker "nhìn thấy" chính xác front-end đã rewrite gì.
- **Capture other users' requests:** smuggle 1 request tới endpoint có phản hồi (echo) dữ liệu request vào response (ví dụ trang search/comment), khiến **request thật của nạn nhân tiếp theo trên cùng kết nối bị "ghép đuôi"** vào request smuggled — response trả về cho attacker (ở request tiếp theo attacker tự gửi để lấy kết quả) chứa luôn nội dung request của nạn nhân (cookie, header...).
- **Deliver reflected XSS:** smuggle request khiến response cho nạn nhân (request tiếp theo) chứa payload XSS attacker chèn sẵn.
- **Web cache poisoning / deception qua smuggling:** kết hợp desync để đầu độc cache cho toàn bộ user sau đó, hoặc lừa cache lưu nhầm response cá nhân hoá dưới key công khai (xem thêm 2 topic Web Cache).

## 6. Request smuggling trong HTTP/2

HTTP/2 dùng cơ chế đóng khung (framing) nhị phân xác định rõ độ dài — không dùng CL/TE theo cách HTTP/1.1, nhưng khi hệ thống **downgrade HTTP/2 → HTTP/1.1** ở tầng back-end (rất phổ biến), lỗi vẫn phát sinh:

| Loại | Mô tả |
|---|---|
| **H2.TE** | Front-end (HTTP/2) chuyển tiếp header giả `Transfer-Encoding` xuống HTTP/1.1 back-end, gây desync tương tự CL.TE cổ điển |
| **H2.CL** | Tương tự nhưng dựa trên chênh lệch Content-Length khi downgrade |
| **CRLF injection trong pseudo-header HTTP/2** | HTTP/2 không dùng CRLF để phân tách nhưng khi convert sang HTTP/1.1, nếu 1 trường (ví dụ path) cho phép chèn `\r\n`, có thể "tự tạo" ranh giới header/request giả sau khi downgrade → request splitting/smuggling |
| **Response queue poisoning** | Desync khiến response bị lệch thứ tự, response của người này bị trả nhầm cho người khác (do hàng đợi response trên kết nối bị lệch pha so với hàng đợi request) |

## 7. Các biến thể nâng cao (0.CL, CL.0, desync tinh vi)

| Loại | Mô tả |
|---|---|
| **0.CL** | Back-end coi request GET/POST không có `Content-Length` là có body (mặc định 1 số server implicit đọc thêm), trong khi front-end coi là không có body → desync dù không hề khai báo CL/TE mâu thuẫn tường minh |
| **CL.0** | Ngược lại — front-end đọc theo CL đầy đủ, back-end trong 1 số trường hợp đặc biệt (ví dụ do 1 middleware nào đó) bỏ qua Content-Length và coi body rỗng |
| **Client-side desync** | Desync không nằm giữa 2 server, mà giữa **trình duyệt nạn nhân và server** — dùng chính trình duyệt nạn nhân (qua 1 trang tấn công) để tạo request desync nhắm vào chính kết nối của họ, biến CSRF/XSS truyền thống thành tấn công mạnh hơn không cần lỗ hổng desync ở tầng proxy |
| **Server-side pause-based** | Server có hành vi "tạm dừng" khi nhận dữ liệu chưa đầy đủ theo 1 cách khác biệt tinh vi (không dựa vào CL/TE mâu thuẫn thông thường mà dựa vào timing xử lý stream), cho phép desync dù cấu hình front-end/back-end tưởng như nhất quán |

## 8. Cách phòng chống

- Ưu tiên dùng **HTTP/2 end-to-end** giữa mọi tầng (loại bỏ hoàn toàn CL/TE ambiguity của HTTP/1.1).
- Nếu bắt buộc dùng HTTP/1.1: cấu hình front-end và back-end **từ chối** request có cả `Content-Length` và `Transfer-Encoding` cùng lúc.
- Chuẩn hoá (normalize) request tại front-end trước khi forward, đảm bảo back-end nhận request đã được viết lại rõ ràng, không mơ hồ.
- Đóng kết nối sau mỗi request (tắt keep-alive) giữa front-end-back-end nếu có thể, giảm bề mặt tấn công (dù ảnh hưởng hiệu năng).
- Cập nhật thường xuyên phần mềm proxy/server, theo dõi các CVE liên quan đến request smuggling.

## 9. Danh sách lab liên quan

1. HTTP request smuggling, confirming a CL.TE vulnerability via differential responses
2. HTTP request smuggling, confirming a TE.CL vulnerability via differential responses
3. Exploiting HTTP request smuggling to bypass front-end security controls, CL.TE vulnerability
4. Exploiting HTTP request smuggling to bypass front-end security controls, TE.CL vulnerability
5. Exploiting HTTP request smuggling to reveal front-end request rewriting
6. Exploiting HTTP request smuggling to capture other users' requests
7. Exploiting HTTP request smuggling to deliver reflected XSS
8. Response queue poisoning via H2.TE request smuggling
9. H2.CL request smuggling
10. HTTP/2 request smuggling via CRLF injection
11. HTTP/2 request splitting via CRLF injection
12. 0.CL request smuggling
13. CL.0 request smuggling
14. HTTP request smuggling, basic CL.TE vulnerability
15. HTTP request smuggling, basic TE.CL vulnerability
16. HTTP request smuggling, obfuscating the TE header
17. Exploiting HTTP request smuggling to perform web cache poisoning
18. Exploiting HTTP request smuggling to perform web cache deception
19. Bypassing access controls via HTTP/2 request tunnelling
20. Web cache poisoning via HTTP/2 request tunnelling
21. Client-side desync
22. Server-side pause-based request smuggling

