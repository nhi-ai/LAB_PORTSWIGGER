# INFORMATION DISCLOSURE — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Information disclosure là gì?](#1-information-disclosure-là-gì)
2. [Nguồn phổ biến gây lộ thông tin](#2-nguồn-phổ-biến-gây-lộ-thông-tin)
3. [Cách phát hiện](#3-cách-phát-hiện)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Danh sách lab liên quan](#5-danh-sách-lab-liên-quan)

## 1. Information disclosure là gì?

Xảy ra khi ứng dụng vô tình tiết lộ thông tin nhạy cảm mà lẽ ra người dùng không nên/không cần thấy — có thể là dữ liệu về hạ tầng kỹ thuật (framework, version, đường dẫn nội bộ), dữ liệu người dùng khác, hoặc mã nguồn/logic nội bộ. Bản thân information disclosure có thể không nguy hiểm trực tiếp, nhưng thường là **bước đệm** giúp phát hiện & khai thác các lỗ hổng khác.

## 2. Nguồn phổ biến gây lộ thông tin

| Nguồn | Ví dụ |
|---|---|
| Thông báo lỗi chi tiết (stack trace, debug mode) | Lộ đường dẫn file server, version framework, câu truy vấn SQL |
| File/endpoint không nên public | `/robots.txt`, `.git/`, `backup.zip`, `.env`, comment trong HTML/JS |
| API trả về nhiều field hơn UI hiển thị | Response JSON chứa field ẩn (password hash, internal ID) dù UI chỉ hiện 1 phần |
| HTTP header/response tiết lộ version | `Server: Apache/2.4.29`, `X-Powered-By: PHP/7.2.1` |
| Comment trong source code | `<!-- TODO: remove debug endpoint /api/internal/debug -->` |
| Version control history bị lộ | Thư mục `.git` accessible → dump toàn bộ lịch sử commit, có thể chứa secret cũ |
| Phân biệt hành vi giữa response hợp lệ/không hợp lệ | Dùng để enumerate ID, username (tương tự username enumeration) |

## 3. Cách phát hiện

- Rà soát response cho các endpoint lỗi (`/nonexistent`, tham số sai kiểu dữ liệu) xem có bật debug/stack trace không.
- Duyệt các đường dẫn "chuẩn" hay bị bỏ quên: `/robots.txt`, `/sitemap.xml`, `/.git/`, `/.env`, `/backup`, `/admin/`, `/phpinfo.php`.
- So sánh response API với những gì UI thực sự hiển thị — luôn xem toàn bộ JSON thô, không chỉ phần render ra UI.
- Xem source HTML/JS tìm comment, biến debug, đường dẫn ẩn.
- Dùng Burp Engagement tools → "Find comments" / "Analyze target" để tự động quét các dấu hiệu information disclosure.

## 4. Cách phòng chống

- Tắt debug mode & custom error page chi tiết trên môi trường production; hiển thị thông báo lỗi chung chung cho người dùng, log chi tiết chỉ ở phía server.
- Không để version control (`.git`), file cấu hình (`.env`), backup nằm trong webroot.
- API chỉ trả về đúng field cần thiết (dùng DTO/serializer rõ ràng), không trả nguyên object DB.
- Loại bỏ/minify comment nhạy cảm trước khi deploy code lên production.
- Định kỳ rà soát response header, loại bỏ header tiết lộ version phần mềm không cần thiết.

## 5. Danh sách lab liên quan

1. Information disclosure in error messages
2. Information disclosure on debug page
3. Source code disclosure via backup files
4. Authentication bypass via information disclosure
5. Information disclosure in version control history

