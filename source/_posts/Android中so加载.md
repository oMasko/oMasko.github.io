---
title: Android中so加载
date: 2018-03-18 01:12:51
tags: Android

---



之前学习了ELF文件格式，然后就想分析分析Android是如何对so文件进行加载的，有助于理解linker机制以及so的加固

<!--more-->

## Java层调用

调用System.loadLibrary对so文件进行加载：

```
/**
     * Loads and links the library with the specified name. The mapping of the
     * specified library name to the full path for loading the library is
     * implementation-dependent.
     *
     * @param libName
     *            the name of the library to load.
     * @throws UnsatisfiedLinkError
     *             if the library could not be loaded.
     */
    public static void loadLibrary(String libName) {
        Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());
    }
```

在这个函数的上面还有一个load函数也是用来加载so文件的，区别在于load需要传入一个绝对路径，所以它可以从外部进行加载

```
/**
     * Loads and links the dynamic library that is identified through the
     * specified path. This method is similar to {@link #loadLibrary(String)},
     * but it accepts a full path specification whereas {@code loadLibrary} just
     * accepts the name of the library to load.
     *
     * @param pathName
     *            the path of the file to be loaded.
     */
    public static void load(String pathName) {
        Runtime.getRuntime().load(pathName, VMStack.getCallingClassLoader());
    }
```

先分析loadLibrary，进行跟进

{%asset_img 0.png%}

这里跟进findLibrary看看是怎样找到so路径的

{%asset_img 2.png%}

居然是返回空？？？是这样的，在Android里ClassLoader只是一个抽象类，部分的实现机制在子类BaseDexClassLoader中，那么跳过去看看

{%asset_img 3.png%}

此处呢，调用了DexPathList类的findlibrary方法，跟进查看

     /**
     * Finds the named native code library on any of the library
     * directories pointed at by this instance. This will find the
     * one in the earliest listed directory, ignoring any that are not
     * readable regular files.
     *
     * @return the complete path to the library or {@code null} if no
     * library was found
     */
    public String findLibrary(String libraryName) {
        String fileName = System.mapLibraryName(libraryName);
        for (File directory : nativeLibraryDirectories) {
            String path = new File(directory, fileName).getPath();
            if (IoUtils.canOpenReadOnly(path)) {
                return path;
            }
        }
        return null;
    }
呐，这里其实就是根据传进来的libName，返回内部SO库文件的完整路径filename。再回到Runtime类，获取filename后调用了doLoad方法

再来看看System.load方法

跟进Runtime类

呐，这里就比较直接了，直接调用了doLoad方法

{%asset_img 4.png%}，可见loadLibrary和load的后面的调用是一样的，前者虽然传入的只是库文件名字，但是后面调用了findLibrary对库文件路径进行了获取，load则直接传入库文件路径即可



那么，继续跟进doLoad，看看是如何加载的

{%asset_img 1.png%}

这里可以发现doLoad调用了native函数nativeLoad加载so文件



## Native层调用



跟进java_lang_runtime.cpp

{%asset_img 5.png%}

那么跟进dvmLoadNativeCode函数查看

{%asset_img 6.png%}

findSharedLibEntry会返回一个SharedLib对象pEntry，此处他会进行一个判断，pEntry对象里保留了so文件的加载信息，如果so以及加载过，上次用来加载的类加载器和当前使用的不一致，或者上次加载失败，都会返回false



接下来会调用dlopen来在当前进程中加载so文件，并且返回一个handle变量

{%asset_img 7.png%}

跟进dlopen函数

```
void* dlopen(const char* filename, int flags) {
  ScopedPthreadMutexLocker locker(&gDlMutex);
  soinfo* result = do_dlopen(filename, flags);
  if (result == NULL) {

__bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
return NULL;

  }
  return result;
}
```

继续跟进do_dlopen

```
823soinfo* do_dlopen(const char* name, int flags) {
824  if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL)) != 0) {
825    DL_ERR("invalid flags to dlopen: %x", flags);
826    return NULL;
827  }
828  set_soinfo_pool_protection(PROT_READ | PROT_WRITE);
829  soinfo* si = find_library(name);
830  if (si != NULL) {
831    si->CallConstructors();
832  }
833  set_soinfo_pool_protection(PROT_READ);
834  return si;
835}
```

find_library会判断so是否加载过，跟进看一下

```
static soinfo* find_library(const char* name) {
786  soinfo* si = find_library_internal(name); //判断是否加载过
787  if (si != NULL) {
788    si->ref_count++;
789  }
790  return si;
791}
```

{%asset_img 8.png%}

跟进load_library

{%asset_img 9.png%}

看一下load函数

```
bool ElfReader::Load() {
  return ReadElfHeader() &&

     VerifyElfHeader() &&
     ReadProgramHeader() &&
     ReserveAddressSpace() &&
     LoadSegments() &&
     FindPhdr();

}
```

一些校验啊，段映射之类的操作，这里就不跟进了

我们回到前面的do_dlopen，加载完后会调用一个CallConstructors()函数

```
void soinfo::CallConstructors() {
  if (constructors_called) {
    return;
  }

  // We set constructors_called before actually calling the constructors, otherwise it doesn't
  // protect against recursive constructor calls. One simple example of constructor recursion
  // is the libc debug malloc, which is implemented in libc_malloc_debug_leak.so:
  // 1. The program depends on libc, so libc's constructor is called here.
  // 2. The libc constructor calls dlopen() to load libc_malloc_debug_leak.so.
  // 3. dlopen() calls the constructors on the newly created
  //    soinfo for libc_malloc_debug_leak.so.
  // 4. The debug .so depends on libc, so CallConstructors is
  //    called again with the libc soinfo. If it doesn't trigger the early-
  //    out above, the libc constructor will be called again (recursively!).
  constructors_called = true;

  if ((flags & FLAG_EXE) == 0 && preinit_array != NULL) {
    // The GNU dynamic linker silently ignores these, but we warn the developer.
    PRINT("\"%s\": ignoring %d-entry DT_PREINIT_ARRAY in shared library!",
          name, preinit_array_count);
  }

  if (dynamic != NULL) {
    for (Elf32_Dyn* d = dynamic; d->d_tag != DT_NULL; ++d) {
      if (d->d_tag == DT_NEEDED) {
        const char* library_name = strtab + d->d_un.d_val;
        TRACE("\"%s\": calling constructors in DT_NEEDED \"%s\"", name, library_name);
        find_loaded_library(library_name)->CallConstructors();
      }
    }
  }

  TRACE("\"%s\": calling constructors", name);

  // DT_INIT should be called before DT_INIT_ARRAY if both are present.
  CallFunction("DT_INIT", init_func);
  CallArray("DT_INIT_ARRAY", init_array, init_array_count, false);
}
```

来看下注释，大致的意思就是会在这调用一些so的初始化函数，比如在这会调用init_func,init_array,init_array_count中的函数

（那么这里我们如果动态调试so时，我们就可以尝试将断点下在CallConstructors,来跟踪一些初始化函数，在脱so壳时很关建）



再回到这来

{%asset_img 10.png%}

此处调用了一个关键的JNI_Onload函数，这个函数可以动态注册一些native函数之类的操作

并且跟踪源码后发现.init段的初始化函数是早于JNI_Onload函数执行的





## 最后

加载过程大致如此，中间还有一些so怎样链接以及链接后填充soinfo结构体的细节没有去仔细分析，这些对于自己实现一套Linker机制还是很有帮助的