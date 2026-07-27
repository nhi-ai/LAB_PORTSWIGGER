# ESSENTIAL SKILLS — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## 1. Giới thiệu

Chủ đề này tập trung vào **kỹ năng dùng công cụ** (chủ yếu Burp Suite) để phát hiện lỗ hổng nhanh và hiệu quả hơn, thay vì 1 loại lỗ hổng cụ thể — nền tảng cho việc kiểm thử bất kỳ ứng dụng thực tế nào.

## 2. Targeted scanning với Burp Scanner

Thay vì scan toàn bộ ứng dụng (tốn thời gian, nhiều false positive), nên:
- Xác định trước các điểm nghi ngờ qua thao tác thủ công (crawl, quan sát chức năng).
- Chọn đúng request/vùng cần scan, dùng **"Scan selected insertion points"** trong Burp Scanner để tập trung tài nguyên vào đúng tham số nghi ngờ thay vì scan mù toàn bộ.
- Kết hợp Burp Proxy history + "Send to Scanner" cho từng request cụ thể sau khi đã khoanh vùng chức năng nhạy cảm.

## 3. Scanning non-standard data structures

Nhiều ứng dụng hiện đại gửi dữ liệu ở định dạng không chuẩn REST/form thông thường (JSON lồng nhau, dữ liệu được base64-encode trong 1 field, hoặc custom serialization). Burp Scanner mặc định có thể **không tự nhận diện được** các insertion point bên trong các cấu trúc này.

**Kỹ thuật:**
- Dùng tính năng "Insertion point" thủ công trong Burp (chọn đúng vị trí trong body/JSON để đánh dấu làm điểm test).
- Với dữ liệu được encode lồng nhau (ví dụ 1 tham số là chuỗi JSON đã base64), có thể cần **decode thủ công**, chèn payload, rồi **encode lại** đúng định dạng trước khi gửi test.
- Cấu hình Burp Scanner để tự động decode/encode 1 layer cụ thể trước khi áp payload (qua tính năng "Custom insertion point" hoặc extension hỗ trợ).

## 4. Danh sách lab liên quan (thực hành trực tiếp trên Burp, không có payload cố định)

1. **Discovering vulnerabilities quickly with targeted scanning** — Dùng Burp Proxy để duyệt qua toàn bộ chức năng ứng dụng, sau đó chọn đúng các request liên quan tới chức năng nhập liệu (search, filter, comment...), click phải → "Scan selected items" → theo dõi tab **Dashboard → Issues** để tìm lỗ hổng được Burp tự động phát hiện (thường là SQLi/XSS phản chiếu) → xác nhận & khai thác theo hướng dẫn tương ứng ở các topic khác.
2. **Scanning non-standard data structures** — Xác định 1 tham số dữ liệu dạng lồng nhau/encode đặc biệt (ví dụ field JSON được base64 trong body), dùng Burp Repeater decode thủ công, chèn payload test (`'`, `<script>`...) vào đúng vị trí bên trong, encode lại, gửi lên → nếu cần scan tự động, dùng "Insertion point" thủ công đánh dấu đúng vị trí đã decode trước khi chạy Scanner → xác nhận lỗ hổng cụ thể (thường là SQLi hoặc XSS ẩn trong cấu trúc dữ liệu phức tạp).

## 5. Kỹ năng rút ra

- Không scan mù toàn bộ ứng dụng — luôn khoanh vùng chức năng nghi ngờ trước.
- Với dữ liệu không chuẩn (nested JSON, base64, custom serialization), phải decode thủ công để tìm đúng insertion point trước khi test hoặc scan.
- Thành thạo Burp Proxy, Repeater, Scanner, và tính năng Insertion Point là nền tảng bắt buộc cho mọi topic pentest khác trong bộ lab này.

