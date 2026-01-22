---
title: '[Dummy] - How different between Pointer and Array from Assembly Perspective ? '
tag: ['Dummy']
cover: /img/post/dummy/different between Pointer and Array/thumbnail.png
category: ['Dummy']
date: 2025-8-17 00:14:35
author:
    - Vuk3r
---
Everyone always told that “Pointer could be an Array” and “Array can be considered as a Ponter” so why it need to be divided into two concepts ?

I create a simple program for testing and debugging.

```
#include <stdio.h>

int main()
{
    int arr[32] = {0x8, 0x10, 0x18, 0x20};
    int *ptr = arr;
    printf("aaaa");
    printf("a[1]: %d\n", arr[1]);
    printf("*(arr + 1): %d\n", *(arr + 1));

    printf("*(ptr+1): %d\n", *(ptr+1));
    printf("ptr[1]: %d\n", ptr[1]);

}
// gcc -o ./test ./test.c
```

Because of its properties, Array can access its element like Pointer, also in reverse with Pointer. It is prove that Pointer can be an Array and Array can be a Pointer. 

But the difference is how Asm see it ?

### Array

I use Pwndbg for debugging. You can see that how it store my `arr` and `ptr` variables in stack :

![image.png](/img/post/dummy/different-between-Pointer-and-Array/1.png)

with the code :

```bash
int arr[32] = {0x8, 0x10, 0x18, 0x20};
```

Whenever declare an Array, it store the same way with normal variables, it just combine all the variables with the same type in sequence for easily access. So it will store directly in the stack.

![image.png](/img/post/dummy/different-between-Pointer-and-Array/2.png)


You can see this is how `*(arr + 1)` work, Asm call directly to the address of `arr+1` and assign value for `eax`, then `esi`. The computer base on the `arr+0` address and calculate to `arr+1` address for accessing data, just like a Pointer :>

![image.png](/img/post/dummy/different-between-Pointer-and-Array/3.png)


And you can see it the same way when calling in normal way `a[1]`  

![image.png](/img/post/dummy/different-between-Pointer-and-Array/4.png)


### Pointer

`ptr` stored in `rbp-8`, storing `arr+0` address for accessing data.

![image.png](/img/post/dummy/different-between-Pointer-and-Array/5.png)


The code is plus `1` to ptr for next address variable:

```bash
 printf("*(ptr+1): %d\n", *(ptr+1));
```

But the Pointer is the opposite with Array, because of knowing entry address of the array, array can access it easily, but the Pointer need to do some calculate for access is adding `0x4` ( size of `ptr`) like `ptr+1` into address value which `ptr` holding. 

![image.png](/img/post/dummy/different-between-Pointer-and-Array/6.png)


and then, `ptr` can access elements just like array. So you can use `ptr` for accessing data like array :

```bash
printf("ptr[1]: %d\n", ptr[1]);
```

Just like that, easily distinguish between **Pointer** and **Array**. May be it the same, may be it not..