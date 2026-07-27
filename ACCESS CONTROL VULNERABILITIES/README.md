# ACCESS CONTROL VULNERABILITIES — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Access control là gì?](#1-access-control-là-gì)
2. [Phân loại lỗ hổng access control](#2-phân-loại-lỗ-hổng-access-control)
3. [Vertical privilege escalation](#3-vertical-privilege-escalation)
4. [Horizontal privilege escalation & IDOR](#4-horizontal-privilege-escalation--idor)
5. [Access control dựa trên nhiều bước (multi-step)](#5-access-control-dựa-trên-nhiều-bước-multi-step)
6. [Access control dựa trên tham số/HTTP method dễ bị bypass](#6-access-control-dựa-trên-tham-sốhttp-method-dễ-bị-bypass)
7. [Cách phòng chống](#7-cách-phòng-chống)
8. [Danh sách lab liên quan](#8-danh-sách-lab-liên-quan)

## 1. Access control là gì?

Access control áp dụng các chính sách để đảm bảo người dùng không thể hành động vượt quá quyền hạn được cấp. Vi phạm access control là loại lỗ hổng web nghiêm trọng và phổ biến nhất, dẫn tới truy cập trái phép dữ liệu nhạy cảm hoặc thực hiện hành động ngoài phạm vi cho phép.

**3 loại chính:**
- **Vertical access control:** phân quyền theo loại người dùng (user thường vs admin).
- **Horizontal access control:** phân quyền giữa các người dùng cùng cấp (user A không được xem dữ liệu của user B).
- **Context-dependent access control:** giới hạn theo trạng thái/luồng nghiệp vụ (ví dụ không được sửa giỏ hàng sau khi thanh toán).

## 2. Phân loại lỗ hổng access control

| Loại | Ví dụ |
|---|---|
| Thiếu kiểm soát hoàn toàn | trang admin không yêu cầu xác thực gì cả |
| Dựa vào "security by obscurity" | URL admin khó đoán nhưng không có auth check |
| Điều khiển bằng tham số client gửi lên | `roleid=2` trong request có thể chỉnh sửa |
| IDOR (Insecure Direct Object Reference) | `account?id=101` - đổi id xem tài khoản người khác |
| Access control theo URL không nhất quán | check ở trang chính nhưng API backend không check |
| Access control theo HTTP method | chỉ chặn `POST`, không chặn `GET`/`PUT` tới cùng endpoint |
| Access control multi-step | chỉ kiểm tra ở bước 1, bỏ qua ở bước xác nhận cuối |
| Access control dựa trên Referer header | Referer dễ giả mạo, không nên dùng làm cơ sở xác thực |

## 3. Vertical privilege escalation

Xảy ra khi user thường có thể truy cập chức năng chỉ dành cho admin. Nguyên nhân phổ biến:
- Trang admin không kiểm tra quyền, chỉ ẩn link trên UI.
- URL admin "khó đoán" nhưng không hề bảo vệ bằng auth (`/administrator-panel-yb556`).
- Vai trò (role) được xác định qua tham số request (`roleid`, `isAdmin=true` trong cookie/hidden field) mà attacker có thể tự sửa.

## 4. Horizontal privilege escalation & IDOR

Xảy ra khi hệ thống dùng **ID do client cung cấp** để xác định đối tượng dữ liệu, mà không kiểm tra đối tượng đó có thuộc về user hiện tại hay không:

```
GET /myaccount?id=wiener   → đổi thành id=carlos để xem tài khoản người khác
```

Ngay cả khi ID là UUID "không đoán được", nếu ID này bị **lộ ở nơi khác** (đường dẫn ảnh, thông báo lỗi, redirect URL) thì vẫn khai thác được → kết hợp horizontal thành vertical (ví dụ lấy được ID + password của admin qua 1 endpoint hở, rồi login).

## 5. Access control dựa trên nhiều bước (multi-step)

Chức năng nhạy cảm (ví dụ "Grant admin rights") thường có nhiều bước: chọn user → xác nhận → submit. Nếu chỉ bước đầu tiên kiểm tra quyền admin, attacker có thể **gọi thẳng bước cuối cùng** (bỏ qua bước 1) để thực hiện hành động mà không cần quyền admin.

## 6. Access control dựa trên tham số/HTTP method dễ bị bypass

- **URL-based:** Nếu access control implement ở tầng front-end proxy dựa theo path (`/admin/*` bị chặn) nhưng backend có route khác trỏ tới cùng chức năng (`/admin/../admin/`, hoặc thêm `%2e/`, `//admin`), có thể bypass.
- **Method-based:** Nếu chỉ chặn `POST /admin/deleteUser` cho non-admin nhưng không chặn `GET`/`PUT` cùng path (nhiều framework tự động chấp nhận method khác cho cùng route) → đổi method để bypass.
- **Referer-based:** Access control dựa vào header `Referer` để xác nhận request đến từ trang admin — header này hoàn toàn do client tự set, dễ dàng giả mạo.

## 7. Cách phòng chống

- Deny-by-default: mặc định từ chối truy cập, chỉ cấp quyền tường minh khi cần.
- Kiểm tra access control **ở tầng backend cho mọi request**, không dựa vào việc ẩn UI hay URL "khó đoán".
- Không dùng ID do client cung cấp để xác định quyền sở hữu dữ liệu — luôn đối chiếu với user trong session hiện tại phía server.
- Kiểm tra quyền ở **từng bước** của luồng nghiệp vụ nhiều bước, không chỉ bước đầu.
- Không dùng Referer/User-Agent hay bất kỳ header client-controlled nào làm cơ sở xác thực.
- Áp dụng cùng 1 policy kiểm soát truy cập cho mọi HTTP method trên cùng 1 route.

## 8. Danh sách lab liên quan

Chi tiết từng bước tại [`LAB ACCESS CONTROL VULNERABILITIES.md`](./LAB%20ACCESS%20CONTROL%20VULNERABILITIES.md).

1. Unprotected admin functionality
2. Unprotected admin functionality with unpredictable URL
3. User role controlled by request parameter
4. User role can be modified in user profile
5. User ID controlled by request parameter
6. User ID controlled by request parameter, with unpredictable user IDs
7. User ID controlled by request parameter with data leakage in redirect
8. User ID controlled by request parameter with password disclosure
9. Insecure direct object references
10. URL-based access control can be circumvented
11. Method-based access control can be circumvented
12. Multi-step process with no access control on one step
13. Referer-based access control

