---
title: Android逆向实实战分析
date: 2018-08-08 01:25:13
tags: Android
---

## 0x00 前言

本来准备写一篇长篇大论从原理到操作都写一写然后拿去投稿的，但是后来就只写了操作，那只好放到博客上自己看看了，原理分析下次再写

<!--more-->

## 0x01 Java层分析

咱先看看Java层，用jadx打开apk

{% asset_img 0.png  %}

很明显，应用被加固了，一般来说，应用加固常用的方法就是外面套一层壳，然后壳程序进行一些解密加载变换的操作，将源应用还原出来，然后进行加载。

那么必然有一个壳程序的入口，那么我们看看入口在哪

{% asset_img 1.png  %}

咱知道，因为Application是早于activity的，那么常见的壳入口一般都会设置在这，这样就能早于源Activity执行一些操作，那么咱跟过去瞅瞅壳在java层都做了些啥

首先这里有一点，那就是attachBaseContext函数是早于onCreate函数执行的，咱们跟过去看看它做了啥

{% asset_img 2.png  %}

咱们看看关键的，跟进这个a函数

{% asset_img 3.png  %}

此处调用了两个主要函数，一个是函数e,一个是native函数load，咱先一个一个来，跟进这个e函数

```
private void e() {
        int i;
        String str;
        int i2 = 0;
        ApplicationInfo applicationInfo = getApplicationInfo();
        String str2 = applicationInfo.dataDir + "/tx_shell";
        b = applicationInfo.sourceDir;
        int i3 = VERSION.SDK_INT;
        if (i3 < 19) {
            i3 = 0;
        } else {
            if (i3 > 19) {
                String[] strArr = Build.SUPPORTED_64_BIT_ABIS;
                if (strArr != null && strArr.length > 1) {
                    i3 = 1;
                }
            } else if (new File("/system/lib64").exists()) {
                i3 = 1;
            }
            i3 = 0;
        }
        String str3 = null;
        if (VERSION.SDK_INT < 21) {
            str3 = Build.CPU_ABI;
        }
        if ((str3 == null || str3.length() < 2) && VERSION.SDK_INT >= 21) {
            str3 = Build.SUPPORTED_ABIS[0];
        }
        if (str3.toLowerCase(Locale.US).contains("x86")) {
            i = 0;
        } else {
            if (VERSION.SDK_INT >= 21) {
                String[] strArr2 = Build.SUPPORTED_ABIS;
                if (strArr2 != null) {
                    i = 1;
                    for (String str4 : strArr2) {
                        if (str4.toLowerCase(Locale.US).contains("x86")) {
                            i = 0;
                        }
                    }
                }
            }
            i = 1;
        }
        int i4 = str3.toLowerCase(Locale.US).contains("mips") ? 1 : 0;
        String str5 = VERSION.SDK_INT > 8 ? applicationInfo.nativeLibraryDir : "/data/data/" + mPKName + "/lib";
        String str6 = "";
        String str7 = "";
        String str8 = "";
        str8 = "";
        if (i != 0) {
            str4 = f() + "-" + c();
        } else {
            str4 = "shellx-" + c();
            str8 = f() + "-" + c();
        }
        str6 = str6 + "lib" + str4 + ".so";
        str8 = str7 + "lib" + str8 + ".so";
        File file = new File(str5 + "/" + str6);
        File file2 = new File(str2 + "/" + str8);
        if (i == 0 && VERSION.SDK_INT < 19) {
            if (!file2.exists()) {
                if (ZipUtil.exist(b, "lib/armeabi/" + str8) == 0) {
                    if (ZipUtil.extract(b, "lib/armeabi/" + str8, str2 + "/" + str8) == 0) {
                    }
                } else if (ZipUtil.extract(b, "lib/armeabi-v7a/" + str8, str2 + "/" + str8) == 0) {
                }
            }
            try {
                Runtime.getRuntime().exec("chmod 700 " + str2 + "/" + str8);
                System.load(str2 + "/" + str8);
            } catch (IOException e) {
                e.printStackTrace();
            }
        } else if (file.exists()) {
            Log.d("SecShell", "start load lib");
            System.loadLibrary(str4);
            Log.d("SecShell", "end load lib");
        } else {
            String str9;
            str5 = "";
            str8 = str2 + "/" + str6;
            if (ZipUtil.exist(b, "lib/" + str3 + "/" + str6) == 0) {
                str9 = "lib/" + str3 + "/" + str6;
            } else {
                if (i3 != 0) {
                    if (i != 0) {
                        if (ZipUtil.exist(b, "lib/arm64-v8a/" + str6) == 0) {
                            str9 = "lib/arm64-v8a/" + str6;
                        } else if (ZipUtil.exist(b, "lib/armeabi/" + str6) == 0) {
                            str9 = "lib/armeabi/" + str6;
                        } else if (ZipUtil.exist(b, "lib/armeabi-v7a/" + str6) == 0) {
                            str9 = "lib/armeabi-v7a/" + str6;
                        }
                    } else if (ZipUtil.exist(b, "lib/x86_64/" + str6) == 0) {
                        str9 = "lib/x86_64/" + str6;
                    } else if (ZipUtil.exist(b, "lib/arm64-v8a/" + str6) == 0) {
                        str9 = "lib/arm64-v8a/" + str6;
                    } else if (ZipUtil.exist(b, "lib/x86/" + str6) == 0) {
                        str9 = "lib/x86/" + str6;
                    } else if (ZipUtil.exist(b, "lib/armeabi/" + str6) == 0) {
                        str9 = "lib/armeabi/" + str6;
                    } else if (ZipUtil.exist(b, "lib/armeabi-v7a/" + str6) == 0) {
                        str9 = "lib/armeabi-v7a/" + str6;
                    }
                } else if (i != 0) {
                    if (i4 == 0 || ZipUtil.exist(b, "lib/mips/" + str6) != 0) {
                        str9 = str5;
                    } else {
                        i2 = 1;
                        str9 = "lib/mips/" + str6;
                    }
                    if (i2 == 0) {
                        if (ZipUtil.exist(b, "lib/armeabi/" + str6) == 0) {
                            str9 = "lib/armeabi/" + str6;
                        } else if (ZipUtil.exist(b, "lib/armeabi-v7a/" + str6) == 0) {
                            str9 = "lib/armeabi-v7a/" + str6;
                        }
                    }
                } else if (ZipUtil.exist(b, "lib/x86/" + str6) == 0) {
                    str9 = "lib/x86/" + str6;
                } else if (ZipUtil.exist(b, "lib/armeabi/" + str6) == 0) {
                    str9 = "lib/armeabi/" + str6;
                } else if (ZipUtil.exist(b, "lib/armeabi-v7a/" + str6) == 0) {
                    str9 = "lib/armeabi-v7a/" + str6;
                }
                str9 = str5;
            }
            if (ZipUtil.extract(b, str9, str8) == 0) {
                try {
                    Runtime.getRuntime().exec("chmod 700 " + str8);
                    System.load(str8);
                } catch (IOException e2) {
                    e2.printStackTrace();
                }
            }
        }
    }

```

有点小长，作用就是加载对于的so库，先是判断一下存不存在，以及根据对应的架构加载相应的so，那么这么多so文件，到底加载了哪个呢，来看这

{% asset_img 4.png  %}

咱们看看f函数以及c函数都返回了啥

{% asset_img 5.png  %}

{% asset_img 6.png  %}

那么很明显了，咱们打开这个so文件进行查看

IDA打开，各种异常报错，先不管，强制打开

{% asset_img 7.png  %}

打开之后发现所有函数都没识别

{% asset_img 8.png  %}

那么很明显，so文件进行了加密，静态分析这条路暂时就走不通了，那么咱这就开始IDA动态分析

## 0x02 so文件动态分析

动态分析的烦处就在于需要找到各种反调试操作然后绕过它，这篇文章会详细讲讲这个

IDA怎样动态调试就不介绍了，要是什么都写一通的话就很累了

附加上调试后，F9运行几次后发现直接异常退出了，那么就存在反调试，

{% asset_img 11.png  %}

这里讲讲几个点

一是.init/init_array

二是jni_Onload函数

一般一些初始化，加解密，反调试操作都会放在这（init_array要早于jni_Onload函数执行），那么咱们先在这两处下断点

将手机下的 `/system/bin/linker`以及`/system/lib/libdvm.so`文件放入IDA中

在linker中搜索 [ Calling %s @ %p for '%s' ]” ，然后在此处下断

{% asset_img 9.png  %}

在libdvm.so的dvmLoadNativeCode 函数下断

{% asset_img 10.png  %}

OK，断点下好后，咱继续开始调试

运行个几次后，开始进入libshella-2.10.1.so文件

{% asset_img 12.png  %}

这里按快捷键P让IDA将它识别成函数

{% asset_img 13.png  %}



之后呢，咱们F5查看伪代码会发现是有两个循环的操作，咱先不知道他是干啥，先慢慢单步执行

然后大概单步N下后发现又回到linker处了

{% asset_img 14.png  %}

那么，大胆猜测，init处的函数应该以及执行完了，那么是不是so已经解密了呢？咱还不确定，不管，先dump下来看看

{% asset_img 15.png  %}

打开dump下来的so文件

依然报异常，先强制打开

{% asset_img 16.png  %}

一个一个看，可以发现大部分函数好像已经解密，但是还是需要修复，因为我们有很多系统函数还是没有识别出来

咱利用so修复工具修复一下section表

修复完后发现jni_Onload函数已经解密，而且大部分系统函数也是别出来了（这里有个地方贼好玩，jni_Onload函数里调用jni_Onload函数，0.0，先不管，后面动态调的时候就能见分晓）

{% asset_img 17.png  %}

这个时候，东瞧瞧，西看看

{% asset_img 18.png  %}

{% asset_img 19.png  %}

{% asset_img 20.png  %}

看的有点晕0.0，好像差不多了

咱刚刚调到哪来着？

哦对了，刚刚执行完init段的解密函数，那么咱现在去jni_Onload函数瞅瞅

为了不让反调试干扰咱们，咱先处理掉这个

这里就直接说操作了，没有为啥，你调个几百行你就晓得哪有反调试了

第一处：

回到刚才解密函数那，此处看起了一个反调试进程，咱把它nop掉

{% asset_img 21.png  %}

咱们可以确认下，以面失误，一步步跟进这个函数

赞和刚才解密出的函数进行个对比，没错，九四它了

{% asset_img 22.png  %}

这里开启了一个线程，具体干了啥，检查了啥，有兴趣都可以更一下，也就那些个操作

我们这直接先把他干掉

{% asset_img 23.png  %}

然后继续运行

进入jni_Onload函数，咱继续对比下

{% asset_img 24.png  %}

此处有个raise(9)的操作，也是反调试杀死进程，咱同上，把此处nop掉

执行完此处的时候，咱们已经获取到了新的jni_Onload地址了，跳到v7处进行查看

{% asset_img 26.png  %}

此处有一个点，Thumb和arm模式的转换，因为IDA并不能每次都那么好地识别，所以需要手动转换（有个好记得办法，地址为奇数，那么就是Thumb)

快捷键alt+g,将value修改为1即可，然后按P创建函数

{% asset_img 27.png  %}

看不懂啊，这写的啥玩意儿？

先别着急，慢慢来看

先导入下JNI常用的几个结构体

{% asset_img 28.png  %}

然后对函数进行查看以及结构体修复然后函数重命名

```
int __fastcall sub_6048B912(JavaVM *a1, int a2, int a3, int a4, int a5, int a6, int a7, int a8, int a9, int a10)
{
  signed int v10; // r4
  JNIEnv *v11; // r6

  v10 = 0;
  v11 = a1;
  a6 = 0;
  if ( Findclass(a1, &a6) )
  {
    if ( Findclass(v11, &a6) )
    {
      if ( Findclass(v11, &a6) )
      {
        if ( Findclass(v11, &a6) )
          return (a10)(v10, a5, a6);
        v10 = 65537;
      }
      else
      {
        v10 = 65538;
      }
    }
    else
    {
      v10 = 65540;
    }
  }
  else
  {
    v10 = 65542;
  }
  if ( a6 )
  {
    string_init();
    sub_6048B8D4(a6);
  }
  return (a10)(v10, a5, a6);
}
```

咱看看这个string_init函数

```
unsigned int string_init()
{
  unsigned int result; // r0

  dword_604A4068 = "java/lang/String";
  dword_604A406C = "getBytes";
  dword_604A4070 = "()[B";
  dword_604A4074 = "android/os/Build$VERSION";
  dword_604A4078 = "SDK_INT";
  dword_604A407C = &unk_6049E707;
  dword_604A408C = "android/content/ContextWrapper";
  dword_604A4080 = "android/app/ActivityThread";
  dword_604A4084 = "currentActivityThread";
  dword_604A4090 = "getPackageName";
  dword_604A4088 = "()Landroid/app/ActivityThread;";
  dword_604A40D0 = "Ljava/lang/ClassLoader;";
  dword_604A4094 = "()Ljava/lang/String;";
  dword_604A40D8 = "java/lang/ClassLoader";
  dword_604A4098 = "mPackages";
  dword_604A409C = "Ljava/util/HashMap;";
  dword_604A40A0 = "Landroid/util/ArrayMap;";
  dword_604A40A4 = "java/util/HashMap";
  dword_604A40A8 = "android/util/ArrayMap";
  dword_604A40AC = "(Ljava/lang/Object;)Ljava/lang/Object;";
  dword_604A40B0 = "()Ljava/lang/Object;";
  dword_604A40B4 = &unk_6049E821;
  dword_604A40B8 = "java/lang/ref/WeakReference";
  dword_604A40BC = "android/app/ActivityThread$PackageInfo";
  dword_604A40C0 = "Landroid/app/ActivityThread$PackageInfo;";
  dword_604A40C4 = "android/app/LoadedApk";
  dword_604A40C8 = "Landroid/app/LoadedApk;";
  dword_604A40CC = "mClassLoader";
  dword_604A40D4 = "()Ljava/lang/ClassLoader;";
  dword_604A40DC = "getApplicationInfo";
  dword_604A40E0 = "()Landroid/content/pm/ApplicationInfo;";
  dword_604A40E4 = "android/content/pm/ApplicationInfo";
  dword_604A40E8 = "sourceDir";
  dword_604A40EC = "Ljava/lang/String;";
  dword_604A40F0 = "getParent";
  dword_604A40F4 = "<init>";
  dword_604A40F8 = "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Ljava/lang/ClassLoader;)V";
  dword_604A40FC = "([BLjava/lang/String;)V";
  dword_604A4100 = "/system/lib/libdvm.so";
  dword_604A4104 = "dexFileParse";
  dword_604A4108 = "_Z12dexFileParsePKhji";
  dword_604A410C = "dvmDexFileOpenPartial";
  dword_604A4110 = "_Z21dvmDexFileOpenPartialPKviPP6DvmDex";
  dword_604A4114 = "dexCreateClassLookup";
  dword_604A4118 = "_Z20dexCreateClassLookupP7DexFile";
  dword_604A411C = "dexSwapAndVerify";
  dword_604A4120 = "dexFixByteOrdering";
  dword_604A4124 = "_Z16dexSwapAndVerifyPhi";
  dword_604A4128 = "dvmDexFileFree";
  dword_604A412C = "_Z14dvmDexFileFreeP6DvmDex";
  dword_604A4130 = "dalvik/system/DexClassLoader";
  dword_604A4134 = "mDexs";
  dword_604A4138 = "[Ldalvik/system/DexFile;";
  dword_604A413C = "dalvik/system/DexPathList";
  dword_604A4140 = "dexElements";
  dword_604A4144 = "[Ldalvik/system/DexPathList$Element;";
  dword_604A4148 = "dalvik/system/DexPathList$Element";
  dword_604A414C = "dexFile";
  dword_604A4150 = "Ldalvik/system/DexFile;";
  dword_604A4154 = "dalvik/system/DexFile";
  dword_604A4158 = "dalvik/system/BaseDexClassLoader";
  dword_604A415C = "pathList";
  dword_604A4160 = "Ldalvik/system/DexPathList;";
  dword_604A4168 = "()Landroid/app/ActivityThread;";
  dword_604A416C = "currentActivityThread";
  dword_604A4164 = "android/app/ActivityThread";
  dword_604A4170 = "mBoundApplication";
  dword_604A4184 = "Landroid/app/Application;";
  dword_604A4174 = "Landroid/app/ActivityThread$AppBindData;";
  dword_604A4188 = "android/app/Application";
  dword_604A4178 = "android/app/ActivityThread$AppBindData";
  dword_604A418C = "mAllApplications";
  dword_604A417C = "info";
  dword_604A4190 = "Ljava/util/ArrayList;";
  dword_604A4180 = "mInitialApplication";
  dword_604A4194 = "remove";
  dword_604A41BC = "mInitialApplication";
  dword_604A4198 = "(Ljava/lang/Object;)Z";
  dword_604A41C0 = "mProviderMap";
  dword_604A419C = "java/util/ArrayList";
  dword_604A41C4 = "values";
  dword_604A41A0 = "mApplicationInfo";
  dword_604A41C8 = "()Ljava/util/Collection;";
  dword_604A41A4 = "Landroid/content/pm/ApplicationInfo;";
  dword_604A41CC = "java/util/Collection";
  dword_604A41A8 = "className";
  dword_604A41D0 = "iterator";
  dword_604A41AC = "appInfo";
  dword_604A41D4 = "()Ljava/util/Iterator;";
  dword_604A41B0 = "mApplication";
  dword_604A41D8 = "java/util/Iterator";
  dword_604A41B4 = "makeApplication";
  dword_604A41DC = "hasNext";
  dword_604A41B8 = "(ZLandroid/app/Instrumentation;)Landroid/app/Application;";
  dword_604A41E0 = &unk_6049EE4E;
  dword_604A41E4 = "next";
  dword_604A41E8 = "android/app/ActivityThread$ProviderClientRecord";
  dword_604A41EC = "android/app/ActivityThread$ProviderRecord";
  dword_604A41F0 = "mLocalProvider";
  dword_604A41F4 = "Landroid/content/ContentProvider;";
  dword_604A41F8 = "android/content/ContentProvider";
  dword_604A41FC = "mContext";
  dword_604A4200 = "Landroid/content/Context;";
  dword_604A4204 = "onCreate";
  dword_604A4208 = &unk_6049EF2E;
  dword_604A420C = "(Ljava/lang/String;I)Z";
  dword_604A4210 = "mCookie";
  dword_604A4214 = "ensureInit";
  dword_604A4218 = &unk_6049EF2E;
  dword_604A421C = "assets/shell.dat";
  dword_604A4220 = "assets";
  dword_604A4224 = "java/lang/ClassLoader";
  result = 0x1C0u;
  dword_604A4228 = "parent";
  dword_604A422C = "Ljava/lang/ClassLoader;";
  dword_604A4230 = "dalvik/system/PathClassLoader";
  dword_604A4234 = "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/ClassLoader;)V";
  dword_604A4238 = "shell.dat";
  dword_604A423C = "assets/meta-data/DEG.BIN";
  return result;
}
```

这里对字符串进行了初始化的操作，方便后面的函数调用

再跟进下一个函数

```
jclass __fastcall sub_6048B8D4(int a1)
{
  jclass v1; // r4

  v1 = RegisterNatives0(a1, "com/tencent/StubShell/TxAppEntry", &unk_604A4004, 5);
  if ( v1 )
    return 1;
  android_P();
  return v1;
}
```

这里进行了native函数的注册

跟过去看下注册了哪些函数

{% asset_img 30.png  %}

此处IDA没有识别出来 ,

直接右键->Structure->JNINativeMethod 

{% asset_img 31.png  %}

咱们跟过去查看下(有几个函数名进行了重命名)

```
int *__fastcall sub_60492C54(JNIEnv *a1, int a2, int a3)
{
  JNIEnv *v3; // r4
  int v4; // r6
  int v5; // r0
  int *result; // r0
  BOOL v7; // r5

  v3 = a1;
  v4 = a3;
  v5 = sub_6048DDFC();
  (unk_6049C20C)(v5);
  android_P();
  result = (unk_6048DFF4)(v3);
  if ( result )
  {
    v7 = sub_6048F59C(v3);
    (unk_6048B34C)(v3);
    if ( v7 )
      result = art_load(v3, v4, 0, 0);
    else
      result = dalvik_load(v3, v4, 0);
  }
  return result;
}
```

查看下dalvik_load

```
int *__fastcall dalvik_load(int a1, int a2, int a3)
{
  int v3; // r6
  int v4; // r0
  int v5; // r4
  int v6; // r0
  int v7; // r0
  int v8; // r0
  _DWORD *v9; // r7
  int v10; // r0
  int v11; // r0
  int v12; // r5
  _DWORD *v13; // r3
  int v14; // r4
  int v15; // r1
  int v16; // r4
  int v17; // ST14_4
  int v18; // r0
  int v19; // r1
  int v20; // r1
  int v21; // r5
  int v22; // ST00_4
  int v23; // r0
  int v24; // ST00_4
  int v25; // r5
  int v26; // r4
  int v27; // r5
  int v28; // r4
  int v29; // r2
  int v30; // r3
  int v31; // r3
  int v32; // r3
  int v33; // r5
  int v34; // r2
  int v35; // r5
  int v36; // r2
  int v37; // r0
  int v38; // r4
  int v39; // r0
  int v40; // r5
  int m; // r4
  int v42; // r0
  int v43; // r0
  int i; // r4
  int v45; // r0
  int v46; // r5
  int v47; // r0
  int j; // r4
  int v49; // r0
  int v50; // r5
  int v51; // r4
  int v52; // r0
  int k; // r4
  int v54; // r0
  int v55; // r5
  int v56; // r4
  int v57; // r7
  int v58; // r0
  int v59; // r0
  int l; // r4
  int v61; // r0
  _DWORD *v62; // r4
  int v63; // r0
  int *result; // r0
  _DWORD *v65; // [sp+10h] [bp-180h]
  int v66; // [sp+10h] [bp-180h]
  int v67; // [sp+14h] [bp-17Ch]
  int v68; // [sp+14h] [bp-17Ch]
  int v69; // [sp+14h] [bp-17Ch]
  int v70; // [sp+18h] [bp-178h]
  int v71; // [sp+18h] [bp-178h]
  int v72; // [sp+1Ch] [bp-174h]
  signed int v73; // [sp+1Ch] [bp-174h]
  int v74; // [sp+1Ch] [bp-174h]
  int v75; // [sp+20h] [bp-170h]
  int v76; // [sp+24h] [bp-16Ch]
  int v77; // [sp+24h] [bp-16Ch]
  int v78; // [sp+28h] [bp-168h]
  int v79; // [sp+2Ch] [bp-164h]
  signed int v80; // [sp+2Ch] [bp-164h]
  int v81; // [sp+30h] [bp-160h]
  int v82; // [sp+34h] [bp-15Ch]
  int v83; // [sp+38h] [bp-158h]
  int v84; // [sp+3Ch] [bp-154h]
  int v85; // [sp+40h] [bp-150h]
  int v86; // [sp+44h] [bp-14Ch]
  int v87; // [sp+4Ch] [bp-144h]
  int v88; // [sp+50h] [bp-140h]
  int v89; // [sp+54h] [bp-13Ch]
  int *v90; // [sp+5Ch] [bp-134h]
  int v91; // [sp+60h] [bp-130h]
  char v92; // [sp+64h] [bp-12Ch]
  int v93; // [sp+78h] [bp-118h]
  char v94; // [sp+7Ch] [bp-114h]
  int v95; // [sp+90h] [bp-100h]
  char v96; // [sp+94h] [bp-FCh]
  int v97; // [sp+A8h] [bp-E8h]
  int v98; // [sp+B4h] [bp-DCh]
  int v99; // [sp+174h] [bp-1Ch]

  v3 = a1;
  v72 = a3;
  v99 = *off_604A3DA4;
  v90 = off_604A3DA4;
  v75 = (unk_6048BCFC)(a1, a2, "android/content/Context", "getClassLoader", "()Ljava/lang/ClassLoader;");
  if ( !v75 )
    goto LABEL_102;
  v4 = (unk_6048B984)(v3, "com/tencent/StubShell/TxAppEntry");
  v5 = v4;
  v6 = (unk_6048D96C)(v3, v4, "mSrcPath", "Ljava/lang/String;");
  v83 = (unk_6048D97A)(v3, v5, v6);
  v79 = (unk_6048BEC4)(v3, v83);
  v7 = (unk_6048D96C)(v3, v5, "mPKName", "Ljava/lang/String;");
  v8 = (unk_6048D97A)(v3, v5, v7);
  v89 = (unk_6048BEC4)(v3, v8);
  v9 = unk_604A3DA8;
  if ( dword_604A46E8 <= 10 )
  {
    v11 = (unk_6048B984)(v3, *(unk_604A3DA8 + 456));
    v12 = (unk_6048D916)(v3, v75, v11);
    v13 = v9 + 51;
    if ( v12 )
    {
      v78 = (unk_6048BF50)(v3, v75, v9[114], *v13, v9[52]);
      v82 = (unk_6048BF50)(v3, v75, v9[114], "mPaths", "[Ljava/lang/String;");
      v84 = (unk_6048BF50)(v3, v75, v9[114], "mFiles", "[Ljava/io/File;");
      v71 = (unk_6048BF50)(v3, v75, v9[114], "mZips", "[Ljava/util/zip/ZipFile;");
    }
    else
    {
      v78 = (unk_6048BF50)(v3, v75, v9[50], *v13, v9[52]);
      v84 = (unk_6048BF50)(v3, v75, v9[50], "mFiles", "[Ljava/io/File;");
      v82 = 0;
      v71 = (unk_6048BF50)(v3, v75, v9[50], "mZips", "[Ljava/util/zip/ZipFile;");
    }
    if ( v78 )
    {
      v14 = 0;
      v76 = (unk_6048B9CA)(v3, v78);
      v65 = 0;
      while ( 1 )
      {
        if ( v14 >= v76 )
        {
          v81 = 0;
          v86 = 0;
          goto LABEL_28;
        }
        v15 = (unk_6048B9D8)(v3, v78, v14);
        if ( v15 && (v65 = (unk_6048C880)(v3, v15, v9[59], v9[106])) != 0 )
        {
          if ( *v65 && !(unk_6049C0BC)() )
          {
            v81 = 0;
            v86 = 0;
            goto LABEL_28;
          }
        }
        else
        {
          android_P();
        }
        ++v14;
      }
    }
    goto LABEL_45;
  }
  v86 = (unk_6048BF50)(v3, v75, *(unk_604A3DA8 + 240), *(unk_604A3DA8 + 244), *(unk_604A3DA8 + 248));
  v81 = (unk_6048BF50)(v3, v86, v9[53], v9[54], v9[55]);
  v76 = (unk_6048B9CA)(v3, v81);
  v70 = 0;
  v65 = 0;
  while ( v70 < v76 )
  {
    v10 = (unk_6048B9D8)(v3, v81, v70);
    v67 = (unk_6048BF50)(v3, v10, v9[56], v9[57], v9[58]);
    if ( v67
      && ((v65 = (unk_6048C880)(v3, v67, v9[59], v9[106])) != 0 || (v65 = (unk_6048CA60)(v3, v67, v9[59], v9[106])) != 0) )
    {
      if ( *v65 && !(unk_6049C0BC)() )
      {
        v71 = 0;
        v84 = 0;
        v82 = 0;
        v78 = 0;
        goto LABEL_28;
      }
    }
    else
    {
      android_P();
    }
    ++v70;
  }
  v71 = 0;
  v84 = 0;
  v82 = 0;
  v78 = 0;
LABEL_28:
  if ( v72 )
    v16 = v72 - 40;
  else
    v16 = (unk_6048F564)(v89, "classes.dex", 0);
  v73 = 0;
  if ( !v16 )
  {
    (unk_6048F6A8)(&v96, v79, &v91);
    android_P();
    (unk_6048FC14)(&v96);
    android_P();
    v16 = (unk_6048F564)(v97, "classes.dex", 0);
    (unk_6048F664)(&v96);
    if ( v16 )
    {
      v73 = 0;
    }
    else
    {
      v16 = (unk_6048E964)(v65) - 40;
      v73 = 1;
    }
  }
  v17 = (unk_6048DA1A)(v16 + 40);
  android_P();
  (unk_6049C12C)(&v96, 0, 224);
  v88 = v16 + v17 + 40;
  (unk_6049C1AC)(&v96, v16 + v17 + 40, 224);
  (unk_604952DC)(&unk_604A0E41, &v96, 224, 32);
  v85 = v16 + v17 + 40;
  v68 = v98;
  v18 = android_P();
  if ( v73 )
  {
    v19 = v68;
    if ( v68 & 0xFFF )
      v19 = (v68 / 4096 + 1) << 12;
    v18 = (unk_6049C10C)(v16, v19, 3);
    if ( v18 )
    {
      v20 = v68;
      if ( v68 & 0xFFF )
        v20 = (v68 / 4096 + 1) << 12;
      v18 = (unk_6049C10C)(v16, v20, 5);
    }
  }
  v21 = (unk_6048DE24)(v18);
  v22 = *(unk_6049C30C)();
  android_P();
  if ( v21 == -1 )
  {
    v23 = android_P();
    if ( (unk_6048DE74)(v23) == -1 )
    {
      android_P();
      v24 = (unk_6049C25C)("/dev/zero", 2);
      v25 = (unk_6049C37C)(0, v68);
      (unk_6049C2BC)(v24);
      if ( !v25 )
      {
LABEL_45:
        android_P();
        goto LABEL_102;
      }
      (unk_6049C1BC)(v25, v88, v68);
      v85 = v25;
    }
  }
  (unk_604952DC)(&unk_604A0E41, v85, 224, 32);
  (unk_6048F6A8)(&v92, "/data/data/", &v91);
  (unk_6048F7DA)(&v92, v89);
  (unk_6048F994)(&v94, &v92, "/mix.so");
  (unk_6048F7DA)(&v92, "/mix.dex");
  v26 = (unk_6048BDB4)(v3, v93);
  v27 = (unk_6048BDB4)(v3, v95);
  if ( (unk_6048E4A8)(v3, v83, v93)
    && (v74 = (unk_6048BAD4)(
                v3,
                "dalvik/system/DexFile",
                "loadDex",
                "(Ljava/lang/String;Ljava/lang/String;I)Ldalvik/system/DexFile;",
                v26,
                v27,
                0)) != 0 )
  {
    v83 = v26;
    v80 = 0;
  }
  else
  {
    android_P();
    (unk_6048F8F8)(&v92, v79);
    v74 = (unk_6048BAD4)(
            v3,
            "dalvik/system/DexFile",
            "loadDex",
            "(Ljava/lang/String;Ljava/lang/String;I)Ldalvik/system/DexFile;",
            v83,
            0,
            0);
    android_P();
    v80 = 1;
  }
  v28 = (unk_6048C880)(v3, v74, v9[59], v9[106]);
  if ( !v28 )
  {
    v28 = (unk_6048CA60)(v3, v74, v9[59], v9[106]);
    if ( !v28 )
      android_P();
  }
  if ( v80 )
  {
    v29 = dword_604A46E8;
  }
  else
  {
    v29 = dword_604A46E8;
    if ( dword_604A46E8 > 10 )
    {
      v30 = *(*(v28 + 8) + 4);
      goto LABEL_61;
    }
  }
  v31 = *(v28 + 12);
  if ( v29 == 8 )
    v30 = *(v31 + 36);
  else
    v30 = *(v31 + 40);
LABEL_61:
  v91 = 0;
  (unk_6048E850)(v3, v85, v68, &v91, v30);
  v32 = v91;
  v33 = *(v91 + 4);
  if ( v80 )
  {
    *(v28 + 8) = v91;
    *(v28 + 4) = 1;
    if ( dword_604A46E8 == 10 )
      v65[4] = v85;
    goto LABEL_71;
  }
  v34 = dword_604A46E8;
  if ( dword_604A46E8 <= 10 )
  {
    *(v28 + 4) = 1;
    *(v28 + 8) = v32;
    if ( v34 == 10 )
      v65[4] = v85;
LABEL_71:
    *(v28 + 12) = 0;
    goto LABEL_72;
  }
  if ( dword_604A46E8 <= 18 && (unk_6048DEC8)() )
    (unk_60493180)(*(*(v28 + 8) + 4), v33, dword_604A46E8);
  else
    (unk_60493194)(*(*(v28 + 8) + 4), v33, dword_604A46E8);
LABEL_72:
  if ( dword_604A46E8 <= 10 )
  {
    v43 = (unk_6048B984)(v3, "dalvik/system/DexFile");
    v66 = (unk_6048D988)(v3, v76 + 1, v43, 0);
    (unk_6048B9E6)(v3, v66, 0, v74);
    for ( i = 0; i < v76; (unk_6048B9E6)(v3, v66, i, v45) )
      v45 = (unk_6048B9D8)(v3, v78, i++);
    if ( v82 )
    {
      v46 = (unk_6048B9CA)(v3, v82);
      v47 = (unk_6048B984)(v3, "java/lang/String");
      v87 = (unk_6048D988)(v3, v46 + 1, v47, 0);
      (unk_6048B9E6)(v3, v87, 0, v83);
      for ( j = 0; j < v46; (unk_6048B9E6)(v3, v87, j, v49) )
        v49 = (unk_6048B9D8)(v3, v82, j++);
    }
    v50 = (unk_6048B9CA)(v3, v84);
    v51 = (unk_6048B984)(v3, "java/io/File");
    v69 = (unk_6048D988)(v3, v50 + 1, v51, 0);
    v52 = (unk_6048B9B2)(v3, v51, "<init>", "(Ljava/lang/String;)V");
    v77 = (unk_6048B998)(v3, v51, v52, v83);
    (unk_6048B9E6)(v3, v69, 0, v77);
    for ( k = 0; k < v50; (unk_6048B9E6)(v3, v69, k, v54) )
      v54 = (unk_6048B9D8)(v3, v84, k++);
    v55 = (unk_6048B9CA)(v3, v71);
    v56 = (unk_6048B984)(v3, "java/util/zip/ZipFile");
    v57 = (unk_6048D988)(v3, v55 + 1, v56, 0);
    v58 = (unk_6048B9B2)(v3, v56, "<init>", "(Ljava/io/File;)V");
    v59 = (unk_6048B998)(v3, v56, v58, v77);
    (unk_6048B9E6)(v3, v57, 0, v59);
    for ( l = 0; l < v55; (unk_6048B9E6)(v3, v57, l, v61) )
      v61 = (unk_6048B9D8)(v3, v71, l++);
    v62 = unk_604A3DA8;
    v63 = (unk_6048B984)(v3, *(unk_604A3DA8 + 456));
    if ( (unk_6048D916)(v3, v75, v63) )
    {
      (unk_6048C00C)(v3, v75, v62[114], v62[51], v62[52], v66);
      (unk_6048C00C)(v3, v75, v62[114], "mPaths", "[Ljava/lang/String;", v87);
      (unk_6048C00C)(v3, v75, v62[114], "mFiles", "[Ljava/io/File;", v69);
      (unk_6048C00C)(v3, v75, v62[114], "mZips", "[Ljava/util/zip/ZipFile;", v57);
    }
    else
    {
      (unk_6048C00C)(v3, v75, v62[50], v62[51], v62[52], v66);
      (unk_6048C00C)(v3, v75, v62[50], "mFiles", "[Ljava/io/File;", v69);
      (unk_6048C00C)(v3, v75, v62[50], "mZips", "[Ljava/util/zip/ZipFile;", v57);
    }
    goto LABEL_85;
  }
  v35 = (unk_6048B984)(v3, "dalvik/system/DexPathList$Element");
  v36 = (unk_6048B9B2)(v3, v35, "<init>", "(Ljava/io/File;Ljava/util/zip/ZipFile;Ldalvik/system/DexFile;)V");
  if ( v36
    || ((unk_6048D90C)(v3),
        (v36 = (unk_6048B9B2)(v3, v35, "<init>", "(Ljava/io/File;Ljava/io/File;Ldalvik/system/DexFile;)V")) != 0) )
  {
    v37 = (unk_6048B998)(v3, v35, v36);
LABEL_79:
    v38 = v37;
    goto LABEL_81;
  }
  (unk_6048D90C)(v3);
  if ( (unk_6048B9B2)(v3, v35, "<init>", "(Ljava/io/File;ZLjava/io/File;Ldalvik/system/DexFile;)V") )
  {
    v37 = (unk_6048B998)(v3, v35);
    goto LABEL_79;
  }
  v38 = 0;
LABEL_81:
  v39 = (unk_6048B984)(v3, "dalvik/system/DexPathList$Element");
  v40 = (unk_6048D988)(v3, v76 + 1, v39, 0);
  (unk_6048B9E6)(v3, v40, 0, v38);
  for ( m = 0; m < v76; (unk_6048B9E6)(v3, v40, m, v42) )
    v42 = (unk_6048B9D8)(v3, v81, m++);
  (unk_6048C00C)(v3, v86, v9[53], v9[54], v9[55], v40);
LABEL_85:
  dword_604A4714 = (*(*v3 + 84))(v3, v74);
  android_P();
  (unk_6048F664)(&v94);
  (unk_6048F664)(&v92);
LABEL_102:
  result = v90;
  if ( v99 != *v90 )
    result = sub_6049C17C(v90);
  return result;
}
```

那么脱壳的话，此处就已经可以开始手动脱了。



那么整个逆向分析部分到这就结束了，后面的函数具体分析以及如何手动脱壳，咱们下次有机会再分享

## 0x03 结语

想说的是，这篇文章在操作上花了很重的笔墨，怎么说呢，要想详细介绍原理细节不是一句两句能说清楚的，比如为什么在这下断，还有dex加载，so加载等等细节，以及一些脱壳点，咱们下次有机会再分享。

## 0x04 参考文章

https://bbs.pediy.com/thread-218782.htm

https://bbs.pediy.com/thread-217556.htm