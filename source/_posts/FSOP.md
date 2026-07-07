---
title: FSOP
tag: ['PWN', 'Dummy']
date: 2026-07-02 22:39:33
cover: /img/post/dummy/fsop/fsop.png
category: ['Writeup']
author:
    - Vuk3r
---
FSOP - File Stream Oriented Programming, đại khái là điều khiển được luồng mà struct của _IO_FILE hoạt động.  Đây là take note của mình tránh trường hợp mình của tương lai đặt các câu hỏi y như lúc mình đang học về lỗi này.

Mọi thứ ở đây là mình tham khảo từ pwn.college.

NOTE : Học về dạng này để không bị lẫn lộn với nhau, thì hãy hiểu là : `read` là đọc từ file ngoài vào RAM - biến trong chương trình, còn `write` là ghi từ RAM - biến trong chương trình ra ngoài file hệ thống.

# cấu trúc _IO_FILE

Tên đầy đủ của cấu trúc được sử dụng đó là struct _IO_FILE_plus, plus ở đây là có thêm vtable được bổ xung ở cuối

```python
type = struct _IO_FILE {
       Offset         Size
      0      |       4     int _flags; // FLAG
 XXX  4-byte hole      
      8      |       8     char *_IO_read_ptr;
     16      |       8     char *_IO_read_end;
     24      |       8     char *_IO_read_base;
     32      |       8     char *_IO_write_base;
     40      |       8     char *_IO_write_ptr;
     48      |       8     char *_IO_write_end;
     56      |       8     char *_IO_buf_base;
     64      |       8     char *_IO_buf_end;
     72      |       8     char *_IO_save_base;
     80      |       8     char *_IO_backup_base;
     88      |       8     char *_IO_save_end;
     96      |       8     struct _IO_marker *_markers;
    104      |       8     struct _IO_FILE *_chain;
    112      |       4     int _fileno;
    116      |       4     int _flags2 : 24;
    119      |       1     char _short_backupbuf[1];
    120      |       8     __off_t _old_offset;
    128      |       2     unsigned short _cur_column;
    130      |       1     signed char _vtable_offset;
    131      |       1     char _shortbuf[1];
 XXX  4-byte hole      
    136      |       8     _IO_lock_t *_lock;
    144      |       8     __off64_t _offset;
    152      |       8     struct _IO_codecvt *_codecvt;
    160      |       8     struct _IO_wide_data *_wide_data;
    168      |       8     struct _IO_FILE *_freeres_list;
    176      |       8     void *_freeres_buf;
    184      |       8     struct _IO_FILE **_prevchain;
    192      |       4     int _mode;
    196      |       4     int _unused3;
    200      |       8     __uint64_t _total_written;
    208      |       8     char _unused2[8];
    216      |       8     const struct _IO_jump_t *vtable;
total size (bytes):  224 
```

Và ta cần chú ý các trường quan trọng : 

`flags:` 16 bit cao thường chứa _IO_MAGIC, Các bit thấp mô tả trạng thái stream, nó là flags nên sẽ cố định các giá trị. Ví dụ : `0xfbad0000`, nó set cho luồng quyền đọc, ghi, có buffer hay không,…

`_IO_read_ptr:` Con trỏ dùng để trỏ đến các byte tiếp theo sẽ được đọc trong bộ đệm

`_IO_read_end:` Con trỏ chỉ cuối vùng bộ đệm sẽ được đọc

`_IO_read_base:` Con trỏ chỉ đầu vùng bộ đệm sẽ được đọc

`_IO_write_base:` Con trỏ chỉ đầu vùng bộ đệmsẽ được ghi

`_IO_write_ptr:` Con trỏ này để xác định 2 cái với 2 biến còn lại, đó là `write_ptr - write_base` là số byte đã được copy từ biến `buf` vào `write_base`, `write_end - write_ptr` là số khoảng trống còn lại trong bộ đệm phục vụ cho việc ghi.

`_IO_write_end:` Con trỏ chỉ cuối vùng bộ đệm sẽ được ghi

`_IO_buf_base:` Con trỏ chỉ đầu vùng bộ đệm

`_IO_buf_end:` Con trỏ chỉ cuối vùng bộ đệm

`_chain:` Với cơ chế là con trỏ FILE stream này sẽ trỏ tới con trỏ tiếp theo để lưu danh sách các Stream như linked list, vùng này sẽ trỏ tới các vùng stream tiếp theo

`_fileno:` Đây là File Descriptor của stream, stdin,out,err sẽ là 0,1,2 và bắt đầu với những file khác là 3

`_lock:` để bảo vệ file tránh việc 2 luồng đều sửa chung 1 file tại 1 thời điểm, giá trị hợp lệ của nó là 1 địa chỉ trỏ tới 1 vùng nhớ ghi được và có giá trị = 0 tại thời điểm đó.

`_IO_wide_data:` Trỏ đến cấu trúc quản lý wide-character stream, có cấu trúc **gần giống** với _IO_FILE_plus đây là 1 thằng quan trọng trong việc khai thác.

`vtable:` vtable này giống như C++, đó là 1 bảng lưu địa chỉ các hàm cần được sài.

đây là những vùng quan trọng, dài nhưng khi hiểu được luồng thực thi của FILE thì sẽ nhớ được các giá trị này.

# Cách chương trình hoạt động với _IO_FILE

## fread(dst, 1, N, fp)

Xét 6 trường của _IO_FILE : `_IO_read_ptr, _IO_read_base, _IO_read_end, _IO_buf_base, _IO_buf_end`

từ `buf_base` đến `buf_end` hãy hình dung nó là internal buffer hỗ trợ cho cấu trúc _IO_FILE, nó được sài chung cho `fread()` và `fwrite()`, thường được cấp phát bằng 1 page (~0x1000) byte.

Khi syscall read được gọi, đọc tối đa 0x1000 byte từ file vào vùng nhớ `buf_base`

Lúc đó `read_base = buf_base` , `read_end *≤* buf_end` và *chỉ mới nạp vào bộ đệm chứ chưa đưa vào biến  `dst`, và đặt con trỏ `read_ptr = read_base`* 

giả sử với mỗi lần copy thì sẽ dịch chuyển con trỏ `read_ptr` lên để copy tiếp, cho đến khi `read_ptr = read_end`

`memcpy(dst, read_ptr, N)`

 con trỏ `read_ptr` có chức năng trỏ tới vùng nhớ cần copy tiếp trong `buf_base, buf_end`.Khi `read_ptr ≥ read_end`, gọi syscall read để reset lại các con trỏ hoặc để đọc trực tiếp từ `fd vào dst`, `read_base=buf_base, read_end ≤ buf_end` (giả sử ở đây là 0x200 byte < 0x1000 byte thì nó sẽ không nằm cùng với `buf_end`), `read_ptr = read_base` và tiếp tục cho lần copy vào dst tiếp theo. → số byte chưa copy từ read_ptr vào dst

### Workflow chính

```python
have : số byte đã được nạp vào `read_base`, nhưng chưa được đọc vào `dst`.
have = read_end - read_ptr
want: số byte còn lại ở chưa copy từ buf_ptr vào dst, ví dụ:
buf_ptr trỏ tới địa chỉ có chuỗi “12345”, đã copy vào `dst` “123”, want còn lại là “45” 

                    fread(dst, 1, N, fp)
                              │
                              ▼
                  gọi vtable slot: xsgetn
                              │
                              ▼
                     _IO_file_xsgetn
                              │
                have = read_end - read_ptr
                              │
             ┌────────────────┴─────────────────┐
             │                                  │
        have > 0                            have == 0
             │                                  │
             ▼                                  ▼
 memcpy(dst, read_ptr, N)        So sánh N còn thiếu với
 read_ptr += N                          buf_end - buf_base
             │                                  │
             │                    ┌─────────────┴──────────────┐
             │                    │                            │
             │            remaining < buffer_size     remaining >= buffer_size
             │                    │                            │
             │                    ▼                            ▼
             │               underflow                 direct syscall
             │                    │                            │
             │                    ▼                            ▼
             │ read(fp->_fileno,buf_base,buffer_size)(1)   read(fp->_fileno, dst, count)
             │                    │
             │                    ▼
             │       memcpy(dst, buf_base/read_ptr, N)
             │
             ▼
         fread hoàn thành
```

## fwrite(src, 1, N, fp)

Xét 6 trường của _IO_FILE : `_IO_write_ptr, _IO_write_base, _IO_write_end, _IO_buf_base, _IO_buf_end`

Với `buf_base` và `buf_end` thì đã giải thích ở trên, tới với `write_ptr` và `write_end` cũng tương tự chức năng, chỉ là với read thì đọc từ file vào biến `dst`, thì write ghi từ `src`ra ngoài file.

Trước khi gọi syscall write, sẽ copy từ `src` vào vùng nhớ `write_base` và dịch chuyển con trỏ `write_ptr`

`memcpy(write_ptr, src, N)`

Khi `write_ptr = write_end` , nghĩa là đã ghi được 0x1000 byte từ biến src vào `internal buffer` ,khi đó hàm `flush()` được gọi, đồng nghĩa gọi tới syscall write để đẩy toàn bộ dữ liệu từ internal buffer vào file. Và khi đó reset lại các con trỏ tương tự như hàm `fread()`.  

### Workflow chính

```python
free_space : Write sẽ ghi từ `src` vào `write_base`, đây vùng nhớ trống, chưa được nạp vào trong `write_base`

remaining : Là số byte chưa được copy từ `src` vào `write_base`

take : số byte sẽ copy từ src vào `write_base`

                    fwrite(src, 1, N, fp)
                              │
                              ▼
                  gọi vtable slot: xsputn
                              │
                              ▼
                     _IO_file_xsputn
                              │
          free_space = write_end - write_ptr
                              │
             ┌────────────────┴─────────────────┐
             │                                  │
       free_space > 0                      free_space == 0 
             │                                  │
             ▼                                  │
 memcpy(write_ptr, src, N)                      │
 write_ptr += N                                 │
 src       += N                                 │
 remaining -= N                                 │
             │                                  │
             └────────────────┬─────────────────┘
                              │
                       remaining == 0?
                 ┌────────────┴────────────┐
                 │                         │
                Có                       Không (nghĩa là còn byte từ src,                                │                               chưa ghi được hết nên sẽ cần                            │                               gọi hàm cấp phát thêm bộ nhớ)
                 │                         │
                 ▼                         ▼
(return)trả về số byte đã ghi được     gọi overflow(fp, EOF)
                                           │
                                           ▼
                                  _IO_file_overflow
                                           │
                    kiểm tra mode / flags / buffer (kiểm tra buffer con chỗ                                                                không, nếu còn thì copy, nếu                                                            hết thì flush, reset con trỏ                                                            rồi mới copy, đây là hướng ta sẽ đi)
                                           │
                                           ▼
                               flush vùng pending:
                              write_base, write_ptr)
                                           │
                                           ▼
                write(fp->_fileno, write_base,write_ptr-write_base) (2)
                                           │
                                           ▼
                              reset write pointers
                                           │
                                           ▼
                         còn remaining của fwrite?
                             ┌─────────────┴─────────────┐
                             │                           │
                      block lớn                    phần dư nhỏ
                             │                           │
                             ▼                           ▼
             write(fp->_fileno, src, size)      memcpy(src → FILE buffer)
```

tham khảo nguồn : [File Struct - Introduction - Google Trang trình bày

# Cơ chế khai thác với _IO_FILE

Giả sử ta ghi đè được các giá trị trong cấu trúc `FILE *fp`. Kiểm soát được các giá trị trong đó.

## Ghi tại 1 vùng nhớ với fread

Khi đó goal của ta là ghi được giá trị vào 1 vùng nhớ tùy ý, với câu lệnh tựa như `read(0,buf,N)`, ta sẽ setup điều kiện và các giá trị để workflow đi đến được câu lệnh này  :

`read(fp->_fileno, buf_base, buffer_size)(1)`

Nếu set `fileno = 0` ~ `stdin`

`buf_base` là 1 địa chỉ muốn ghi

`buffer_size = buf_end - buf_base` 

Đó là do 3 trường ta điều khiển được 

⇒ Ta có thể điều hướng câu lệnh trên thành hàm nhập với địa chỉ tùy ý.

Để đi được tới nhánh đó và để câu lệnh hoạt động như cách ta mong muốn thì cần setup điều kiện cụ thể, giải thích đơn giản luồng muốn đi, đó là mình sẽ cần chương trình fill internal buffer, đồng nghĩa khi buffer cạn, nghĩa là khi read_ptr = read_end. Khi đó chương trình sẽ gọi 

`read(fp→_fileno, buf_base, 0x1000)`

và như đã nói ở trên thì mình điều khiển được 2 giá trị này nên sẽ biến nó thành luồng :

`read(0, địa chỉ mình muốn ghi, 0x1000)`

Điều kiện cụ thể để trigger đó là :

- Set giá trị `flag` để có thể ghi
- read_ptr = read_end → Trigger nhánh fill `internal buffer`
- buf_base = `address` → buf_base là địa chỉ mà ta muốn ghi vào
- buf_end = `address+offset` → phục vụ cho việc điều chỉnh số lượng byte nhập từ input
- `buffer_size = buf_end - buf_base > N` với N là số byte mà code yêu cầu, việc này bắt buộc chương trình đi qua internal buffer, vì giả sử `N=0x2000` byte, mà internal buffer tối đa chỉ được `0x1000` byte thì chương trình sẽ đọc trực tiếp : `read(fp→_fileno,src,0x2000)`

Tóm lại là phải bắt nó sài internal buffer, khi đó nó sẽ sài địa chỉ `buf_base` - 1 địa chỉ do mình kiểm soát được để ghi.

## Đọc tại 1 vùng nhớ với fwrite

Với nhánh flow ta cần là để in ra giá trị tại 1 địa chỉ, ta cần câu lệnh in ra tựa như : `write(1,buf,N)` 

Để làm được điều đó thì cần setup điều kiện và giá trị để workflow đi đến được câu lệnh này:

`write(fp->_fileno, write_base,write_ptr-write_base)(2)`

Nếu set fileno = 1 ~ stdout

buf_base là 1 địa chỉ muốn đọc

`buffer_size = buf_end - buf_base`

Đó là do 3 trường ta điều khiển được 

⇒ Ta có thể điều hướng câu lệnh trên với địa chỉ tùy ý.

Để đi được tới nhánh đó và để câu lệnh hoạt động như cách ta mong muốn thì cũng cần setup điều kiện cụ thể, giải thích đơn giản là để đi được nhánh đó ta cần chỉnh địa chỉ của write_base thành địa chỉ muốn in ra và đợi hoặc ép nó flush dữ liệu, để chắc chắn nó flush dữ liệu thì ta phải làm cho write buffer cạn, không còn chỗ để copy vào internal buffer, nghĩa là set con trỏ `write_ptr = write_end` (ở đây ta bỏ qua điều kiện là còn byte chưa xử lí, vì nếu hết byte thì chương trình return trước thay vì flush dữ liệu), khi hết chỗ nó sẽ flush dữ liệu, reset con trỏ để phục vụ cho việc write tiếp. Khi đó điều kiện cụ thể như sau :

- Set giá trị `flag` để có quyền write
- `write_ptr = write_end` → ép nó flush dữ liệu
- `read_end = write_base` → đây là điều kiện để tránh hệ thống đi vào hàm lseek(), có khả năng làm payload ta truyền vào không hoạt động, debug sẽ thấy.
- `write_base` = <address> → buf_base là địa chỉ mà ta muốn in ra.
- `write_ptr` = <address+offset> → phục vụ cho số lượng byte muốn in ra.

## Ret2win với vtable

### Workflow tấn công vtable

```python
                        TRIGGER
             exit / fflush / wide output
                           │
                           ▼
                  normal virtual call
                           │
                           ▼
       FILE->vtable->__overflow / __xsputn / ...
                           │
                           │ IO_validate_vtable
                           ▼
               một vtable hợp lệ trong glibc
                           │
                           ▼
                  _IO_wfile_overflow
                           │
                           ▼
             sử dụng fp->_wide_data fields
                           │
                           ▼
                    _IO_wdoallocbuf
                           │
                           ▼
               _IO_WDOALLOCATE(fp)
                           │
                           ▼
 fp->_wide_data->_wide_vtable->__doallocate(fp)
                           │
                           ▼
             indirect call do attacker chọn
```

Đại khái là với hàm fread() fwrite() bình thường nó sẽ gọi tới virtual table là  _IO_file_overflow, nhưng thằng này lại được valid bởi hệ thống bằng cách kiểm tra địa chỉ của vtable có phải thuộc vùng nhớ hệ thống hay không, nên nếu ghi đè nó bằng địa chỉ khác thì sẽ bị exit.

Thay vào đó cũng có 1 thằng tựa như `_IO_FILE` đó là `wide_data`, cấu trúc gần giống, và cách gọi vtable cũng như `_IO_FILE` nhưng nếu gọi tới vtable của `wide_data` sẽ không bị validate và nếu thay vtable của nó bằng 1 fake vtable khác thì sẽ không detect được. hay ở chỗ là các offset gọi tới từ vtable cũng tựa nhau, nên không detect được. 

Workflow của hướng attack là như này : 

- Ghi đè `_IO_file_jumps` - vtable của `_IO_FILE` bằng `_IO_wfile_jumps` - vtable của `wide_data`
- IO_FILE gọi tới `_IO_wfile_jumps + offset` , với offset tựa như vtable cũ.
- setup các vùng nhớ trong internal buffer để trigger được hàm `_IO_wfile_overflow`
- Khi đó `_IO_wfile_overflow` cần gọi tới `_IO_wdoallocbuf` bằng cách lấy ra vtable của wide_data + offset.
- Fake `wide_data` và vtable của `wide_data` ta sẽ gọi được hàm mong muốn.

# Kết

đây chỉ là 1 đống lý thuyết mình xây dựng lại từ việc học fsop để dễ hình dung workflow, 1 phần để luyện về góc nhìn và mindset khi nhìn vào 1 workflow trông có vẻ bình thường nhưng nếu điều khiển được các giá trị của nó thì sẽ điều khiển luồng thực thi như ý mình muốn. Và ghi lại để mình vừa ghi nhớ và có quên thì cũng nhìn lại được :> Have a good pwn.