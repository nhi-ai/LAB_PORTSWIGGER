# CSRF (CROSS-SITE REQUEST FORGERY) — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [CSRF là gì?](#1-csrf-là-gì)
2. [Điều kiện để 1 request dễ bị CSRF](#2-điều-kiện-để-1-request-dễ-bị-csrf)
3. [Cơ chế phòng chống CSRF phổ biến](#3-cơ-chế-phòng-chống-csrf-phổ-biến)
4. [Các lỗi khiến CSRF token bị bypass](#4-các-lỗi-khiến-csrf-token-bị-bypass)
5. [SameSite cookie & cách bypass](#5-samesite-cookie--cách-bypass)
6. [Referer/Origin-based defense & cách bypass](#6-refererorigin-based-defense--cách-bypass)
7. [Cách phòng chống đúng](#7-cách-phòng-chống-đúng)
8. [Danh sách lab liên quan](#8-danh-sách-lab-liên-quan)

## 1. CSRF là gì?

CSRF lừa nạn nhân (đang đăng nhập) gửi 1 request không mong muốn tới ứng dụng, thực hiện hành động thay mặt họ (đổi email, đổi password, chuyển tiền...) mà không hề hay biết. Trình duyệt tự động đính kèm cookie session khi gửi request cross-site, nên request giả mạo vẫn được xác thực hợp lệ.

## 2. Điều kiện để 1 request dễ bị CSRF

- Request thực hiện hành động **có giá trị** (đổi thông tin, không phải chỉ đọc dữ liệu).
- Xác thực hoàn toàn dựa vào cookie (session cookie), không có thêm token/header khó đoán khác.
- Các tham số của request có thể **đoán được hoặc cố định** (attacker biết chính xác cần gửi gì).

## 3. Cơ chế phòng chống CSRF phổ biến

- **CSRF token:** giá trị ngẫu nhiên, gắn với session, gửi kèm mọi request thay đổi trạng thái, server verify token khớp với session trước khi xử lý.
- **SameSite cookie attribute:** hạn chế trình duyệt gửi cookie trong request cross-site.
- **Kiểm tra header `Referer`/`Origin`:** xác minh request đến từ chính domain ứng dụng.

## 4. Các lỗi khiến CSRF token bị bypass

| Lỗi | Mô tả |
|---|---|
| Token validation phụ thuộc HTTP method | Server chỉ check token khi method là `POST`, đổi sang `GET` cùng tham số → bypass |
| Token validation phụ thuộc việc "có token hay không" | Nếu request **không gửi kèm tham số token luôn** (xoá hẳn field) thay vì gửi token sai → 1 số implementation chỉ validate khi token tồn tại, bỏ qua nếu thiếu hẳn |
| Token không gắn với session cụ thể | Token hợp lệ với **bất kỳ session nào**, không riêng theo user → attacker tự lấy 1 token hợp lệ của chính mình rồi dùng cho request giả mạo gửi tới nạn nhân |
| Token gắn với cookie khác không phải session | Nếu giá trị token so khớp với 1 cookie khác (không phải session) mà **attacker tự set được cookie đó** cho nạn nhân (qua lỗ hổng khác hoặc cookie không có `HttpOnly`/domain scope hẹp) → attacker tự cấp cặp token+cookie khớp nhau |
| Token bị duplicate vào cookie | Nếu server so sánh token trong request body với token trong 1 cookie riêng (không phải session cookie), và attacker có thể **tự set cookie đó cho nạn nhân** (ví dụ qua subdomain khác không có isolation) → attacker set cả 2 giá trị khớp nhau theo ý mình |

## 5. SameSite cookie & cách bypass

`SameSite` giới hạn khi nào cookie được gửi kèm request cross-site:

| Giá trị | Hành vi |
|---|---|
| `Strict` | Không gửi cookie trong bất kỳ request cross-site nào, kể cả điều hướng top-level (click link) |
| `Lax` (mặc định hiện nay trên nhiều trình duyệt) | Gửi cookie với **GET request điều hướng top-level** (click link) nhưng không gửi với request nền (`POST`/`fetch`/`iframe`/`image`...) |
| `None` | Gửi cookie trong mọi trường hợp cross-site (cần kèm `Secure`) |

**Cách bypass:**
- **Lax bypass via method override:** nếu server hỗ trợ override method qua tham số (`_method=POST` gửi kèm 1 `GET` request), attacker chỉ cần dụ nạn nhân click 1 **link GET thông thường** (được SameSite=Lax cho phép gửi cookie) nhưng server xử lý nó như `POST`.
- **Lax bypass via cookie refresh:** lợi dụng cơ chế trình duyệt coi cookie "mới được set trong vài giây gần đây" là ngoại lệ được gửi cả với request non-top-level (2 phút đầu theo 1 số cài đặt trình duyệt cũ) — dụ nạn nhân qua 1 bước trung gian khiến cookie được set lại ngay trước khi request CSRF thực sự được gửi.
- **Strict bypass via client-side redirect:** nếu ứng dụng victim có 1 endpoint redirect mở (open redirect) hoặc DOM-based redirect, attacker điều hướng nạn nhân **trước tiên đến chính domain victim** (qua link do JS trung gian điều hướng) rồi redirect tiếp tới request CSRF — vì bước điều hướng cuối cùng được trình duyệt coi là "same-site" (do đã ở đúng domain khi redirect xảy ra) nên cookie vẫn được gửi.
- **Strict bypass via sibling domain:** nếu victim có 1 subdomain khác (`sibling domain`) chứa lỗ hổng (ví dụ XSS) và cookie session được set dùng chung cho toàn bộ domain cha (không giới hạn subdomain) → request CSRF xuất phát từ chính sibling domain đó được tính là "same-site" hợp lệ.

## 6. Referer/Origin-based defense & cách bypass

- **Referer validation phụ thuộc header có tồn tại:** nếu server chỉ kiểm tra Referer **khi header này được gửi**, attacker có thể khiến trình duyệt nạn nhân **không gửi Referer** (ví dụ dùng thẻ `<meta name="referrer" content="never">` hoặc policy tương tự trong trang tấn công) → bypass hoàn toàn kiểm tra.
- **Broken Referer validation:** nếu server chỉ kiểm tra Referer có **chứa** domain hợp lệ ở đâu đó (không kiểm tra đúng vị trí/định dạng URL) → attacker đặt domain hợp lệ dưới dạng query string hoặc path trên chính domain của mình để "giả mạo" Referer hợp lệ, ví dụ: `https://attacker.com/csrf-attack?victim-website.com`.

## 7. Cách phòng chống đúng

- Dùng CSRF token ngẫu nhiên, gắn chặt với session hiện tại, validate ở mọi request thay đổi trạng thái, validate token bắt buộc phải tồn tại (không có ngoại lệ khi thiếu).
- Set `SameSite=Strict` (hoặc `Lax` kèm kiểm tra bổ sung) cho session cookie khi có thể.
- Kiểm tra `Origin` header (đáng tin cậy hơn `Referer` vì khó bị disable/giả mạo) làm lớp phòng thủ bổ sung, không thay thế token.
- Áp dụng đồng nhất cho mọi HTTP method trên cùng route.

## 8. Danh sách lab liên quan

1. CSRF vulnerability with no defenses
2. CSRF where token validation depends on request method
3. CSRF where token validation depends on token being present
4. CSRF where token is not tied to user session
5. CSRF where token is tied to non-session cookie
6. CSRF where token is duplicated in cookie
7. SameSite Lax bypass via method override
8. SameSite Strict bypass via client-side redirect
9. SameSite Strict bypass via sibling domain
10. SameSite Lax bypass via cookie refresh
11. CSRF where Referer validation depends on header being present
12. CSRF with broken Referer validation

