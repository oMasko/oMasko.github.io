---
title: Dalvik加载dex
date: 2018-03-17 19:30:24
tags: Android
---

## 0x00 前言

> 早期的一种Android加固手段是通过动态地加载dex文件来达到保护目的，当然这种办法早就已经过时，那么要了解这种加固手段就必须了解Android的动态加载机制

<!--more-->

## 0x01 DexClassLoader和PathClassLoader

Android中有许多类加载器，当然本文讨论范围主要是`DexClassLoader`和`PathClassLoader`这两个，这两个类都继承自`BaseDexCloader`这个父类，区别在于PathClassLoader这个类默认的优化路径参数为null，所以只能是内部存储路径，而DexClassLoader则可以选择内部存储路径或者外部存储路径

PathClassLoader:

```
public class PathClassLoader extends BaseDexClassLoader {
	public PathClassLoader(String dexPath, ClassLoader parent) {
		super(dexPath, null, null, parent);
	}
	public PathClassLoader(String dexPath, String libraryPath,
ClassLoader parent) {
		super(dexPath, null, libraryPath, parent);
	}
}
```

DexClassLoader:
​    

```
public class DexClassLoader extends BaseDexClassLoader {
	public DexClassLoader(String dexPath, String optimizedDirectory,
	String libraryPath, ClassLoader parent) {
	super(dexPath, new File(optimizedDirectory), libraryPath, parent);
	}
}
```

发现他们均是调用父类的构造函数，跟进查看：
​    

```
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
String libraryPath, ClassLoader parent) {
	super(parent);
	this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
}
```

第二个参数`optimizedDirectory`就是dex文件优化后的输出路径，可以看到`DexClassLoader`中使用了`new File()`函数来指定相应的路径，而`PathClassLoader`则是传入了`null`值，那么只能是默认的内部存储

这两个类差距就在这，其他的区别不大，都是调用父类的构造函数

## 0x02 Davlik加载Dex文件

> 应用程序在第一次启动app的时候，会在/dalvik/dalvik-cache目录下生成odex文件结构，其实就是在原app的dex文件结构基础上，增加odex文件头和并在原dex文件末尾增加依赖库信息和辅助信息，依赖库信息知名该dex文件所需要的本地函数库。
>
> 辅助信息记录了dex中类的索引表，dalvik为每一个dex文件创建一个对应DexClassLookup对象用于记dex文件所有的类索引信息，DexClassLookup对象为的每个类都生成了table结构体对象，该对象记录了类的描述符的哈希值、类描述符在dex文件中的偏移和类在dex文件中的偏移。其table结构体对象定义如下。也就是说通过该table结构体对象就可以对dex文件中某一个类快速定位。

{%asset_img 0.png %}

Davlik加载解析dex的过程中有几个重要的数据结构`DexFile`,`RawDexFile`,`DvmDex`和`ClassObject`,加载的dex解析之后就是以ClassObject的形式存储在内存中，然后通过解析具体的指令集去执行。存储指令集的数据结构就是Method结构体中的insns数组，这个了解过Dex文件结构的应该清楚。ClassObject结构体对象几乎包含类目标在运行时所需的所有资源，包括当前类的超类、当前类使用的类加载器、Method类型指针等。

{%asset_img 1.png %}

app在启动过程中创建了PathClassLoader去加载dex文件：
源码位置：[http://androidxref.com/4.4_r1/xref/frameworks/base/core/java/android/app/ApplicationLoaders.java](http://androidxref.com/4.4_r1/xref/frameworks/base/core/java/android/app/ApplicationLoaders.java)

```
class ApplicationLoaders
{
public static ApplicationLoaders getDefault()
{
    return gApplicationLoaders;
}

public ClassLoader getClassLoader(String zip, String libPath, ClassLoader parent)
{
    /*
     * This is the parent we use if they pass "null" in.  In theory
     * this should be the "system" class loader; in practice we
     * don't use that and can happily (and more efficiently) use the
     * bootstrap class loader.
     */
    ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();

    synchronized (mLoaders) {
        if (parent == null) {
            parent = baseParent;
        }

        /*
         * If we're one step up from the base class loader, find
         * something in our cache.  Otherwise, we create a whole
         * new ClassLoader for the zip archive.
         */
        if (parent == baseParent) {
            ClassLoader loader = mLoaders.get(zip);
            if (loader != null) {
                return loader;
            }

            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
            PathClassLoader pathClassloader =
                new PathClassLoader(zip, libPath, parent);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            mLoaders.put(zip, pathClassloader);
            return pathClassloader;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
        PathClassLoader pathClassloader = new PathClassLoader(zip, parent);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        return pathClassloader;
    }
}

private final ArrayMap<String, ClassLoader> mLoaders = new ArrayMap<String, ClassLoader>();

private static final ApplicationLoaders gApplicationLoaders
    = new ApplicationLoaders();
}
```

跟进PathClassLoader个构造函数：

```
public PathClassLoader(String dexPath, ClassLoader parent) {
    super(dexPath, null, null, parent);
}
```

跟进父类BaseDexClassLoader:

```
/**
 * Constructs an instance.
 *
 * @param dexPath the list of jar/apk files containing classes and
 * resources, delimited by {@code File.pathSeparator}, which
 * defaults to {@code ":"} on Android
 * @param optimizedDirectory directory where optimized dex files
 * should be written; may be {@code null}
 * @param libraryPath the list of directories containing native
 * libraries, delimited by {@code File.pathSeparator}; may be
 * {@code null}
 * @param parent the parent class loader
 */
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
}
```

这里有四个参数：

- dexPath--------------------------dex的路径
- optimizedDirectory ------------优化后的存储路径
- libraryPath-----------------------包含native方法的库文件路径
- parent----------------------------父加载器

调用了父类ClassLoader构造函数，这里重点关注这个DexPathList函数，跟进查看其构造函数：

```
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    if (definingContext == null) {
        throw new NullPointerException("definingContext == null");
    }

    if (dexPath == null) {
        throw new NullPointerException("dexPath == null");
    }

    if (optimizedDirectory != null) {
        if (!optimizedDirectory.exists())  {
            throw new IllegalArgumentException(
                    "optimizedDirectory doesn't exist: "
                    + optimizedDirectory);
        }

        if (!(optimizedDirectory.canRead()
                        && optimizedDirectory.canWrite())) {
            throw new IllegalArgumentException(
                    "optimizedDirectory not readable/writable: "
                    + optimizedDirectory);
        }
    }
    //上面各种验证
    this.definingContext = definingContext;
    ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                       suppressedExceptions);
    if (suppressedExceptions.size() > 0) {
        this.dexElementsSuppressedExceptions =
            suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
    } else {
        dexElementsSuppressedExceptions = null;
    }
    this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
}
```

前面巴拉巴拉的一大堆验证，先不管，跟进此处这个重要的函数`makeDexElements(splitDexPath(dexPath), optimizedDirectory,suppressedExceptions);`

这个splitDexPath函数作用是将传进来的dexPath路径进行字符串分割

```
/**
 * Splits the given path strings into file elements using the path
 * separator, combining the results and filtering out elements
 * that don't exist, aren't readable, or aren't either a regular
 * file or a directory (as specified). Either string may be empty
 * or {@code null}, in which case it is ignored. If both strings
 * are empty or {@code null}, or all elements get pruned out, then
 * this returns a zero-element list.
 */
private static ArrayList<File> splitPaths(String path1, String path2,
        boolean wantDirectories) {
    ArrayList<File> result = new ArrayList<File>();

    splitAndAdd(path1, wantDirectories, result);
    splitAndAdd(path2, wantDirectories, result);
    return result;
}

/**
 * Helper for {@link #splitPaths}, which does the actual splitting
 * and filtering and adding to a result.
 */
private static void splitAndAdd(String searchPath, boolean directoriesOnly,
        ArrayList<File> resultList) {
    if (searchPath == null) {
        return;
    }
    for (String path : searchPath.split(":")) {
        try {
            StructStat sb = Libcore.os.stat(path);
            if (!directoriesOnly || S_ISDIR(sb.st_mode)) {
                resultList.add(new File(path));
            }
        } catch (ErrnoException ignored) {
        }
    }
}
```

将分割后的字符串传递给`makeDexElements`函数

{%asset_img 2.png %}

发现无论是dex结尾的还是jar,apk结尾的都会去调用loadDexFile函数，然后返回一个Dexfile类型

`dex = loadDexFile(file, optimizedDirectory);`

那么跟进查看

```
/**
 * Constructs a {@code DexFile} instance, as appropriate depending
 * on whether {@code optimizedDirectory} is {@code null}.
 */
private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```

这里会先判断传递进来的optimizedDirectory是否为空，那么因为之前是由PathClassLoader传递进来的，那么这个变量肯定就是null了，那么我们就跟进DexFile的构造函数：

```
/**
 * Opens a DEX file from a given File object. This will usually be a ZIP/JAR
 * file with a "classes.dex" inside.
 *
 * The VM will generate the name of the corresponding file in
 * /data/dalvik-cache and open it, possibly creating or updating
 * it first if system permissions allow.  Don't pass in the name of
 * a file in /data/dalvik-cache, as the named file is expected to be
 * in its original (pre-dexopt) state.
 *
 * @param file
 *            the File object referencing the actual DEX file
 *
 * @throws IOException
 *             if an I/O error occurs, such as the file not being found or
 *             access rights missing for opening it
 */
public DexFile(File file) throws IOException {
    this(file.getPath());
}
```

注释写的很清楚，继续跟进：

```
/**
 * Opens a DEX file from a given filename. This will usually be a ZIP/JAR
 * file with a "classes.dex" inside.
 *
 * The VM will generate the name of the corresponding file in
 * /data/dalvik-cache and open it, possibly creating or updating
 * it first if system permissions allow.  Don't pass in the name of
 * a file in /data/dalvik-cache, as the named file is expected to be
 * in its original (pre-dexopt) state.
 *
 * @param fileName
 *            the filename of the DEX file
 *
 * @throws IOException
 *             if an I/O error occurs, such as the file not being found or
 *             access rights missing for opening it
 */
public DexFile(String fileName) throws IOException {
    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie);
}
```

这里调用了一个openDexFile()，其实不管你传进来的optimizedDirectory是否为空，最后都会调用到这个函数

调用完后会返回一个Cookie记录，那么我们跟进这个函数

```
/*
 * Open a DEX file.  The value returned is a magic VM cookie.  On
 * failure, an IOException is thrown.
 */
private static int openDexFile(String sourceName, String outputName,
    int flags) throws IOException {
    return openDexFileNative(new File(sourceName).getCanonicalPath(),
                             (outputName == null) ? null : new File(outputName).getCanonicalPath(),
                             flags);
}

native private static int openDexFileNative(String sourceName, String outputName,
    int flags) throws IOException;
```

发现最终会调用一个native函数，继续跟进

[http://androidxref.com/4.4_r1/xref/dalvik/vm/native/dalvik_system_DexFile.cpp](http://androidxref.com/4.4_r1/xref/dalvik/vm/native/dalvik_system_DexFile.cpp)

{%asset_img 3.png %}

下面有几个判断，一个一个来：

```
if (hasDexExtension(sourceName)
        && dvmRawDexFileOpen(sourceName, outputName, &pRawDexFile, false) == 0) {
    ALOGV("Opening DEX file '%s' (DEX)", sourceName);

    pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
    pDexOrJar->isDex = true;
    pDexOrJar->pRawDexFile = pRawDexFile;
    pDexOrJar->pDexMemory = NULL;
} 
```

`hasDexExtension`判断是否以dex结尾，如果是，继续`dvmRawDexFileOpen()`操作

```
else if (dvmJarFileOpen(sourceName, outputName, &pJarFile, false) == 0) {
    ALOGV("Opening DEX file '%s' (Jar)", sourceName);

    pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
    pDexOrJar->isDex = false;
    pDexOrJar->pJarFile = pJarFile;
    pDexOrJar->pDexMemory = NULL;
} 
```

apk或者jar结尾的就进行`dvmJarFileOpen`操作

那么这个native函数主要也就是进行这两个操作，其实两个函数操作差不多，只是第二个多了一个提取，将dex文件提取出来。

跟进第一个函数，有点长，一段一段来：

{%asset_img 4.png %}

然后往下跟进：

```
 /*
 * See if the cached file matches. If so, optFd will become a reference
 * to the cached file and will have been seeked to just past the "opt"
 * header.
 */

if (odexOutputName == NULL) {
    cachedName = dexOptGenerateCacheFileName(fileName, NULL);
    if (cachedName == NULL)
        goto bail;
} else {
    cachedName = strdup(odexOutputName);
}
```

这里cachedName是就是新建的一个缓存，那么这里只要先在这个缓存操作，然后再把缓存拷贝到目标文件即可，注意这个strdup函数，这是个拷贝函数内部会调用malloc创建内存，后面有free函数跟他相匹配出现

继续跟进

```
optFd = dvmOpenCachedDexFile(fileName, cachedName, modTime,
adler32, isBootstrap, &newFile, /*createIfMissing=*/true);
```

这个函数的作用就是给dex文件写入一个opt头，如果写成功那么newFile的值变为true

具体就不跟进，主要操作如下：

{%asset_img 5.png %}

如果写入成功：

{%asset_img 6.png %}

再往下：

```
if (dvmDexFileOpenFromFd(optFd, &pDvmDex) != 0) {
    ALOGI("Unable to map cached %s", fileName);
    goto bail;
}
```

`dvmDexFileOpenFromFd`这个函数最主要就是，将优化后得dex文件（也就是odex文件）通过`mmap`映射到内存中，并通过`mprotect`修改它的映射内存为只读权限，然后将映射为只读的这块dex数据中的内容全部提取到`DexFile`这个数据结构中去

然后最后成功的话就返回给RawDexFile

然后回到这：

{%asset_img 7.png %}

那么dvmJarFileOpen函数的操作大同小异，就是多了一个提取的操作

最后将这个返回给之前的Cookie

```
RETURN_PTR(pDexOrJar);
```

看一下宏定义：

```
#define RETURN_PTR(_val)do { pResult->l = (Object*)(_val); return; } while(0)
```

那么所谓cookie其实是个DexOrJar类型结构体的指针

那么整个加载的流程差不多就是这样了。。。。。。



## 0x03 运行时解析

类加载的目的其实就是为所需的类生成一个ClassObject的实例，然后需要的时候调用这个实例对象

看一下ClassObject的定义，里面有一些描述变量以及一些辅助的描述信息

```
/*
 * Class objects have many additional fields.  This is used for both
 * classes and interfaces, including synthesized classes (arrays and
 * primitive types).
 *
 * Class objects are unusual in that they have some fields allocated with
 * the system malloc (or LinearAlloc), rather than on the GC heap.  This is
 * handy during initialization, but does require special handling when
 * discarding java.lang.Class objects.
 *
 * The separation of methods (direct vs. virtual) and fields (class vs.
 * instance) used in Dalvik works out pretty well.  The only time it's
 * annoying is when enumerating or searching for things with reflection.
 */
struct ClassObject : Object {
    /* leave space for instance data; we could access fields directly if we
       freeze the definition of java/lang/Class */
    u4              instanceData[CLASS_FIELD_SLOTS];

    /* UTF-8 descriptor for the class; from constant pool, or on heap
       if generated ("[C") */
    const char*     descriptor;
    char*           descriptorAlloc;

    /* access flags; low 16 bits are defined by VM spec */
    u4              accessFlags;

    /* VM-unique class serial number, nonzero, set very early */
    u4              serialNumber;

    /* DexFile from which we came; needed to resolve constant pool entries */
    /* (will be NULL for VM-generated, e.g. arrays and primitive classes) */
    DvmDex*         pDvmDex;

    /* state of class initialization */
    ClassStatus     status;

    /* if class verify fails, we must return same error on subsequent tries */
    ClassObject*    verifyErrorClass;

    /* threadId, used to check for recursive <clinit> invocation */
    u4              initThreadId;

    /*
     * Total object size; used when allocating storage on gc heap.  (For
     * interfaces and abstract classes this will be zero.)
     */
    size_t          objectSize;

    /* arrays only: class object for base element, for instanceof/checkcast
       (for String[][][], this will be String) */
    ClassObject*    elementClass;

    /* arrays only: number of dimensions, e.g. int[][] is 2 */
    int             arrayDim;

    /* primitive type index, or PRIM_NOT (-1); set for generated prim classes */
    PrimitiveType   primitiveType;

    /* superclass, or NULL if this is java.lang.Object */
    ClassObject*    super;

    /* defining class loader, or NULL for the "bootstrap" system loader */
    Object*         classLoader;

    /* initiating class loader list */
    /* NOTE: for classes with low serialNumber, these are unused, and the
       values are kept in a table in gDvm. */
    InitiatingLoaderList initiatingLoaderList;

    /* array of interfaces this class implements directly */
    int             interfaceCount;
    ClassObject**   interfaces;

    /* static, private, and <init> methods */
    int             directMethodCount;
    Method*         directMethods;

    /* virtual methods defined in this class; invoked through vtable */
    int             virtualMethodCount;
    Method*         virtualMethods;

    /*
     * Virtual method table (vtable), for use by "invoke-virtual".  The
     * vtable from the superclass is copied in, and virtual methods from
     * our class either replace those from the super or are appended.
     */
    int             vtableCount;
    Method**        vtable;

    /*
     * Interface table (iftable), one entry per interface supported by
     * this class.  That means one entry for each interface we support
     * directly, indirectly via superclass, or indirectly via
     * superinterface.  This will be null if neither we nor our superclass
     * implement any interfaces.
     *
     * Why we need this: given "class Foo implements Face", declare
     * "Face faceObj = new Foo()".  Invoke faceObj.blah(), where "blah" is
     * part of the Face interface.  We can't easily use a single vtable.
     *
     * For every interface a concrete class implements, we create a list of
     * virtualMethod indices for the methods in the interface.
     */
    int             iftableCount;
    InterfaceEntry* iftable;

    /*
     * The interface vtable indices for iftable get stored here.  By placing
     * them all in a single pool for each class that implements interfaces,
     * we decrease the number of allocations.
     */
    int             ifviPoolCount;
    int*            ifviPool;

    /* instance fields
     *
     * These describe the layout of the contents of a DataObject-compatible
     * Object.  Note that only the fields directly defined by this class
     * are listed in ifields;  fields defined by a superclass are listed
     * in the superclass's ClassObject.ifields.
     *
     * All instance fields that refer to objects are guaranteed to be
     * at the beginning of the field list.  ifieldRefCount specifies
     * the number of reference fields.
     */
    int             ifieldCount;
    int             ifieldRefCount; // number of fields that are object refs
    InstField*      ifields;

    /* bitmap of offsets of ifields */
    u4 refOffsets;

    /* source file name, if known */
    const char*     sourceFile;

    /* static fields */
    int             sfieldCount;
    StaticField     sfields[0]; /* MUST be last item */
};
```



大致分析一下流程

回到BaseDexClassLoader.java

这里的findclass方法就是对指定的类名开始进行搜索加载，跟进查看，这里调用了DexPathList类的findclass方法



    /**
     * Finds the named class in one of the dex files pointed at by
     * this instance. This will find the one in the earliest listed
     * path element. If the class is found but has not yet been
     * defined, then this method will define it in the defining
     * context that this instance was constructed with.
     *
     * @param name of class to find
     * @param suppressed exceptions encountered whilst finding the class
     * @return the named class or {@code null} if the class is not
     * found in any of the dex files
     */
    public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
    
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
这里的dexElements，来看下定义 ，那么这里就是遍历一下之前所有的DexFile实例，其实也就是遍历了所有加载过的dex文件

     /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;
然后调用了loadClassBinaryName方法，跟过去看下

     /**
     * See {@link #loadClass(String, ClassLoader)}.
     *
     * This takes a "binary" class name to better match ClassLoader semantics.
     *
     * @hide
     */
    public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, suppressed);
    }
调用了defineClass，跟进

        private static Class defineClass(String name, ClassLoader loader, int cookie, List<Throwable> suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie);
        } catch (NoClassDefFoundError e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        } catch (ClassNotFoundException e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        }
        return result;
    }
此处调用了一个native层的方法defineClassNative

```
private static native Class defineClassNative(String name, ClassLoader loader, int cookie)
        throws ClassNotFoundException, NoClassDefFoundError;
```



有点长，一段段来分析

```
static void Dalvik_dalvik_system_DexFile_defineClassNative(const u4* args,
    JValue* pResult)
{
    StringObject* nameObj = (StringObject*) args[0];
    Object* loader = (Object*) args[1];
    int cookie = args[2];
    ClassObject* clazz = NULL;
    DexOrJar* pDexOrJar = (DexOrJar*) cookie;
    DvmDex* pDvmDex;
    char* name;
    char* descriptor;

    name = dvmCreateCstrFromString(nameObj); //类型转换
    descriptor = dvmDotToDescriptor(name);//得到描述符
    ALOGV("--- Explicit class load '%s' l=%p c=0x%08x",
        descriptor, loader, cookie);
    free(name);

    if (!validateCookie(cookie))//校验是不是有效的cookie
        RETURN_VOID();

    if (pDexOrJar->isDex)//判断是dex或者jar
        pDvmDex = dvmGetRawDexFileDex(pDexOrJar->pRawDexFile);//这里拿到DvmDex结构体
    else
        pDvmDex = dvmGetJarFileDex(pDexOrJar->pJarFile);//同上

    /* once we load something, we can't unmap the storage */
    pDexOrJar->okayToFree = false;

    clazz = dvmDefineClass(pDvmDex, descriptor, loader);
    Thread* self = dvmThreadSelf();
    if (dvmCheckException(self)) {
        /*
         * If we threw a "class not found" exception, stifle it, since the
         * contract in the higher method says we simply return null if
         * the class is not found.
         */
        Object* excep = dvmGetException(self);
        if (strcmp(excep->clazz->descriptor,
                   "Ljava/lang/ClassNotFoundException;") == 0 ||
            strcmp(excep->clazz->descriptor,
                   "Ljava/lang/NoClassDefFoundError;") == 0)
        {
            dvmClearException(self);
        }
        clazz = NULL;
    }

    free(descriptor);
    RETURN_PTR(clazz);
}
```

跟进一下dvmDefineClass

```
ClassObject* dvmDefineClass(DvmDex* pDvmDex, const char* descriptor,
    Object* classLoader)
{
    assert(pDvmDex != NULL);

    return findClassNoInit(descriptor, classLoader, pDvmDex);
}
```



这里返回调用了findClassNoInit方法，跟过去看一下（此处参考文章http://pwn4.fun/2016/04/11/Dalvik%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%A8%A1%E5%9D%97/）

这函数有点长0.0，就取关键部分分析

```
static ClassObject* findClassNoInit(const char* descriptor, Object* loader,
    DvmDex* pDvmDex)
{
    Thread* self = dvmThreadSelf();
    ClassObject* clazz; // 类加载的最终形式
    bool profilerNotified = false;
    /* 判断目标类是否有类加载器，对于系统类，虚拟机将从默认的启动路径实现其加载工作
    对于用户类，虚拟机一般情况下使用默认的类加载器实现类加载工作 */
    if (loader != NULL) {
        LOGVV("#### findClassNoInit(%s,%p,%p)", descriptor, loader,
            pDvmDex->pDexFile);
    }
    ...
        /* 根据目标类的描述符从hash表(系统已加载类)里查找是否已经有该Class的信息，如果已经加载，则返回
    其ClassObject对象，否则，对目标类进行加载*/
    clazz = dvmLookupClass(descriptor, loader, true);
    if (clazz == NULL) {
        const DexClassDef* pClassDef;

        dvmMethodTraceClassPrepBegin();
        profilerNotified = true;

#if LOG_CLASS_LOADING
        u8 startTime = dvmGetThreadCpuTimeNsec();
#endif
        /* 判断是否存在DvmDex结构体对象，如果存在，则表示目标类为一个用户类，将从一个解析的Dex文件
        中进行加载，对于一个解析过的Dex文件，是一定存在一个DvmDex结构体对象的，故pDvmDex一定不为空
        若为空，则表示目标类是一个系统类，虚拟机将调用searchBootPathForClass函数从启动路径下查找并
        加载目标类 */
        if (pDvmDex == NULL) {
            assert(loader == NULL);     /* shouldn't be here otherwise */
            /* 从BOOTCLASSPATH里那一堆jar包文件中，看看哪个jar包声明了目标类返回的是一个打开了的代
            表odex文件的DvmDex对象 */
            pDvmDex = searchBootPathForClass(descriptor, &pClassDef);
        } else {
            // 查找目标类的类定义资源
            pClassDef = dexFindClass(pDvmDex->pDexFile, descriptor);
        }
	...
        /* 当获得了加载目标类所需的各项资源，主函数将调用loadClassFromDex函数对目标类进行加载 */
        clazz = loadClassFromDex(pDvmDex, pClassDef, loader);
        if (dvmCheckException(self)) {
            /* class was found but had issues */
            if (clazz != NULL) {
                dvmFreeClassInnards(clazz);
                dvmReleaseTrackedAlloc((Object*) clazz, NULL);
            }
            goto bail;
        }
        /* 将目前使用的类锁住，防止其他进程更改 */
        dvmLockObject(self, (Object*) clazz);
        clazz->initThreadId = self->threadId;
        
        assert(clazz->classLoader == loader);
        if (!dvmAddClassToHash(clazz)) { // class对象加到Hash表里
	        clazz->initThreadId = 0;
            dvmUnlockObject(self, (Object*) clazz);

            /* Let the GC free the class. */
            dvmFreeClassInnards(clazz);
            dvmReleaseTrackedAlloc((Object*) clazz, NULL);
            /* 从已加载的类的系统Hash表中重新得到类 */
            clazz = dvmLookupClass(descriptor, loader, true);
            assert(clazz != NULL);
            goto got_class;
        }
        ...
                /*
         * 准备开始连接类，对clazz进行一些处理：
         * 1.解析ClassObject对象的基类信息，和它实现了那些接口
         * 2.校验：比如父类是final的，那么就不应该有它的派生类等
         * 此函数调用成功后，clazz的状态将是CLASS_RESOLVED或CLASS_VERIFIED
         */
        if (!dvmLinkClass(clazz)) {
            assert(dvmCheckException(self));
            /* Make note of the error and clean up the class. */
            removeClassFromHash(clazz);
            clazz->status = CLASS_ERROR;
            dvmFreeClassInnards(clazz);
            /* Let any waiters know. */
            clazz->initThreadId = 0;
            dvmObjectNotifyAll(self, (Object*) clazz);
            dvmUnlockObject(self, (Object*) clazz);
            ...
        }
        /* 将类的状态增加到全局变量中去 */
        gDvm.numLoadedClasses++;
        gDvm.numDeclaredMethods +=
            clazz->virtualMethodCount + clazz->directMethodCount;
        gDvm.numDeclaredInstFields += clazz->ifieldCount;
        gDvm.numDeclaredStaticFields += clazz->sfieldCount;
        ...
    /* check some invariants */
    assert(dvmIsClassLinked(clazz));
    assert(gDvm.classJavaLangClass != NULL);
    assert(clazz->clazz == gDvm.classJavaLangClass);
    assert(dvmIsClassObject(clazz));
    assert(clazz == gDvm.classJavaLangObject || clazz->super != NULL);
    if (!dvmIsInterfaceClass(clazz)) {
        //ALOGI("class=%s vtableCount=%d, virtualMeth=%d",
        //    clazz->descriptor, clazz->vtableCount,
        //    clazz->virtualMethodCount);
        assert(clazz->vtableCount >= clazz->virtualMethodCount);
    }

bail:
    if (profilerNotified)
        dvmMethodTraceClassPrepEnd();
    assert(clazz != NULL || dvmCheckException(self));
    return clazz;
}
```

调用dvmLookupClass函数判断本类是否已经被加载，

如果已经加载：直接返回，

如果没加载未加载：判断能否找到Dex文件，能找到调用`dexFindClass`在指定Dex文件中根据类的描述符查找相关类（用户类）；找不到调用`searchBootPathForClass`从系统启动基本路径中查找并加载目标类（系统类）。调用loadClassFromDex函数实现加载类达到可运行状态。调用dvmAddClassToHash实现将新加载的类添加到哈希表中方便在此查找。



loadClassFromDex 函数将调用`loadClassFromDex0`函数完成对该类的加载工作，返回值为一个`ClassObject`结构体对象。

{%asset_img 9.png%}



```
static ClassObject* loadClassFromDex0(DvmDex* pDvmDex,
    const DexClassDef* pClassDef, const DexClassDataHeader* pHeader,
    const u1* pEncodedData, Object* classLoader)
{
    ClassObject* newClass = NULL; // 目标类的类实例对象
    const DexFile* pDexFile;      // 用于存储目标Dex文件所对应的DexFile数据结构实例对象
    const char* descriptor;       // 用于存储目标类的描述符
    int i;
    // 获取相应的类信息
    pDexFile = pDvmDex->pDexFile;
    descriptor = dexGetClassDescriptor(pDexFile, pClassDef);
    ...
    // 为即将生成的类对象实例申请内存空间
    assert(descriptor != NULL);
    // 判断是不是java.lang.Class类，此类已经初始化过了
    if (classLoader == NULL &&
        strcmp(descriptor, "Ljava/lang/Class;") == 0) {
        assert(gDvm.classJavaLangClass != NULL);
        newClass = gDvm.classJavaLangClass;
    } else {
        // 取得对象实例大小并在内存中申请相应内存
        size_t size = classObjectSize(pHeader->staticFieldsSize);
        newClass = (ClassObject*) dvmMalloc(size, ALLOC_NON_MOVING);
    }
    if (newClass == NULL)
        return NULL;

    // 对新的类对象实例进行初始化
    DVM_OBJECT_INIT(newClass, gDvm.classJavaLangClass);
    dvmSetClassSerialNumber(newClass);
    newClass->descriptor = descriptor;
    assert(newClass->descriptorAlloc == NULL);
    SET_CLASS_FLAG(newClass, pClassDef->accessFlags);
    // 设定字段对象
    dvmSetFieldObject((Object *)newClass,
                      OFFSETOF_MEMBER(ClassObject, classLoader),
                      (Object *)classLoader);
    // 设定类的相关指针
    newClass->pDvmDex = pDvmDex;
    newClass->primitiveType = PRIM_NOT;
    newClass->status = CLASS_IDX;
    // 将这个类的父类的索引加入到类对象的指针区域
    assert(sizeof(u4) == sizeof(ClassObject*)); /* 32-bit check */
    newClass->super = (ClassObject*) pClassDef->superclassIdx;

    const DexTypeList* pInterfacesList;
    // 得到接口列表
    pInterfacesList = dexGetInterfacesList(pDexFile, pClassDef);
    if (pInterfacesList != NULL) {
        newClass->interfaceCount = pInterfacesList->size;
        newClass->interfaces = (ClassObject**) dvmLinearAlloc(classLoader,
                newClass->interfaceCount * sizeof(ClassObject*));
        // newClass实现了哪些接口类，此处也先以接口类的index存储，后续放到dvmLinkClass来解析
        for (i = 0; i < newClass->interfaceCount; i++) {
            const DexTypeItem* pType = dexGetTypeItem(pInterfacesList, i);
            newClass->interfaces[i] = (ClassObject*)(u4) pType->typeIdx;
        }
        dvmLinearReadOnly(classLoader, newClass->interfaces);
    }
    // 对字段进行加载，首先加载静态字段
    if (pHeader->staticFieldsSize != 0) {
        /* static fields stay on system heap; field data isn't "write once" */
        int count = (int) pHeader->staticFieldsSize;
        u4 lastIndex = 0;
        DexField field;
        // 取得字段数
        newClass->sfieldCount = count;
        // 逐一加载字段
        for (i = 0; i < count; i++) {
            dexReadClassDataField(&pEncodedData, &field, &lastIndex);
            // 解析newClass定义的静态成员信息
            loadSFieldFromDex(newClass, &field, &newClass->sfields[i]);
        }
    }
    // 加载实例字段
    if (pHeader->instanceFieldsSize != 0) {
        int count = (int) pHeader->instanceFieldsSize;
        u4 lastIndex = 0;
        DexField field;
        // 取得字段数
        newClass->ifieldCount = count;
        newClass->ifields = (InstField*) dvmLinearAlloc(classLoader,
                count * sizeof(InstField));
        // 逐一加载字段
        for (i = 0; i < count; i++) {
            dexReadClassDataField(&pEncodedData, &field, &lastIndex);
            loadIFieldFromDex(newClass, &field, &newClass->ifields[i]);
        }
        dvmLinearReadOnly(classLoader, newClass->ifields);
    }
    // 对类方法进行加载
    if (pHeader->directMethodsSize != 0) {
        int count = (int) pHeader->directMethodsSize;
        u4 lastIndex = 0;
        DexMethod method;
        // 取得方法数目
        newClass->directMethodCount = count;
        newClass->directMethods = (Method*) dvmLinearAlloc(classLoader,
                count * sizeof(Method));
        // 逐一加载方法
        for (i = 0; i < count; i++) {
            dexReadClassDataMethod(&pEncodedData, &method, &lastIndex);
            loadMethodFromDex(newClass, &method, &newClass->directMethods[i]);
            if (classMapData != NULL) {
                const RegisterMap* pMap = dvmRegisterMapGetNext(&classMapData);
                if (dvmRegisterMapGetFormat(pMap) != kRegMapFormatNone) {
                    newClass->directMethods[i].registerMap = pMap;
                    /* TODO: add rigorous checks */
                    assert((newClass->directMethods[i].registersSize+7) / 8 ==
                        newClass->directMethods[i].registerMap->regWidth);
                }
            }
        }
        dvmLinearReadOnly(classLoader, newClass->directMethods);
    }
    // 加载虚方法
    if (pHeader->virtualMethodsSize != 0) {
        int count = (int) pHeader->virtualMethodsSize;
        u4 lastIndex = 0;
        DexMethod method;
        // 取得虚方法数目
        newClass->virtualMethodCount = count;
        newClass->virtualMethods = (Method*) dvmLinearAlloc(classLoader,
                count * sizeof(Method));
        // 逐一处理方法
        for (i = 0; i < count; i++) {
            dexReadClassDataMethod(&pEncodedData, &method, &lastIndex);
            loadMethodFromDex(newClass, &method, &newClass->virtualMethods[i]);
            if (classMapData != NULL) {
                const RegisterMap* pMap = dvmRegisterMapGetNext(&classMapData);
                if (dvmRegisterMapGetFormat(pMap) != kRegMapFormatNone) {
                    newClass->virtualMethods[i].registerMap = pMap;
                    /* TODO: add rigorous checks */
                    assert((newClass->virtualMethods[i].registersSize+7) / 8 ==
                        newClass->virtualMethods[i].registerMap->regWidth);
                }
            }
        }
        dvmLinearReadOnly(classLoader, newClass->virtualMethods);
    }
    // 保存源文件信息
    newClass->sourceFile = dexGetSourceFile(pDexFile, pClassDef);

    /* caller must call dvmReleaseTrackedAlloc */
    return newClass; // 返回类对象
}
```

依次完成：

1.在内存中为类对象申请存储空间；

2.设置字段信息；

3.为超类建立索引；

4.加载类接口；

5.加载类字段；

6.加载类方法，并将以上数据封装成一个ClassObject结构体对象并返回。



## 0x04 最后

每次分析源码都能看到一些新的东西，常看常新，上面的知识点对于脱早期壳非常重要

比如熟悉Dalvik加载dex的机制，那么就能知道可以在何处下断，然后在内存中dump下dex文件

熟悉类的运行时解析机制，就可以对付某些讲类指令抽空的壳





