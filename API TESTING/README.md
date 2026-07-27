# API TESTING — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [API là gì & vì sao cần test riêng](#1-api-là-gì--vì-sao-cần-test-riêng)
2. [Xác định các API endpoint](#2-xác-định-các-api-endpoint)
3. [Khai thác lỗ hổng dựa trên tài liệu API (documentation)](#3-khai-thác-lỗ-hổng-dựa-trên-tài-liệu-api-documentation)
4. [Mass assignment](#4-mass-assignment)
5. [Server-side parameter pollution (SSPP)](#5-server-side-parameter-pollution-sspp)
6. [Cách phòng chống](#6-cách-phòng-chống)
7. [Danh sách lab liên quan](#7-danh-sách-lab-liên-quan)

## 1. API là gì & vì sao cần test riêng

API cho phép các thành phần phần mềm giao tiếp với nhau. Do API thường chỉ được dùng bởi phần khác của hệ thống (không phải trình duyệt trực tiếp) nên đôi khi được lập trình viên **giả định là "đáng tin"**, dẫn đến ít kiểm soát bảo mật hơn giao diện web thông thường — đây là nguồn gốc của nhiều lỗ hổng API testing.

## 2. Xác định các API endpoint

- Đọc tài liệu API công khai (`/api/swagger.json`, `/api-docs`, `/openapi.json`...).
- Dò các endpoint **không được liệt kê công khai** nhưng vẫn tồn tại (unused/undocumented endpoint), thường tìm thấy qua:
  - Phân tích file JS phía client (chứa các lời gọi API ẩn).
  - Đoán theo pattern REST chuẩn (nếu có `/api/v1/user`, thử `/api/v1/users`, `/api/v2/user`...).
  - Sử dụng wordlist + fuzzing (Burp Intruder / ffuf).

## 3. Khai thác lỗ hổng dựa trên tài liệu API (documentation)

Tài liệu API (Swagger/OpenAPI) đôi khi liệt kê **endpoint quản trị hoặc chức năng ẩn** mà giao diện web không hiển thị link tới — nhưng backend vẫn expose và không luôn được bảo vệ đúng access control. Đọc kỹ tài liệu để tìm các endpoint như `DELETE /api/user/{id}` không có trên UI.

## 4. Mass assignment

Xảy ra khi framework tự động "bind" toàn bộ field trong request body vào object dữ liệu (model) mà không giới hạn field nào được phép client set. Nếu object có field nhạy cảm (`isAdmin`, `role`, `credit`) mà lẽ ra chỉ server được set, attacker có thể **tự thêm field đó vào JSON request** để ghi đè giá trị, dù UI/tài liệu không hề đề cập tới field này:

```json
POST /api/user/update
{
  "email": "test@test.com",
  "roleid": 2
}
```

Nếu backend không whitelist field hợp lệ mà bind trực tiếp toàn bộ JSON vào model DB → `roleid` bị ghi đè trái phép.

## 5. Server-side parameter pollution (SSPP)

Xảy ra khi ứng dụng frontend/backend nội bộ **nối thêm tham số của attacker** vào 1 request/URL nội bộ khác (ví dụ gọi đến 1 API backend khác) mà không encode đúng cách, cho phép attacker **chèn thêm tham số mới** vào request nội bộ đó.

Ví dụ: ứng dụng nhận `productId` từ user rồi tự dựng URL gọi API nội bộ:
```
GET /api/internal/order?productId=1
```
Nếu user gửi `productId=1%26admin=true` (đã URL-encode `&`), khi backend **decode 2 lần** hoặc nối chuỗi không an toàn, giá trị `&admin=true` trở thành 1 tham số HTTP thật sự mới:
```
GET /api/internal/order?productId=1&admin=true
```
→ Chèn được tham số tuỳ ý vào request nội bộ mà bình thường attacker không kiểm soát được (có thể áp dụng cho query string hoặc REST URL segment).

## 6. Cách phòng chống

- Không auto-bind toàn bộ field JSON vào model DB — dùng DTO/whitelist field rõ ràng.
- Áp dụng access control cho MỌI endpoint, kể cả endpoint "nội bộ"/không công khai trong docs.
- Encode đúng khi nối tham số vào URL nội bộ, tránh double-decode.
- Không công khai tài liệu API chứa endpoint nhạy cảm nếu không bảo vệ đúng access control.

## 7. Danh sách lab liên quan

1. Exploiting an API endpoint using documentation
2. Exploiting server-side parameter pollution in a query string
3. Finding and exploiting an unused API endpoint
4. Exploiting a mass assignment vulnerability
5. Exploiting server-side parameter pollution in a REST URL

