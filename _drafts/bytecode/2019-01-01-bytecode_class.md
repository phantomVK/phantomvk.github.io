---
layout:     post
title:      "Java字节码 — 入门"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - ByteCode
---

```java
public class ByteCode {
    public static void main(String[] args) {}
}
```

通过 __javac__ 指令编译源码为 __ByteCode.class__

```bash
$ javac ByteCode.java
```

再用 __javap__ 把 __ByteCode.class__ 的字节码内容输出为文本，以便查看

```bash
$ javap -private -verbose ByteCode > ByteCode.txt
```

#### 完整内容

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