
# PROTOTYPE POLLUTION — KIẾN THỨC TỔNG HỢP

## Mục lục
1. [Prototype pollution là gì?](#1-prototype-pollution-là-gì)
2. [Client-side prototype pollution](#2-client-side-prototype-pollution)
3. [Server-side prototype pollution (Node.js)](#3-server-side-prototype-pollution-nodejs)
4. [Quy trình khai thác chung](#4-quy-trình-khai-thác-chung)
5. [Cách phòng chống](#5-cách-phòng-chống)
6. [Danh sách lab liên quan](#6-danh-sách-lab-liên-quan)

## 1. Prototype pollution là gì?

JavaScript dùng cơ chế prototype-based inheritance — mọi object thường kế thừa thuộc tính từ `Object.prototype`. Prototype pollution xảy ra khi attacker có thể **ghi thêm/ghi đè thuộc tính vào chính `Object.prototype`** (thông qua merge/clone object không an toàn với input attacker kiểm soát chứa key `__proto__`), khiến **mọi object trong ứng dụng** (kể cả object tạo sau này) đều "thừa hưởng" thuộc tính độc hại đó.

```js
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
merge({}, JSON.parse('{"__proto__":{"isAdmin":true}}'));
// Sau lệnh này, MỌI object thường (kể cả {} rỗng) đều có obj.isAdmin === true
```

## 2. Client-side prototype pollution

**Bước 1 — Tìm nguồn gây pollution:** thường qua parse query string bằng thư viện không an toàn (jQuery `$.extend` cũ, hoặc code tự viết đệ quy merge object) đọc từ `location.search`:
```
?__proto__[foo]=bar
```
**Bước 2 — Tìm "gadget":** đoạn code sau đó **đọc 1 thuộc tính từ object thường** (không kiểm tra `hasOwnProperty`) và dùng giá trị đó cho 1 **sink nguy hiểm** (`innerHTML`, `eval`...). Vì object thường giờ "thừa hưởng" property đã bị polluted từ `Object.prototype`, gadget vô tình đọc phải giá trị độc hại của attacker.

**Ví dụ gadget dẫn tới XSS:**
```js
// code ứng dụng đọc config.transport_url không kiểm tra hasOwnProperty
var config = {};
document.write('<script src="' + (config.transport_url || '/default.js') + '"></script>');
```
Payload: `?__proto__[transport_url]=data:,alert(1)//` → sau khi pollute, `config.transport_url` thừa hưởng giá trị độc hại dù `config` là object rỗng.

## 3. Server-side prototype pollution (Node.js)

Tương tự client-side nhưng nguồn thường là **body JSON** của request được merge vào object cấu hình nội bộ bằng thư viện không an toàn (`lodash.merge` phiên bản cũ, hoặc code tự viết):
```json
{"__proto__":{"isAdmin":true}}
```
Gadget phía server có thể dẫn tới:
- Bypass access control (nếu code check `if (user.isAdmin)` mà object `user` không có sẵn thuộc tính này).
- **Remote Code Execution:** nếu gadget cuối cùng dùng giá trị polluted để build 1 lệnh `child_process.exec()`, hoặc polluted 1 thuộc tính ảnh hưởng tới cách 1 template engine/module nội bộ hoạt động (ví dụ ghi đè thuộc tính cấu hình `execArgv`/`shell` được dùng bởi 1 thư viện spawn process).

## 4. Quy trình khai thác chung

1. **Xác định pollution source:** endpoint/tham số nào merge object attacker-controlled vào object khác không an toàn (thử `__proto__[canary]=1`, dùng DevTools kiểm tra `Object.prototype.canary` có bị set không).
2. **Tìm gadget:** rà soát code (hoặc dùng công cụ tự động như Burp DOM Invader hỗ trợ phát hiện prototype pollution phía client) để tìm nơi đọc property không kiểm tra `hasOwnProperty`, dẫn tới sink nguy hiểm.
3. **Kết hợp source + gadget:** xây payload polluted đúng property mà gadget sẽ đọc, dẫn tới hành vi mong muốn (XSS, bypass logic, RCE).

## 5. Cách phòng chống

- Dùng `Object.create(null)` hoặc `Map` thay vì object literal thường cho dữ liệu key-value động từ input người dùng (không kế thừa `Object.prototype`).
- Khi merge/clone object, luôn kiểm tra và loại bỏ key `__proto__`, `constructor`, `prototype` trước khi gán.
- Dùng thư viện đã vá lỗi này (`lodash` bản mới, hoặc `Object.freeze(Object.prototype)` để chặn hoàn toàn việc ghi đè).
- Trong code đọc property từ object động, luôn dùng `Object.prototype.hasOwnProperty.call(obj, key)` thay vì `obj[key]`/`key in obj` đơn thuần.

## 6. Danh sách lab liên quan

1. Client-side prototype pollution via browser APIs
2. DOM XSS via client-side prototype pollution
3. Client-side prototype pollution via flawed sanitization
4. Detecting server-side prototype pollution without polluted property reflection
5. Bypassing flawed input filters for server-side prototype pollution
6. Privilege escalation via server-side prototype pollution
7. Remote code execution via server-side prototype pollution
8. Client-side prototype pollution via a JQuery URL parsing gadget
9. Client-side prototype pollution via document.write sink
10. Auditing server-side prototype pollution
