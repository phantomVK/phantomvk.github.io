---
layout:     post
title:      "Android源码系列(26) -- MultiDex"
date:       2019-11-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、介绍

#### 1.1 功能

经过长期需求迭代、引入大量第三方代码库之后，构建的安装包包含海量方法。即便经过代码混淆，依然会在不久的将来遇到 __Android64K方法数__ 问题。

官方对此这样解释：

> Android 应用 (APK) 文件包含 [Dalvik](https://source.android.com/devices/tech/dalvik/) Executable (DEX) 文件形式的可执行字节码文件，这些文件包含用来运行您的应用的已编译代码。Dalvik Executable 规范将可在单个 DEX 文件内引用的方法总数限制为 65,536，其中包括 Android 框架方法、库方法以及您自己的代码中的方法。在计算机科学领域内，术语[*千（简称 K）*](https://en.wikipedia.org/wiki/Kilo-)表示 1024（或 2^10）。由于 65,536 等于 64 X 1024，因此这一限制称为“64K 引用限制”。

既然单个Dex文件不能容纳应用所有方法引用，应运而生解决方案：把多余的方法引用放到第二个、第三个等后续Dex文件中。方法引用越多，最终分包数量越多。

#### 1.2 构建

添加以下配置后，代码构建时自动分包：

```groovy
android {
    defaultConfig {
        multiDexEnabled true
    }
}
```

__MultiDex__ 用于应用启动时加载被分割的子dex，让后续类加载能从子dex找到目标类。

```groovy
implementation "androidx.multidex:multidex:2.0.0"
```

#### 1.3 疑难

当然还可能会遇到：

- 启动类没有分到主包引起 __ClassNotFoundException__；
- 读取dex时间长导致 __ANR__ 提示；

这些问题网上很多文章都有提及，自行查找就有解决方案。更详细的说明可以参考官方文档：[为方法数超过 64K 的应用启用多 dex 文件](https://developer.android.com/studio/build/multidex?hl=zh-cn)。

为提高文章阅读性和不影响理解的前提，下文移除部分日志并微调代码格式，插图可以浏览器右键打开查看。

## 二、集成

最简单方式是继承 __MultiDexApplication__ 类。

如果不方便继承父类，可以选择在自定义的 __Application.attachBaseContext(Context base)__ 里主动调用 __MultiDex.install(this);__

```java
public class MultiDexApplication extends Application {
    public MultiDexApplication() {
    }

    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
}
```

## 三、提取

#### 3.1 IS_VM_MULTIDEX_CAPABLE

__Android4.4(API19)__ 获取 __System.getProperty("java.vm.version")__ 结果为 __1.6.0__。

```java
private static final boolean IS_VM_MULTIDEX_CAPABLE = 
  isVMMultidexCapable(System.getProperty("java.vm.version"));
```

#### 3.2 install()

上文提到的 __MultiDex.install(this);__ 调用以下方法：

```java
public static void install(Context context) {
    if (IS_VM_MULTIDEX_CAPABLE) {
        // 如果VM本身已经支持分包就不需要调用MultiDex，因为安装过程已完成相同操作
        Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
    } else if (VERSION.SDK_INT < 4) {
        // 最低支持API4
        throw new RuntimeException("MultiDex installation failed. SDK " + VERSION.SDK_INT
                                   + " is unsupported. Min SDK version is " + 4 + ".");
    } else {
        try {
            // 获取应用的信息
            ApplicationInfo applicationInfo = getApplicationInfo(context);
            if (applicationInfo == null) {
                return;
            }
          
            // 开始装载操作
            doInstallation(context, 
                           new File(applicationInfo.sourceDir),
                           new File(applicationInfo.dataDir),
                           "secondary-dexes", "", true);
        } catch (Exception var2) {
            throw new RuntimeException("MultiDex installation failed (" + var2.getMessage() + ").");
        }
    }
}
```

#### 3.3 doInstallation()

调用 __doInstallation()__ 开始装载操作：

```java
private static void doInstallation(Context mainContext, File sourceApk, File dataDir,
                                   String secondaryFolderName, String prefsKeyPrefix,
                                   boolean reinstallOnPatchRecoverableException)
  throws IOException, IllegalArgumentException, IllegalAccessException,
         NoSuchFieldException, InvocationTargetException, NoSuchMethodException,
         SecurityException, ClassNotFoundException, InstantiationException {

    // 加锁保证安装线程安全
    synchronized(installedApk) {
        if (!installedApk.contains(sourceApk)) {
            installedApk.add(sourceApk);

            // 类加载器
            ClassLoader loader;
            try {
                // 获取类加载器，提取的Dex后续通过反射添加到类加载器
                loader = mainContext.getClassLoader();
            } catch (RuntimeException var25) {
                // Failure while trying to obtain Context class loader.
                // Must be running in test mode. Skip patching.
                return;
            }

            if (loader == null) {
                // Context class loader is null.
                // Must be running in test mode. Skip patching.
            } else {
                try {
                    clearOldDexDir(mainContext);
                } catch (Throwable var24) {
                    // Something went wrong when trying to clear old MultiDex extraction
                    // continuing without cleaning.
                }

                // dataDir: /data/data/com.phantomvk.playground
                // secondaryFolderName: secondary-dexes
                // dexDir: /data/data/com.phantomvk.playground/code_cache/secondary-dexes
                File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);

                // sourceApk: /data/app/com.phantomvk.playground-1.apk
                MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);
                IOException closeException = null;

                try {
                    // 从MultiDexExtractor加载dexes获得zipList
                    List files = extractor.load(mainContext, prefsKeyPrefix, false);

                    try {
                        // 加载获取Dexes安装到ClassLoader
                        installSecondaryDexes(loader, dexDir, files);
                    } catch (IOException var26) {
                        if (!reinstallOnPatchRecoverableException) {
                            throw var26;
                        }

                        // 加载失败重试，从MultiDexExtractor加载dexes获得zips
                        files = extractor.load(mainContext, prefsKeyPrefix, true);

                        // 加载获取的Dexes安装到ClassLoader
                        installSecondaryDexes(loader, dexDir, files);
                    }
                } finally {
                    try {
                        extractor.close();
                    } catch (IOException var23) {
                        closeException = var23;
                    }
                }

                if (closeException != null) {
                    throw closeException;
                }
            }
        }
    }
}
```

变量 __dexDir__ 调试路径：

![dexDir](/img/android/multidex/dexDir.png)

进入 __MultiDexExtractor.load__ 提取器获取列表

```java
List<? extends File> load(Context context, String prefsKeyPrefix,
                          boolean forceReload) throws IOException {

    if (!this.cacheLock.isValid()) {
        throw new IllegalStateException("MultiDexExtractor was closed");
    } else {
        List files;
        // 如果不是'强制提取'和'文件已被修改'就尝试复用文件
        if (!forceReload && !isModified(context, this.sourceApk, this.sourceCrc, prefsKeyPrefix)) {
            try {
                // 直接复用已缓存的Dexes文件
                files = this.loadExistingExtractions(context, prefsKeyPrefix);
            } catch (IOException var6) {
                // 复用缓存出现错误，重新提取文件
                files = this.performExtractions();
                // 最后执行的数据会保存到SharedPreferences
                putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk),
                                 this.sourceCrc, files);
            }
        } else {
            // 更新App后sourceCrc改变会触发dex重新提取
            files = this.performExtractions();
            // 最后执行的数据会保存到SharedPreferences
            putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk),
                             this.sourceCrc,files);
        }

        return files;
    }
}
```

#### 3.4 MultiDexExtractor.performExtractions()

实际执行的提取操作方法：

```java
private List<MultiDexExtractor.ExtractedDex> performExtractions() throws IOException {
    // extractedFilePrefix: com.phantomvk.playground-1.apk.classes
    String extractedFilePrefix = this.sourceApk.getName() + ".classes";

    this.clearDexDir();
    List<MultiDexExtractor.ExtractedDex> files = new ArrayList();

    // APK本质是Zip，所以APK用Zip类型打开
    ZipFile apk = new ZipFile(this.sourceApk);

    try {
        // 名为classes.dex的主dex安装过程已完成提取
        // 运行时只从classes2.dex开始遍历，即secondaryNumber=2
        int secondaryNumber = 2;
        
        // apk作为zip文件打开后，可从zip文件列表中获取指定文件名dex
        // 从apk内逐个获取dex文件，把dex文件写为zip文件
        for(ZipEntry dexFile = apk.getEntry("classes" + secondaryNumber + ".dex");
            dexFile != null;
            dexFile = apk.getEntry("classes" + secondaryNumber + ".dex")) {
            
            // fileName: com.phantomvk.playground-1.apk.classes2.zip
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";

            // dexDir: /data/data/com.phantomvk.playground/code_cache/secondary-dexes/
            MultiDexExtractor.ExtractedDex extractedFile =
                new MultiDexExtractor.ExtractedDex(this.dexDir, fileName);
            files.add(extractedFile);

            // 每个dex提取失败累计重试最多3次
            int numAttempts = 0;
            boolean isExtractionSuccessful = false;

            while(numAttempts < 3 && !isExtractionSuccessful) {
                ++numAttempts;

                // 提取dexFile(如classes2.dex)为extractedFile
                extract(apk, dexFile, extractedFile, extractedFilePrefix);

                try {
                    // 获取提取后文件的CRC
                    extractedFile.crc = getZipCrc(extractedFile);
                    isExtractionSuccessful = true;
                } catch (IOException var18) {
                    isExtractionSuccessful = false;
                }

                // 提取失败则删除提取出来的文件
                if (!isExtractionSuccessful) {
                    extractedFile.delete();
                }
            }

            // 任意dex提取失败3次终止App启动流程
            if (!isExtractionSuccessful) {
                throw new IOException("Could not create zip file "
                    + extractedFile.getAbsolutePath()
                    + " for secondary dex (" + secondaryNumber + ")");
            }
            
            // 完成本次提取执行++secondaryNumber，继续遍历余下dex
            ++secondaryNumber;
        }
    } finally {
        try {
            apk.close();
        } catch (IOException var17) {
            Log.w("MultiDex", "Failed to close resource", var17);
        }

    }

    // 返回提取出来的zip列表
    return files;
}
```

提取完成后的 __files__ 结构如下，可见子dex总共有4个：

![filesAfter](/img/android/multidex/filesAfter.png)

#### 3.5 MultiDexExtractor.extract()

这里介绍如何从 __dex__ 转换为列表 __files__ 里保存的 __zip__：

```java
// @param apk 安装包源文件apkName.apk
// @param dexFile 安装包源文件解压获得的classes2.dex、classes3.dex等等
// @param extractTo 提取某个dex保存到对应Zip文件
// @param extractedFilePrefix 提取出来文件前缀
private static void extract(ZipFile apk, ZipEntry dexFile, File extractTo, String extractedFilePrefix) throws IOException, FileNotFoundException {
    // 为dex文件创建InputStream
    InputStream in = apk.getInputStream(dexFile);
    ZipOutputStream out = null;

    // tmpFileName:   tmp-com.phantomvk.playground-1.apk.classes-2082951698.zip
    // extractParent: /data/data/com.phantomvk.playground/code_cache/secondary-dexes/
    File tmp = File.createTempFile("tmp-" + extractedFilePrefix, ".zip", extractTo.getParentFile());

    try {
        // 包装temp文件创建ZipOutputStream
        out = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(tmp)));

        try {
            // 每个zip文件名不一样，但是zip里面的dex都命名为classes.dex
            ZipEntry classesDex = new ZipEntry("classes.dex");
            classesDex.setTime(dexFile.getTime());
            // 每个zip压缩包保存一个子文件dex
            out.putNextEntry(classesDex);
            byte[] buffer = new byte[16384]; // 16KB缓冲区

            // 从dex文件输入流in，写到zip文件输出流out
            for(int length = in.read(buffer); length != -1; length = in.read(buffer)) {
                out.write(buffer, 0, length);
            }

            out.closeEntry();
        } finally {
            out.close();
        }

        if (!tmp.setReadOnly()) {
            throw new IOException("Failed to mark readonly \"" + tmp.getAbsolutePath() + "\" (tmp of \"" + extractTo.getAbsolutePath() + "\")");
        }

        if (!tmp.renameTo(extractTo)) {
            throw new IOException("Failed to rename \"" + tmp.getAbsolutePath() + "\" to \"" + extractTo.getAbsolutePath() + "\"");
        }
    } finally {
        closeQuietly(in);
        tmp.delete();
    }
}
```

## 四、装载

上文只完成从安装包获得dexes，逐一提取 __zip__ 文件到磁盘的工作，提取完成的文件并还没有添加到 __ClassLoader__。

#### 4.1 installSecondaryDexes()

继续调用 __installSecondaryDexes__ 进行装载，根据不同版本进入不同分支：

```java
private static void installSecondaryDexes(ClassLoader loader,
                                          File dexDir,
                                          List<? extends File> files)
  throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException,
InvocationTargetException, NoSuchMethodException, IOException,
SecurityException, ClassNotFoundException, InstantiationException {

    // 提取zip文件列表不为空
    if (!files.isEmpty()) {
        if (VERSION.SDK_INT >= 19) {
            MultiDex.V19.install(loader, files, dexDir);
        } else if (VERSION.SDK_INT >= 14) {
            MultiDex.V14.install(loader, files);
        } else {
            MultiDex.V4.install(loader, files);
        }
    }
}
```

用 __Android4.4__ 作为示例进入 __V19__：

```java
static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries, File optimizedDirectory)
  throws IllegalArgumentException, IllegalAccessException,
         NoSuchFieldException, InvocationTargetException,
         NoSuchMethodException, IOException {

    // dalvik.system.DexPathList dalvik.system.BaseDexClassLoader.pathList
    Field pathListField = MultiDex.findField(loader, "pathList");
    Object dexPathList = pathListField.get(loader);

    ArrayList<IOException> suppressedExceptions = new ArrayList();

    // 1. makeDexElements()优化zip为Element对象；
    // 2. expandFieldArray()里把Element对象存入ClassLoader；
    MultiDex.expandFieldArray(dexPathList, "dexElements",
                              makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries),
                                              optimizedDirectory, suppressedExceptions));

    // 下面是异常情况的处理，可以忽略
    if (suppressedExceptions.size() > 0) {
        Iterator var6 = suppressedExceptions.iterator();

        while(var6.hasNext()) {
            IOException e = (IOException)var6.next();
            Log.w("MultiDex", "Exception in makeDexElement", e);
        }

        Field suppressedExceptionsField = MultiDex.findField(dexPathList, "dexElementsSuppressedExceptions");
        IOException[] dexElementsSuppressedExceptions = (IOException[])((IOException[])suppressedExceptionsField.get(dexPathList));
        if (dexElementsSuppressedExceptions == null) {
            dexElementsSuppressedExceptions = (IOException[])suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            IOException[] combined = new IOException[suppressedExceptions.size() + dexElementsSuppressedExceptions.length];
            suppressedExceptions.toArray(combined);
            System.arraycopy(dexElementsSuppressedExceptions, 0, combined,
                             suppressedExceptions.size(),
                             dexElementsSuppressedExceptions.length);
            dexElementsSuppressedExceptions = combined;
        }

        suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
        IOException exception = new IOException("I/O exception during makeDexElement");
        exception.initCause((Throwable)suppressedExceptions.get(0));
        throw exception;
    }
}
```

#### 4.2 makeDexElements()

__dexPathList__ 和 __optimizedDirectory__ 作为参数调用 __makeDexElements()__。

#### 4.2.1 dexPathList

__DexPathList__ 声明于[__BaseDexClassLoader__](http://androidxref.com/4.4.4_r1/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java#30)

```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    ....
}
```

__dexPathList__ 变量运行时内存结构：

![loader](/img/android/multidex/loader.png)

#### 4.2.2 optimizedDirectory

__optimizedDirectory__ 变量运行时内存结构：

![optimizedDirectory](/img/android/multidex/optimizedDirectory.png)

#### 4.2.3 调用makeDexElements

这个是 __V19.makeDexElements()__，反射调用 __DexPathList.makeDexElements()__

```java
private static Object[] makeDexElements(Object dexPathList,
                                        ArrayList<File> files,
                                        File optimizedDirectory,
                                        ArrayList<IOException> suppressedExceptions)
  throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {

    // 反射获得DexPathList.makeDexElements()
    Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements",
                                                 ArrayList.class, File.class, ArrayList.class);

    // 调用DexPathList.makeDexElements()把zip列表转换为Element[]
    return (Object[])((Object[])makeDexElements.invoke(dexPathList, files,
                                                       optimizedDirectory,
                                                       suppressedExceptions));
}
```

__makeDexElements(ArrayList, File, ArrayList)__ 位于类 [__DexPathList__](http://androidxref.com/4.4.4_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java#206)

```java
// Makes an array of dex/resource path elements, one per element of the given array.
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
                                         ArrayList<IOException> suppressedExceptions) {
    ArrayList<Element> elements = new ArrayList<Element>();

    // 遍历files里面的dex文件
    for (File file : files) {
        File zip = null;
        DexFile dex = null;
        String name = file.getName();

        if (name.endsWith(DEX_SUFFIX)) {
            // 原生dex文件且不是放在zip/jar里面
            try {
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException ex) {
                System.logE("Unable to load dex file: " + file, ex);
            }
        } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                || name.endsWith(ZIP_SUFFIX)) {
            // 因为是zip后缀，所以进入这个分支
            zip = file;

            try {
                // 调用loadDexFile()方法，看下文
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException suppressed) {
                 // IOException might get thrown "legitimately" by the DexFile constructor if the
                 // zip file turns out to be resource-only (that is, no classes.dex file in it).
                 // Let dex == null and hang on to the exception to add to the tea-leaves for
                 // when findClass returns null.
                suppressedExceptions.add(suppressed);
            }
        } else if (file.isDirectory()) {
            // We support directories for looking up resources.
            // This is only useful for running libcore tests.
            elements.add(new Element(file, true, null, null));
        } else {
            System.logW("Unknown file type for: " + file);
        }

        if ((zip != null) || (dex != null)) {
            // 封装DexFile为Element
            elements.add(new Element(file, false, zip, dex));
        }
    }

    // ArrayList to Array.
    return elements.toArray(new Element[elements.size()]);
}
```

把zip所在文件夹传到方法。zip文件封装为 __DexFile__ 过程会调用 [原生代码](http://androidxref.com/4.4.4_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexFile.java#294) 进行优化。

```java
// Constructs a {@code DexFile} instance, as appropriate depending
// on whether {@code optimizedDirectory} is {@code null}.
private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
  
    // 从上文可知optimizedDirectory非空
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        // 传入zip所在文件夹路径
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);

        // DexFile构建过程会优化源码
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```

#### 4.3 expandFieldArray()

 __original[]__ 存放已经加载的主dex，__extraElements[]__ 存放需要新增的子dex。

具体步骤：

- 通过反射 __dexElements__ 获取 __original[]__；
- 合成新数组 __original[]__ + __extraElements[]__ = __combined[]__；
- 用 __combined[]__ 覆盖 __dexElements[]__ 数据。

```java
// Replace the value of a field containing a non null array, by a new array containing the
// elements of the original array plus the elements of extraElements.
// @param instance the instance whose field is to be modified.
// @param fieldName the field to modify.
// @param extraElements elements to append at the end of the array.
private static void expandFieldArray(Object instance, String fieldName,
        Object[] extraElements) throws NoSuchFieldException, IllegalArgumentException,
        IllegalAccessException {
    // dexPathList获取makeDexElements数组相对地址
    Field jlrField = findField(instance, fieldName);

    // 从实例根据地址获取成员
    Object[] original = (Object[]) jlrField.get(instance);

    // 此数组将混合original已有的元素和新加入的提取数据
    Object[] combined = (Object[]) Array.newInstance(
            original.getClass().getComponentType(), original.length + extraElements.length);
    System.arraycopy(original, 0, combined, 0, original.length);
    System.arraycopy(extraElements, 0, combined, original.length, extraElements.length);

    // 用combined覆盖原original数组
    jlrField.set(instance, combined);
}
```

调用 __expandFieldArray()__ 前 __dexElements__ 只有主dex：

![beforeExpandFieldArray](/img/android/multidex/beforeExpandFieldArray.png)

调用 __expandFieldArray()__ 后新增4个子dex：

![beforeExpandFieldArray](/img/android/multidex/afterExpandFieldArray.png)

提取工作完成后内存布局：

![classload_done](/img/android/multidex/classload_done.png)

#### 4.4 putStoredApkInfo()

提取成功的dexes信息保存到 __SharedPreferences__ 以便下次对比决定是否重用。

```java
// 只能在获取文件锁LOCK_FILENAME之后才能调用
private static void putStoredApkInfo(Context context, String keyPrefix, long timeStamp,
        long crc, List<ExtractedDex> extractedDexes) {

    SharedPreferences prefs = getMultiDexPreferences(context);
    SharedPreferences.Editor edit = prefs.edit();

    edit.putLong(keyPrefix + KEY_TIME_STAMP, timeStamp); // 时间戳
    edit.putLong(keyPrefix + KEY_CRC, crc); // CRC检验码
    edit.putInt(keyPrefix + KEY_DEX_NUMBER, extractedDexes.size() + 1); // dex数量

    int extractedDexId = 2;
    for (ExtractedDex dex : extractedDexes) {
        // 记录每个子dex的CRC和最后修改时间
        edit.putLong(keyPrefix + KEY_DEX_CRC + extractedDexId, dex.crc);
        edit.putLong(keyPrefix + KEY_DEX_TIME + extractedDexId, dex.lastModified());
        extractedDexId++;
    }

    // 同步写入磁盘
    edit.commit();
}
```

## 五、总结

为规避首次启动解析dex时间过长的问题，个人有以下建议：

- 尽可能提高混淆强度，减少代码和方法使用量；
- 只使用一次的方法手动合并到调用点，相当于内联；
- 尽可能把代码存放在主dex，令提取工作在安装过程完成；
- 不常用功能使用动态加载，令安装时间和提取时间都减少；

虽然 __Android5.0__ 及后期系统使用 __ART__ 虚拟机，在安装过程会全部或部分优化dex，再也不会在应用启动时影响体验。但是减少代码量，即使只能降低安装时长也总归是好事。