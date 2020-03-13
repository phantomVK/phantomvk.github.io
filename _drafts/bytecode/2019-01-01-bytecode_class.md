---
layout:     post
title:      "Java字节码(1) — 入门与类"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - ByteCode
---

## 开篇

这是“Java字节码”系列文章开篇文章。原预定写一篇独立文章，用来介绍字节码和虚拟机的关系。后来想想，认为对学习字节码有需求的读者，想必对 __Java__ 语言和 __JVM__ 有一定了解。于是把开篇文浓缩为这一段开篇词，省去长篇累牍。

由于字节码是所有 __Java-based__ 的基础，所以基于 __JVM__ 运行的语言会"编译"为字节码这种中间语言。除了 __Java__ 外，__Kotlin__ 受欢迎程度日渐提升。不过相比 __Java__ ，由于 __Kotlin__ 的 “糖” 分更高，同样功能对应字节码更复杂，普及性暂没 __Java__ 高。所以这个系列文章，选择 __Java__ 作为编程语言，通过最简单的语法，从简到易一步一步深入字节码的魅力。

另外一点，系列章节间没有绝对关系，也不会像出版书籍一样完美设计目录。但所有文章均不要求读者对字节码指令有相关经验，毕竟写这个系列的目标是记录字节码学习历程。有必要细化讨论的细节会另起文章讨论，再通过链接的方法补充到系列内。

文章基于__Oracle Java8__ 及 __Java8+__ JDK编译字节码，根据以往经历表明，部分编译出来的字节码与 __JDK7__ 有明显区别。有生产需要的读者 __务必__ 以实际JDK为准。

笔者技术水平有限，难免出现错误，请见谅并留下您的评论。

## 类

这是 __Java__ 初学者都会练习的类，仅包含没有作用的入口 __main__ 方法

```java
public class ByteCode {
    public static void main(String[] args) {}
}
```

通过 __javac__ 指令编译源码为 __ByteCode.class__。日后无特别说明，均以这个命令编译

```bash
$ javac ByteCode.java
```

再用 __javap__ 把 __ByteCode.class__ 的字节码内容输出为文本，以便查看

```bash
$ javap -private -verbose ByteCode > ByteCode.txt
```

#### 完整内容

这是编译为字节码后的完整文本，后面会分段解析

```
Classfile /Users/phantomvk/java/ByteCode/src/ByteCode.class
  Last modified 2020年3月5日; size 261 bytes
  MD5 checksum 978111a1d1d3faa0f7b7724b1b98ed4a
  Compiled from "ByteCode.java"
public class ByteCode
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ByteCode
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #3.#12         // java/lang/Object."<init>":()V
   #2 = Class              #13            // ByteCode
   #3 = Class              #14            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               main
   #9 = Utf8               ([Ljava/lang/String;)V
  #10 = Utf8               SourceFile
  #11 = Utf8               ByteCode.java
  #12 = NameAndType        #4:#5          // "<init>":()V
  #13 = Utf8               ByteCode
  #14 = Utf8               java/lang/Object
{
  public ByteCode();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 3: 0
}
SourceFile: "ByteCode.java"
```

#### 文件属性

```
Classfile /Users/phantomvk/java/ByteCode/src/ByteCode.class
  Last modified 2020年3月5日; size 261 bytes
  MD5 checksum 978111a1d1d3faa0f7b7724b1b98ed4a
  Compiled from "ByteCode.java"
```

#### 类属性

```
public class ByteCode
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ByteCode
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
```

#### 常量池

```
Constant pool:
   #1 = Methodref          #3.#12         // java/lang/Object."<init>":()V
   #2 = Class              #13            // ByteCode
   #3 = Class              #14            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               main
   #9 = Utf8               ([Ljava/lang/String;)V
  #10 = Utf8               SourceFile
  #11 = Utf8               ByteCode.java
  #12 = NameAndType        #4:#5          // "<init>":()V
  #13 = Utf8               ByteCode
  #14 = Utf8               java/lang/Object
```

#### 方法体

```
{
  public ByteCode();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 3: 0
}
```

```
SourceFile: "ByteCode.java"
```