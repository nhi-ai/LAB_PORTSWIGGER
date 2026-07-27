# BUSINESS LOGIC VULNERABILITIES — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Business logic vulnerability là gì?](#1-business-logic-vulnerability-là-gì)
2. [Nguyên nhân phổ biến](#2-nguyên-nhân-phổ-biến)
3. [Các dạng lỗi logic thường gặp](#3-các-dạng-lỗi-logic-thường-gặp)
4. [Cách phát hiện](#4-cách-phát-hiện)
5. [Cách phòng chống](#5-cách-phòng-chống)
6. [Danh sách lab liên quan](#6-danh-sách-lab-liên-quan)

## 1. Business logic vulnerability là gì?

Là các lỗ hổng phát sinh từ **thiết kế/luồng nghiệp vụ bị sai sót**, không phải lỗi kỹ thuật kiểu injection. Ứng dụng vẫn hoạt động "đúng" theo logic đã viết, nhưng logic đó bị attacker lợi dụng theo cách nhà phát triển không lường trước — ví dụ mua hàng giá âm, vượt giới hạn số lần thử, bỏ qua 1 bước bắt buộc trong quy trình.

## 2. Nguyên nhân phổ biến

- Tin tưởng quá mức vào kiểm soát phía client (validate chỉ ở JS, không check lại ở server).
- Giả định người dùng luôn tuân theo đúng thứ tự luồng thao tác dự kiến trên UI.
- Áp dụng rule nghiệp vụ không nhất quán giữa các phần khác nhau của hệ thống.
- Không xử lý input "ngoại lệ" (giá trị âm, số 0, ký tự đặc biệt, kiểu dữ liệu khác thường).
- Endpoint dùng chung cho nhiều mục đích (dual-use) không tách biệt rõ ràng quyền truy cập.

## 3. Các dạng lỗi logic thường gặp

| Loại | Ví dụ |
|---|---|
| Excessive trust in client-side controls | Giá sản phẩm gửi kèm trong request (`price=100`) thay vì server tự tra DB → sửa giá trực tiếp |
| High-level logic vulnerability | Áp mã giảm giá nhiều lần, hoặc mua số lượng âm để được hoàn tiền dương |
| Inconsistent security controls | Trang chủ chặn email domain lạ, nhưng API đăng ký khác lại không chặn |
| Flawed enforcement of business rules | Rule "chỉ áp dụng 1 mã giảm giá/đơn" chỉ check ở UI, không check ở server |
| Low-level logic flaw | Lỗi làm tròn số khi tính giá (rounding) bị lợi dụng nhiều lần để lấy lợi nhuận |
| Inconsistent handling of exceptional input | Gửi số âm vào ô "số lượng" khiến tổng tiền âm → được hoàn tiền |
| Weak isolation on dual-use endpoint | Endpoint dùng chung cho "đổi email của mình" và "đổi email user khác" (do admin) không tách quyền |
| Insufficient workflow validation | Bỏ qua bước xác nhận giữa chừng của quy trình nhiều bước |
| Authentication bypass via flawed state machine | Máy trạng thái đăng nhập cho phép nhảy cóc bước xác thực |
| Bypassing access controls using email parsing discrepancies | Các hệ thống khác nhau (mail server, ứng dụng) hiểu địa chỉ email theo cách khác nhau, có thể lợi dụng ký tự đặc biệt (`+`, khoảng trắng ẩn, comment `()`) để "hợp lệ hoá" 1 email thực chất trỏ tới nơi khác |

## 4. Cách phát hiện

- Đọc kỹ mọi bước của 1 luồng nghiệp vụ, đặt câu hỏi: "điều gì xảy ra nếu tôi bỏ qua bước này / lặp lại bước kia / gửi giá trị âm / gửi giá trị vượt giới hạn?"
- Thử can thiệp trực tiếp vào request (giá tiền, số lượng, ID sản phẩm, coupon code) thay vì thao tác qua UI.
- Kiểm tra tính nhất quán giữa các endpoint cùng xử lý 1 loại dữ liệu (ví dụ email) — có thể 1 nơi validate chặt, nơi khác lỏng lẻo.

## 5. Cách phòng chống

- Không bao giờ tin dữ liệu nhạy cảm (giá, số lượng, quyền) gửi từ client — luôn tính toán/tra cứu lại phía server.
- Áp dụng rule nghiệp vụ nhất quán ở **mọi điểm vào** xử lý cùng loại dữ liệu.
- Validate input với giá trị biên (0, âm, rất lớn, kiểu dữ liệu sai) ở mọi bước.
- Kiểm tra trạng thái/quyền sở hữu ở từng bước của quy trình nhiều bước, không giả định thứ tự thao tác của user.
- Chuẩn hoá cách xử lý địa chỉ email giữa các hệ thống liên quan (theo đúng RFC, tránh dựa vào implementation khác nhau).

## 6. Danh sách lab liên quan

1. Excessive trust in client-side controls
2. High-level logic vulnerability
3. Inconsistent security controls
4. Flawed enforcement of business rules
5. Low-level logic flaw
6. Inconsistent handling of exceptional input
7. Weak isolation on dual-use endpoint
8. Insufficient workflow validation
9. Authentication bypass via flawed state machine
10. Bypassing access controls using email address parsing discrepancies

