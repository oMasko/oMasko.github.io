---
title: 一道so加密的逆向题
date: 2018-06-07 01:03:04
tags: Android

---



## 0x00 前言

看雪上看到的一道题，自己下载下来做了下



## 0x01 初步分析

JEB打开，可以发现加载了net库，并且调用了一个native函数upload,那么直接拖入IDA中开始分析so

{%asset_img 2.png%}

{%asset_img 0.png%}

{%asset_img 1.png%}

可以发现so文件加密了，那么没法静态分析了

那么动态调试吧！！

<!--more-->

## 0x02 动态调试

这里其实就是需要动态运行时去解密so文件然后再分析，那么可以在linker下断点，这样就能在加载so文件之前断下来

首先先将手机里/system/bin/linker拖出来，IDA对其分析

首先搜索这串字符串，找到需要下断点的地方

{%asset_img 3.png%}

记下偏移0x274C

{%asset_img 4.png%}

然后开始IDA附加应用进程开始调试，具体步骤就不罗嗦了，先找到需要下断点的地方，加载的基址加上之前的偏移

{%asset_img 5.png%}

下断

{%asset_img 6.png%}

运行，这里已经断了下来，那么F7跟进查看

{%asset_img 7.png%}

此处已经运行在libnet.so中了，我们一步一步跟进

一路F8下去，友回到了linker中，此时libnet.so应该已经解密完了

{%asset_img 8.png%}

{%asset_img 9.png%}

跳到libnet.so中去看看

此时JNI_Onload函数已经解密

{%asset_img 10.png%}

那么进行下结构体修复，这样便于分析

{%asset_img 11.png%}

那么很清楚了，这里动态注册的函数便是java层看到的函数upload，跟过去看看是否正确

此处IDA没有识别出来

{%asset_img 12.png%}

直接右键->Structure->JNINativeMethod

{%asset_img 13.png%}

跟到函数地址查看：

F5报错了，那么先Create Function，直接快捷键P即可，这里还是报错了

{%asset_img 14.png%}

那么跟过去，一次性把这些全都make code (快捷键C)试试

{%asset_img 15.png%}

然后再跳回到之前的函数地址处Create Function即可（这其实就是IDA的一些识别不了的问题，手动解决下即可）

函数没怎么分析，看了下看雪那篇文章，主要就是这两处的异或，其他的一大堆无非就是混淆视听，那么看下这两处异或后结果是啥

{%asset_img 16.png%}

一不小心多摁了几下。。。

{%asset_img 17.png%}







