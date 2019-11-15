---
layout:     post
title:      "MultiDex源码解析"
date:       2019-10-22
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 一、初见MultiDex

经过长期需求开发、引入多个第三方依赖库后，构建的APK包含大量方法。即使经过代码混淆里的优化过程，依然会在不就的将来遇到 __Android64K方法数__ 问题。官方对此是这样解释的：

> Android 应用 (APK) 文件包含 [Dalvik](https://source.android.com/devices/tech/dalvik/) Executable (DEX) 文件形式的可执行字节码文件，这些文件包含用来运行您的应用的已编译代码。Dalvik Executable 规范将可在单个 DEX 文件内引用的方法总数限制为 65,536，其中包括 Android 框架方法、库方法以及您自己的代码中的方法。在计算机科学领域内，术语[*千（简称 K）*](https://en.wikipedia.org/wiki/Kilo-)表示 1024（或 2^10）。由于 65,536 等于 64 X 1024，因此这一限制称为“64K 引用限制”。

既然一个Dex文件不能容纳所有方法引用，应运而生解决方案：把多余的方法引用放到第二个、第三个等后续Dex文件中。方法引用越多，最终分包数量越多。

以下配置让代码打包时自动classes分包：

```
android {
    compileSdkVersion 29
    buildToolsVersion "28.0.3"
    defaultConfig {
        .....
        multiDexEnabled true // 开启分包
    }
}
```

而 __MultiDex__ 则是应用启动时读取被分割的包，令后续类加载时能从非主包找到了目标类。当然还可能会遇到：启动类没有分到主包引起 __ClassNotFoundException__、读取Dex时间过长导致 __ANR__ 提示等等问题。这些问题网上很多文章提及，自行查找就有解决方案。

更详细的说明可以参考官方文档：[为方法数超过 64K 的应用启用多 dex 文件](https://developer.android.com/studio/build/multidex?hl=zh-cn)

## 二、用法

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

## MultiDex

在 __Android4.4(API19)__ 获取 __System.getProperty("java.vm.version")__ 的结果为 __1.6.0__

```java
private static final boolean IS_VM_MULTIDEX_CAPABLE = isVMMultidexCapable(System.getProperty("java.vm.version"));
```

通过 __ApplicationContext__ 调用以下方法

```java
public static void install(Context context) {
    Log.i("MultiDex", "Installing application");
    if (IS_VM_MULTIDEX_CAPABLE) {
        Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
    } else if (VERSION.SDK_INT < 4) {
        throw new RuntimeException("MultiDex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
    } else {
        try {
            ApplicationInfo applicationInfo = getApplicationInfo(context);
            if (applicationInfo == null) {
                Log.i("MultiDex", "No ApplicationInfo available, i.e. running on a test Context: MultiDex support library is disabled.");
                return;
            }

            doInstallation(context, new File(applicationInfo.sourceDir), new File(applicationInfo.dataDir), "secondary-dexes", "", true);
        } catch (Exception var2) {
            Log.e("MultiDex", "MultiDex installation failure", var2);
            throw new RuntimeException("MultiDex installation failed (" + var2.getMessage() + ").");
        }

        Log.i("MultiDex", "install done");
    }
}
```



```java
private static void doInstallation(Context mainContext, File sourceApk, File dataDir,
                                   String secondaryFolderName, String prefsKeyPrefix, boolean reinstallOnPatchRecoverableException)
  throws IOException, IllegalArgumentException, IllegalAccessException,
         NoSuchFieldException, InvocationTargetException, NoSuchMethodException,
         SecurityException, ClassNotFoundException, InstantiationException {
    synchronized(installedApk) {
        if (!installedApk.contains(sourceApk)) {
            installedApk.add(sourceApk);
            if (VERSION.SDK_INT > 20) {
                Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT
                      + ": SDK version higher than " + 20 + " should be backed by "
                      + "runtime with built-in multidex capabilty but it's not the "
                      + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
            }

            ClassLoader loader;
            try {
                loader = mainContext.getClassLoader();
            } catch (RuntimeException var25) {
                Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var25);
                return;
            }

            if (loader == null) {
                Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
            } else {
                try {
                    clearOldDexDir(mainContext);
                } catch (Throwable var24) {
                    Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var24);
                }

                // dexDir路径:/data/data/com.phantomvk.playground/code_cache/secondary-dexes
                // dataDir: /data/data/com.phantomvk.playground
                // sec....Name: secondary-dexes
                File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
                // sourceApk路径:/data/app/com.phantomvk.playground-1.apk
                MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);
                IOException closeException = null;

                try {
                    // 加载
                    List files = extractor.load(mainContext, prefsKeyPrefix, false);

                    try {
                        installSecondaryDexes(loader, dexDir, files);
                    } catch (IOException var26) {
                        if (!reinstallOnPatchRecoverableException) {
                            throw var26;
                        }

                        Log.w("MultiDex", "Failed to install extracted secondary dex files, retrying with forced extraction", var26);
                        files = extractor.load(mainContext, prefsKeyPrefix, true);
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

## MultiDexExtractor

```java
List<? extends File> load(Context context, String prefsKeyPrefix, boolean forceReload) throws IOException {
    Log.i("MultiDex", "MultiDexExtractor.load(" + this.sourceApk.getPath() + ", " + forceReload + ", " + prefsKeyPrefix + ")");
    if (!this.cacheLock.isValid()) {
        throw new IllegalStateException("MultiDexExtractor was closed");
    } else {
        List files;
        if (!forceReload && !isModified(context, this.sourceApk, this.sourceCrc, prefsKeyPrefix)) {
            try {
                files = this.loadExistingExtractions(context, prefsKeyPrefix);
            } catch (IOException var6) {
                Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var6);
                files = this.performExtractions();
                putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk), this.sourceCrc, files);
            }
        } else {
            if (forceReload) {
                Log.i("MultiDex", "Forced extraction must be performed.");
            } else {
                Log.i("MultiDex", "Detected that extraction must be performed.");
            }

            files = this.performExtractions();
            putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk), this.sourceCrc, files);
        }

        Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
        return files;
    }
}
```

```java
private List<MultiDexExtractor.ExtractedDex> performExtractions() throws IOException {
    // com.phantomvk.playground-1.apk.classes
    String extractedFilePrefix = this.sourceApk.getName() + ".classes";
    this.clearDexDir();
    List<MultiDexExtractor.ExtractedDex> files = new ArrayList();
    ZipFile apk = new ZipFile(this.sourceApk);

    try {
        int secondaryNumber = 2;

        // 从apk内逐个获取dex文件，把dex文件写为zip文件
        for(ZipEntry dexFile = apk.getEntry("classes" + secondaryNumber + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + secondaryNumber + ".dex")) {
            // fileName = com.phantomvk.playground-1.apk.classes2.zip
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";
            // extractedFile = /data/data/com.phantomvk.playground/code_cache/secondary-dexes/com.phantomvk.playground-1.apk.classes2.zip
            MultiDexExtractor.ExtractedDex extractedFile = new MultiDexExtractor.ExtractedDex(this.dexDir, fileName);
            files.add(extractedFile);
            Log.i("MultiDex", "Extraction is needed for file " + extractedFile);
            int numAttempts = 0;
            boolean isExtractionSuccessful = false;

            while(numAttempts < 3 && !isExtractionSuccessful) {
                ++numAttempts;
                // dexFile: classes2.dex
                extract(apk, dexFile, extractedFile, extractedFilePrefix);

                try {
                    extractedFile.crc = getZipCrc(extractedFile);
                    isExtractionSuccessful = true;
                } catch (IOException var18) {
                    isExtractionSuccessful = false;
                    Log.w("MultiDex", "Failed to read crc from " + extractedFile.getAbsolutePath(), var18);
                }

                Log.i("MultiDex", "Extraction " + (isExtractionSuccessful ? "succeeded" : "failed")
                      + " '" + extractedFile.getAbsolutePath() + "': length " + extractedFile.length()
                      + " - crc: " + extractedFile.crc);
                if (!isExtractionSuccessful) {
                    extractedFile.delete();
                    if (extractedFile.exists()) {
                        Log.w("MultiDex", "Failed to delete corrupted secondary dex '" + extractedFile.getPath() + "'");
                    }
                }
            }

            if (!isExtractionSuccessful) {
                throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + secondaryNumber + ")");
            }

            ++secondaryNumber;
        }
    } finally {
        try {
            apk.close();
        } catch (IOException var17) {
            Log.w("MultiDex", "Failed to close resource", var17);
        }

    }

    return files;
}
```

从apk文件中提取dex为zip文件

```java
private static void extract(ZipFile apk, ZipEntry dexFile, File extractTo, String extractedFilePrefix) throws IOException, FileNotFoundException {
    InputStream in = apk.getInputStream(dexFile);
    ZipOutputStream out = null;
    // tmp = /data/data/com.phantomvk.playground/code_cache/secondary-dexes/tmp-com.phantomvk.playground-1.apk.classes-2082951698.zip
    File tmp = File.createTempFile("tmp-" + extractedFilePrefix, ".zip", extractTo.getParentFile());
    Log.i("MultiDex", "Extracting " + tmp.getPath());

    try {
        // 用temp文件创建ZipOutputStream
        out = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(tmp)));

        try {
            ZipEntry classesDex = new ZipEntry("classes.dex");
            classesDex.setTime(dexFile.getTime());
            out.putNextEntry(classesDex);
            byte[] buffer = new byte[16384];

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

        Log.i("MultiDex", "Renaming to " + extractTo.getPath());
        if (!tmp.renameTo(extractTo)) {
            throw new IOException("Failed to rename \"" + tmp.getAbsolutePath() + "\" to \"" + extractTo.getAbsolutePath() + "\"");
        }
    } finally {
        closeQuietly(in);
        tmp.delete();
    }
}
```

安装 __SecondaryDexes__，根据不同版本进入不同分支

```java
private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<? extends File> files)
  throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException,
InvocationTargetException, NoSuchMethodException, IOException,
SecurityException, ClassNotFoundException, InstantiationException {
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

__Android4.4__ 进入V19

```java
private static final class V19 {
    private V19() {
    }

    static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries, File optimizedDirectory)
      throws IllegalArgumentException, IllegalAccessException,
  NoSuchFieldException, InvocationTargetException,
  NoSuchMethodException, IOException {
        // private final dalvik.system.DexPathList dalvik.system.BaseDexClassLoader.pathList
        Field pathListField = MultiDex.findField(loader, "pathList");
        // DexPathList[[zip file "/data/app/com.phantomvk.playground-1.apk"],nativeLibraryDirectories=[/data/app-lib/com.phantomvk.playground-1, /system/lib]]
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new ArrayList();
        MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
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
                System.arraycopy(dexElementsSuppressedExceptions, 0, combined, suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
                dexElementsSuppressedExceptions = combined;
            }

            suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
            IOException exception = new IOException("I/O exception during makeDexElement");
            exception.initCause((Throwable)suppressedExceptions.get(0));
            throw exception;
        }
    }

    private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions)
      throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        // 反射获得makeDexElements()
        Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class, ArrayList.class);
        // 调用方法
        return (Object[])((Object[])makeDexElements.invoke(dexPathList, files, optimizedDirectory, suppressedExceptions));
    }
}
```

执行以下方法前

```java
MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
```

执行之后

