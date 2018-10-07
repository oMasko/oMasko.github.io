---
title: 手脱早期Ali壳
date: 2018-03-17 19:29:23
tags: Android
---

## 0x00 前言

emmmmm,就是一个记录，原理下次再分析

样本是2014年ali CTF的一道题

<!--more-->

## 0x01 脱壳



{%asset_img 0.png%}

识别出了加了壳，对于一些早期的壳，有一个通杀的办法，那就是在`DvmDexFileOpenPartial`下断点，然后可以获取到dex文件在内存中的基址以及大小，然后利用脚本dump出来，原理下次分析，这次主要是操作，那么IDA动态调试



- 将android_server复制到手机上，并给予运行权限，然后运行：

- 端口转发 

  ```
  adb forword tcp:23946 tcp:23946
  ```

- 然后准备调试应用

  ```
  adb shell am start -D -n com.ali.tg.testapp/.MainActivity
  ```

- IDA附加进程，然后找到`libdvm`模块，并且定位到`DvmDexFileOpenPartial`,下好断点

- java层跑起来 

  ```
  jdb -connect com.sun.jdi.SocketAttach:port=8700,hostname=localhost
  ```

  ​

{%asset_img 1.png%}

然后运行，会停在端点处，继续F8单步，这时候，寄存器R0的值就是dex在内存中的基址，R1就是dex的大小

编写IDC脚本dump下dex:

```
static main(void)

    {

        auto fp, begin, end, dexbyte;

        fp = fopen("f:\\dump.dex", "wb"); //打开或创建一个文件

        begin =  0x75871008;              //dex基址

        end = begin + 0x941fc;            //dex基址 + dex文件大小

        for ( dexbyte = begin; dexbyte < end;dexbyte ++ )

        {

            fputc(Byte(dexbyte), fp);     //按字节将其dump到本地文件中

        }

    }
```

再用010Editor打开就已经可以识别了

{%asset_img 2.png%}

