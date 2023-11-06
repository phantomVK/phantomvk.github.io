---
layout:     post
title:      "MacOS MAT无法创建JVM"
date:       2020-12-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - JavaVM
---

### 问题

MacOS用JDK8启动MAT时报错，提示内容为：

```
Failed to create the Java Virtual Machine.
```

如图：

![MemoryAnalyzer_VM](/img/java/MemoryAnalyzer_VM.png)



### 解决办法

打开MAT的配置文件

```bash
/Applications/mat.app/Contents/Eclipse/MemoryAnalyzer.ini
```

在配置文件增加以下内容：请替换为已安装JDK的版本，并注意换行格式

```
-vm
/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/bin
```

以下为完成后的配置文件，注意内容增加的位置

```
-startup
../Eclipse/plugins/org.eclipse.equinox.launcher_1.5.700.v20200207-2156.jar
--launcher.library
../Eclipse/plugins/org.eclipse.equinox.launcher.cocoa.macosx.x86_64_1.1.1100.v20190907-0426
-vm
/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/bin
-vmargs
-Xmx1024m
-Dorg.eclipse.swt.internal.carbon.smallFonts
-XstartOnFirstThread
```

重启启动MAT即可