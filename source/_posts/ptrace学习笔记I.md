---
title: ptrace学习笔记I
date: 2018-04-19 23:30:02
tags: 
- Hook
- ptrace
- Linux
---

## 0x00 前言

早就想深入学习下ptrace了，这玩意儿实用性太强，什么注入啊，hook什么的都能用到它，甚至你还能根据ptrace编写自己的调试工具，emmm我也想写一个，呐，这篇文章就是记录下自己学习的过程

<!--more-->

代码参考自https://www.linuxjournal.com/article/6100

(代码稍微做了一下兼容处理，不然会报错0.0)



## 0x01 ptrace常见用法

-  PTRACE_TRACEME

  形式：ptrace(PTRACE_TRACEME,0 ,0 ,0)
  描述：本进程被其父进程所跟踪。其父进程应该希望跟踪子进程。


-  PTRACE_PEEKTEXT,PTRACE_PEEKDATA

  形式：ptrace(PTRACE_PEEKTEXT, pid, addr, data)
  描述：从内存地址中读取一个字节，pid表示被跟踪的子进程，内存地址由addr给出，data为用户变量地址用于返回读到的数据。在Linux（i386）中用户代码段与用户数据段重合所以读取代码段和数据段数据处理是一样的。

- PTRACE_POKETEXT,PTRACE_POKEDATA
  形式：ptrace(PTRACE_POKETEXT, pid, addr, data)
  描述：往内存地址中写入一个字节。pid表示被跟踪的子进程，内存地址由addr给出，data为所要写入的数据。

- TRACE_PEEKUSR
  形式：ptrace(PTRACE_PEEKUSR, pid, addr, data)
  描述：从USER区域中读取一个字节，pid表示被跟踪的子进程，USER区域地址由addr给出，data为用户变量地址用于返回读到的数据。USER结构为core文件的前面一部分，它描述了进程中止时的一些状态，如：寄存器值，代码、数据段大小，代码、数据段开始地址等。在Linux（i386）中通过PTRACE_PEEKUSER和PTRACE_POKEUSR可以访问USER结构的数据有寄存器和调试寄存器。

- PTRACE_POKEUSR
  形式：ptrace(PTRACE_POKEUSR, pid, addr, data)
  描述：往USER区域中写入一个字节，pid表示被跟踪的子进程，USER区域地址由addr给出，data为需写入的数据。

- PTRACE_CONT
  形式：ptrace(PTRACE_CONT, pid, 0, signal)
  描述：继续执行。pid表示被跟踪的子进程，signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。

- PTRACE_SYSCALL
  形式：ptrace(PTRACE_SYS, pid, 0, signal)
  描述：继续执行。pid表示被跟踪的子进程，signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。与PTRACE_CONT不同的是进行系统调用跟踪。在被跟踪进程继续运行直到调用系统调用开始或结束时，被跟踪进程被中止，并通知父进程。

- PTRACE_KILL
  形式：ptrace(PTRACE_KILL,pid)
  描述：杀掉子进程，使它退出。pid表示被跟踪的子进程。

- PTRACE_SINGLESTEP
  形式：ptrace(PTRACE_KILL, pid, 0, signle)
  描述：设置单步执行标志，单步执行一条指令。pid表示被跟踪的子进程。signal为0则忽略引起调试进程中止的信号，若不为0则继续处理信号signal。当被跟踪进程单步执行完一个指令后，被跟踪进程被中止，并通知父进程。

- PTRACE_ATTACH
  形式：ptrace(PTRACE_ATTACH,pid)
  描述：跟踪指定pid 进程。pid表示被跟踪进程。被跟踪进程将成为当前进程的子进程，并进入中止状态。

- PTRACE_DETACH 形式：ptrace(PTRACE_DETACH,pid) 描述：结束跟踪。 pid表示被跟踪的子进程。结束跟踪后被跟踪进程将继续执行。

- PTRACE_GETREGS
  形式：ptrace(PTRACE_GETREGS, pid, 0, data)
  描述：读取寄存器值，pid表示被跟踪的子进程，data为用户变量地址用于返回读到的数据。此功能将读取所有17个基本寄存器的值。

- PTRACE_SETREGS
  形式：ptrace(PTRACE_SETREGS, pid, 0, data)
  描述：设置寄存器值，pid表示被跟踪的子进程，data为用户数据地址。此功能将设置所有17个基本寄存器的值。

- PTRACE_GETFPREGS 形式：ptrace(PTRACE_GETFPREGS, pid, 0, data)
  描述：读取浮点寄存器值，pid表示被跟踪的子进程，data为用户变量地址用于返回读到的数据。此功能将读取所有浮点协处理器387的所有寄存器的值。

- PTRACE_SETFPREGS
  形式：ptrace(PTRACE_SETREGS, pid, 0, data)
  描述：设置浮点寄存器值，pid表示被跟踪的子进程，data为用户数据地址。此功能将设置所有浮点协处理器387的所有寄存器的值。



## 0x02 跟踪ls

```
//
// Created by mask on 18-4-19.
//

#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/user.h>
#include <sys/reg.h>
#include <stdio.h>


int main()
{
    pid_t child;
    long orig_eax;
   

    child = fork();

    if (child==0){
        ptrace(PTRACE_TRACEME,0,NULL,NULL);//子进程告诉内核，让别人跟踪他
        execl("/bin/ls","ls",NULL);
    } else{
        wait(NULL);//wait函数等待内核通知，子进程结束后就会收到
        orig_eax = ptrace(PTRACE_PEEKUSER,child,4*ORIG_EAX,NULL;//读取子进程运行时eax寄存器的值
        printf("The child made a system call %ld\n",orig_eax);
        ptrace(PTRACE_CONT,child,NULL,NULL);//让子进程继续运行
    }
    return 0;

}

```

来看一下效果,execve的调用号就是11

{%asset_img 0.png%}

## 0x03 进一步系统调用查看参数

```
//
// Created by mask on 18-4-19.
//

#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/user.h>
#include <sys/reg.h>
#include <stdio.h>
#include <syscall.h>

int main()
{
    pid_t child;
    long orig_eax, eax;
    long params[3];
    int status;
    int insyscall =0;
    child = fork();
    if(child ==0){
        ptrace(PTRACE_TRACEME,0, NULL, NULL);//子进程告诉内核让别人来跟踪
        execl("/bin/ls","ls", NULL);
    } else {
        while(1) {
            wait(&status);
            if(WIFEXITED(status))/*wait函数使用status变量来检查子进程是否已退出。它是用来判断子进程是被ptrace暂停掉还是已经运行结束并退出。有一组宏可以通过status的值来判断进程的状态，比如WIFEXITED等*/
                break;
            orig_eax = ptrace(PTRACE_PEEKUSER,
                              child,4* ORIG_EAX, NULL);
            if(orig_eax == SYS_write) {    //如果是SYS_write调用
                if(insyscall == 0) {
                    /* Syscall entry */
                    insyscall =1;
                    //获取参数
                    params[0]= ptrace(PTRACE_PEEKUSER,
                                      child,4* EBX,
                                      NULL);
                    params[1]= ptrace(PTRACE_PEEKUSER,
                                      child,4* ECX,
                                      NULL);
                    params[2]= ptrace(PTRACE_PEEKUSER,
                                      child,4* EDX,
                                      NULL);
                    printf("Write called with "
                           "%ld, %ld, %ld\n",
                           params[0], params[1],
                           params[2]);
                } else {/* Syscall exit */
                    eax = ptrace(PTRACE_PEEKUSER,
                                 child,4* EAX, NULL);
                    printf("Write returned "
                           "with %ld\n", eax);
                    insyscall =0;
                }
            }
            ptrace(PTRACE_SYSCALL,
                   child, NULL, NULL); /*使用PTRACE_SYSCALL作为ptrace的第一个参数，使内核在子进程做出系统调用或者准备退出的时候暂停它。*/
        }
    }
    return 0;
}


```



运行看下效果

{%asset_img 1.png%}

这里我们执行的是ls命令，为什么会调用write呢，我们用strace跟踪下系统调用看看

呐，确实是调用了

{%asset_img 2.png%}

换一种方式实现，ptrace有专门的方式去读取系统调用或者进程终止时的寄存器的值



```
//
// Created by mask on 18-4-19.
//
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/user.h>
#include <sys/reg.h>
#include <sys/syscall.h>
#include <stdio.h>

int main(){

    pid_t child;
    long orig_eax,eax;
    long params[3];
    int status;
    int insyscall = 0;
    struct user_regs_struct regs;
    child = fork();

    if (child==0){
        ptrace(PTRACE_TRACEME,0,NULL,NULL);
        execl("/bin/ls","ls",NULL);
    } else{
        while (1){
            wait(&status);
            if (WIFEXITED(status))/*wait函数使用status变量来检查子进程是否已退出。它是用来判断子进程是被ptrace暂停掉还是已经运行结束并退出。*/
                break;
            orig_eax = ptrace(PTRACE_PEEKUSER,child,4*ORIG_EAX,NULL);/*PTRACE_PEEKUSER来察看write系统调用的参数*/
            if (orig_eax==SYS_write){
                if (insyscall==0){
                    insyscall = 1;
                    ptrace(PTRACE_GETREGS,child,NULL,&regs);/*通过将PTRACE_PEEKUSER作为ptrace 的第一个参数进行调用，可以取得与子进程相关的寄存器值。*/

                    printf("Write called with %ld,%ld,%ld\n",regs.ebx,regs.ecx,regs.edx);


                } else{
                /*说明程序已经退出*/
                    eax = ptrace(PTRACE_PEEKUSER,child,4*EAX,NULL);
                    printf("Write returned with %ld\n",eax);/*返回值保存在eax寄存器*/
                    insyscall = 0;
                }
            }
            ptrace(PTRACE_SYSCALL,child,NULL,NULL);/*使用PTRACE_SYSCALL作为ptrace的第一个参数，使内核在子进程做出系统调用或者准备退出的时候暂停它。*/
        }

    }
    return 0;

}

```

看下效果

{%asset_img 3.png%}



