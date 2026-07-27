# JWT (JSON WEB TOKENS) — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [JWT là gì & cấu trúc](#1-jwt-là-gì--cấu-trúc)
2. [Các lỗ hổng JWT thường gặp](#2-các-lỗ-hổng-jwt-thường-gặp)
3. [Kỹ thuật khai thác chi tiết](#3-kỹ-thuật-khai-thác-chi-tiết)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Danh sách lab liên quan](#5-danh-sách-lab-liên-quan)

## 1. JWT là gì & cấu trúc

JWT (JSON Web Token) là chuẩn mã hoá thông tin dạng token, gồm 3 phần ngăn cách bởi dấu `.`, mỗi phần base64url-encode:
```
header.payload.signature
```
- **Header:** chứa thuật toán ký (`alg`) và loại token (`typ`), ví dụ `{"alg":"HS256","typ":"JWT"}`.
- **Payload:** chứa các "claim" (dữ liệu), ví dụ `{"sub":"wiener","admin":false}`.
- **Signature:** chữ ký đảm bảo header+payload không bị chỉnh sửa, tính bằng thuật toán khai báo trong header + 1 secret key (HMAC) hoặc private key (RSA/ECDSA).

## 2. Các lỗ hổng JWT thường gặp

| Lỗ hổng | Mô tả |
|---|---|
| Không verify chữ ký | Server chỉ decode payload mà không verify signature → sửa tuỳ ý payload |
| Chấp nhận `alg: none` | JWT cho phép khai báo không ký (`"alg":"none"`) — nếu server không chặn, gửi token không chữ ký vẫn được chấp nhận |
| Algorithm confusion (RS256 → HS256) | Server hỗ trợ cả RSA và HMAC; nếu code không ràng buộc đúng thuật toán khi verify, attacker đổi `alg` thành `HS256` và **dùng public key RSA (thường công khai) làm secret HMAC** để tự ký token giả |
| Weak/guessable HMAC secret | Secret key HS256 yếu, có thể brute-force offline bằng wordlist |
| JWK header injection (`jwk`) | Header JWT cho phép nhúng public key ngay trong token (`jwk` claim) — nếu server tin tưởng key trong chính token để verify, attacker tự tạo cặp khoá riêng, ký token bằng private key của mình, nhúng public key tương ứng vào header |
| `jku` header injection | Header `jku` trỏ tới URL chứa JWK Set để verify — nếu server fetch URL này mà không whitelist domain, attacker host JWK Set riêng chứa public key của mình |
| `kid` header injection (path traversal / SQLi) | Header `kid` (key ID) dùng để server tra cứu đúng key trong file/DB — nếu giá trị `kid` không được sanitize, có thể dùng path traversal trỏ tới file server biết trước nội dung (ví dụ `/dev/null`) làm key, hoặc SQL injection vào câu truy vấn tra cứu key |

## 3. Kỹ thuật khai thác chi tiết

**Không verify chữ ký:**
```
header.payload.signature
```
→ Sửa payload (ví dụ `"admin":true`), giữ nguyên chữ ký gốc (sai) → nếu server chỉ decode mà không verify → vẫn được chấp nhận.

**`alg: none`:**
```json
{"alg":"none","typ":"JWT"}
```
Bỏ hoàn toàn phần signature (giữ dấu `.` cuối, để trống): `eyJhbGciOiJub25lIn0.eyJhZG1pbiI6dHJ1ZX0.`

**Algorithm confusion RS256→HS256:**
1. Lấy public key RSA của server (thường công khai qua endpoint `/jwks.json` hoặc chứng chỉ TLS).
2. Đổi header `alg` từ `RS256` → `HS256`.
3. Ký lại token bằng HMAC-SHA256, dùng **nguyên văn public key** (bao gồm cả `-----BEGIN PUBLIC KEY-----`) làm secret.
4. Server verify bằng cùng key đó nhưng theo thuật toán HMAC (do code không ràng buộc `alg` cố định) → chấp nhận token giả.

**`jwk` header injection:**
```json
{
  "alg":"RS256",
  "jwk": {
    "kty":"RSA","kid":"attacker-key","use":"sig",
    "n":"...","e":"AQAB"
  }
}
```
Ký token bằng private key tương ứng với `n`/`e` đã nhúng → nếu server verify bằng chính public key trong header thay vì key đã lưu sẵn → token giả hợp lệ.

**`kid` path traversal:**
```json
{"alg":"HS256","kid":"../../../../dev/null"}
```
Nếu server đọc file tại đường dẫn `kid` chỉ định làm secret key, và `/dev/null` đọc ra là chuỗi rỗng → ký token bằng HMAC với secret là chuỗi rỗng (`""`).

## 4. Cách phòng chống

- Luôn verify chữ ký JWT trên **mọi** request, không chỉ decode.
- Cấu hình thư viện JWT chỉ chấp nhận **1 thuật toán cố định** đã biết trước, từ chối token có `alg` khác (đặc biệt `none`).
- Không lấy key để verify từ chính nội dung token (`jwk`, `jku`, `kid` không kiểm soát) — luôn dùng key đã cấu hình cứng phía server hoặc whitelist domain nghiêm ngặt cho `jku`.
- Sanitize/validate chặt giá trị `kid` nếu dùng để tra cứu key (whitelist danh sách kid hợp lệ, không cho phép ký tự path traversal/SQL).
- Dùng secret key HMAC đủ dài, ngẫu nhiên, không đoán được.
- Đặt thời hạn ngắn cho token, có cơ chế thu hồi (revocation) khi cần.

## 5. Danh sách lab liên quan

1. JWT authentication bypass via unverified signature
2. JWT authentication bypass via flawed signature verification
3. JWT authentication bypass via weak signing key
4. JWT authentication bypass via jwk header injection
5. JWT authentication bypass via jku header injection
6. JWT authentication bypass via kid header path traversal
7. JWT authentication bypass via algorithm confusion
8. JWT authentication bypass via algorithm confusion with no exposed key

