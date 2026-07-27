# GRAPHQL API VULNERABILITIES — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [GraphQL là gì?](#1-graphql-là-gì)
2. [Discovery: tìm endpoint & introspection](#2-discovery-tìm-endpoint--introspection)
3. [Lỗ hổng access control trong GraphQL](#3-lỗ-hổng-access-control-trong-graphql)
4. [Bypass rate-limit/brute-force protection qua GraphQL](#4-bypass-rate-limitbrute-force-protection-qua-graphql)
5. [CSRF qua GraphQL](#5-csrf-qua-graphql)
6. [Cách phòng chống](#6-cách-phòng-chống)
7. [Danh sách lab liên quan](#7-danh-sách-lab-liên-quan)

## 1. GraphQL là gì?

GraphQL là ngôn ngữ truy vấn API thay thế REST, cho phép client chỉ định chính xác dữ liệu cần lấy trong 1 request duy nhất tới **1 endpoint chung** (thường `/graphql`). Vì tính linh hoạt cao, GraphQL cũng mang theo các dạng lỗ hổng đặc thù.

## 2. Discovery: tìm endpoint & introspection

- Endpoint thường tại `/graphql`, `/graphql/v1`, `/api/graphql`... — có thể ẩn (không public) nhưng vẫn hoạt động.
- **Introspection query** cho phép client tự khám phá toàn bộ schema (mọi type, field, query, mutation khả dụng) — nếu bật trên production, attacker có thể liệt kê toàn bộ API bề mặt tấn công:
  ```graphql
  {
    __schema {
      types { name fields { name } }
    }
  }
  ```
- Nếu introspection bị tắt, có thể dùng kỹ thuật **field suggestion** (GraphQL trả gợi ý "Did you mean...?" khi gõ sai tên field) để dò từng field một cách thủ công.

## 3. Lỗ hổng access control trong GraphQL

- **Accessing private data:** field/query không public trên UI nhưng vẫn tồn tại trong schema và không có access control riêng — gọi trực tiếp bằng query tuỳ chỉnh để lấy dữ liệu private.
- **Accidental exposure of private fields:** 1 type dữ liệu có field không nên public (ví dụ `isSensitive:true`) nhưng khi query lồng (nested) qua 1 field khác vẫn trả về đầy đủ toàn bộ field của type đó, kể cả field đáng ra bị ẩn ở UI chính.

## 4. Bypass rate-limit/brute-force protection qua GraphQL

GraphQL cho phép gộp **nhiều truy vấn (aliased queries) trong 1 request HTTP duy nhất**:
```graphql
query {
  q1: login(username:"carlos", password:"123456") { success }
  q2: login(username:"carlos", password:"password") { success }
  q3: login(username:"carlos", password:"qwerty") { success }
}
```
Nếu rate-limit chỉ đếm theo số lượng HTTP request (không đếm theo số phép toán/truy vấn con bên trong) → có thể brute-force hàng loạt credential chỉ trong vài request → bypass hoàn toàn rate-limit.

## 5. CSRF qua GraphQL

Nếu endpoint GraphQL chấp nhận request dạng `GET` với query string, hoặc `POST` với `Content-Type: text/plain`/`application/x-www-form-urlencoded` (thay vì bắt buộc `application/json`), request có thể được gửi **cross-site bằng form HTML thông thường** (không cần JS phức tạp để set custom header) → dễ dàng thực hiện tấn công CSRF cổ điển nếu thiếu token bảo vệ.

## 6. Cách phòng chống

- Tắt introspection trên môi trường production.
- Áp dụng access control ở tầng resolver cho từng field/query, không chỉ ẩn trên UI.
- Giới hạn độ phức tạp truy vấn (query cost analysis) và đếm rate-limit theo **số lượng operation con** thực tế, không chỉ theo số request HTTP.
- Chỉ chấp nhận GraphQL request qua `POST` với `Content-Type: application/json`, từ chối `GET`/form-encoded để giảm bề mặt CSRF; kết hợp CSRF token nếu cần hỗ trợ nhiều Content-Type.

## 7. Danh sách lab liên quan

1. Accessing private GraphQL posts
2. Accidental exposure of private GraphQL fields
3. Finding a hidden GraphQL endpoint
4. Bypassing GraphQL brute force protections
5. Performing CSRF exploits over GraphQL

