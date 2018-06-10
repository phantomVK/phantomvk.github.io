---
layout:     post
title:      "apkbuilder找不到的问题"
date:       2016-11-23
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

apkbuilder在 Android SDK build tools r22里面被移除了。如果我们还需要使用这个工具的话，可以通过以下方式重新取得apkbuilder。

切换工作目录，把工作目录切换到tools下面。例子中是在linux平台的tools.

```
android-sdk-linux/tools
```

对于Windows：

```bash
$ copy android.bat apkbuilder.bat 
```
然后修改apkbuilder的内容

```bash
$ modify apkbuilder.bat: change com.android.sdkmanager.Main to com.android.sdklib.build.ApkBuilderMain
```

对于Linux/Mac，在tools下面执行以下命令，创建apkbuilder文件：

```bash
$ cat android | sed -e 's/com.android.sdkmanager.Main/com.android.sdklib.build.ApkBuilderMain/g' > apkbuilder 
```

给apkbuilder赋予执行权：

```bash
$ chmod a+x apkbuilder
```

参考来源：[android-build-fails-no-such-file-or-directory-apkbuilder](http://stackoverflow.com/questions/19273237/android-build-fails-no-such-file-or-directory-apkbuilder)

