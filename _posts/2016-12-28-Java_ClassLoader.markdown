---
layout:     post
title:      "分析ClassLoader原理"
date:       2016-12-28
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - JVM
---

# 类加载

Java类通过编译生成对应.class文件，JVM根据运行需要，把所需类从.class动态加载到内存并创建实例，ClassLoader负责完成这个加载任务。有了ClassLoader，Java运行时系统不需要知道文件与文件系统的设置。

正是因为Java类必须由某个类加载器装入到内存，我们也可以在运行时才指定类文件。

# Java中的三个默认类加载器

除了Bootstrap ClassLoader，其他类装载器都有父装载器（parent class loader），且ExtClassLoader和AppClassLoader均继承ClassLoader类。

* 引导（Bootstrap）类加载器。由原生代码（C++）编写，不继承自`java.lang.ClassLoader`。负责加载存储在`<JAVA_HOME>/jre/lib`目录中的核心Java库。

* 扩展（Extensions）类加载器。用来在`<JAVA_HOME>/jre/lib/ext`或`java.ext.dirs`指明的目录中加载 Java扩展库，Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。该类由`sun.misc.Launcher$ExtClassLoader`实现。

* 应用（Application）类加载器。根据 Java应用程序的类路径`（java.class.path或CLASSPATH环境变量）`来加载 Java 类。一般来说，Java 应用的类都是由它来完成加载的，可以通过 `ClassLoader.getSystemClassLoader()`来获取。该类由`sun.misc.Launcher$AppClassLoader实现`。



![img](/img/java/classloader.png)

__通过Bootstrap ClassLoader加载的库__

```java
URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
for (URL url : urls) {
    System.out.println(url.toExternalForm());
}
```

结果

```
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/resources.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/rt.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/sunrsasign.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jsse.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jce.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/charsets.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/lib/jfr.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre/classes
```

# ClassLoader类加载

ClassLoader通过双亲委托的方式来搜索类，而双亲委托是一种委派思想。当ClassLoader加载类的时候，ClassLoader会委托其父加载器去完成：

* Bootstrap ClassLoader尝试加载该类；
* 加载失败把工作交给ExtClassLoader；
* ExtClassLoader加载失败把工作交给AppClassLoader；

如果三个默认类加载器都加载失败，工作只能还给发起工作的ClassLoader，由这个加载器自行选择加载类文件系统或URL。如果所有加载器都无法加载这个类，JVM抛出ClassNotFoundException异常。

如果按照这个加载步骤成功，类加载器会把这个类载入内存，初始化并返回实例。


# 双亲委托

使用双亲委托是为了两个目的：运行安全和避免重复加载。

后者容易理解，类已经被父加载器加载的话，子加载器没有必要再次重复加载；对于前者，有些系统级的类涉及到整个JVM的运行安全，仅能通过Bootstrap ClassLoader加载。如果不使用双亲委托，使用自定义的类来动态替代Java核心定义类型，后续系统将完全处于危险和混乱之中。


# ClassLoader层次结构

先编写类加载的代码，其中`Man`是一个自行定义的普通类

```java
final String dir = "file:/Users/phantomVK/repositories/intelliJ/cl/src";
URLClassLoader loader = new URLClassLoader(new URL[] { new URL(dir) });
Class clazz = loader.loadClass("com.phantomvk.Man");
ClassLoader classLoader = clazz.getClassLoader();

while (classLoader != null) {
    System.out.println(classLoader);
    classLoader = classLoader.getParent();
}

System.out.println(classLoader);
```
加载Man的类加载器显示结果

```
sun.misc.Launcher$AppClassLoader@4b67cf4d
sun.misc.Launcher$ExtClassLoader@61bbe9ba
null
```

加载层次：

* Man类的类加载器是AppClassLoader
* AppClassLoader的类加载器是ExtClassLoader
* ExtClassLoader的类加载器是BootstrapLoader。

BootstrapLoader由C++实现而不是Java，不运行在JVM中。所以ExtClassLoader的类加载器没法显示BootstrapLoader的引用地址，只能显示null。


# 自定义ClassLoader

自定义ClassLoader比较简单

* 只需要继承ClassLoader父类；
* 仅重写Class<?> findClass(String name)方法，查找并返回这个类；
* 剩余的加载过程由父类完成，无需手动处理。






