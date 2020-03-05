---
layout:     post
title:      ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

```java
public class ByteCode {
    public static int value;
    public static void main(String[] args) { value = 100; }
}

```

```bash
$ javac ByteCode.java 
$ javap -v ByteCode > ByteCode.txt
```

```
Classfile /Users/tanwenkang/java/ByteCode/src/ByteCode.class
  Last modified 2020年3月5日; size 359 bytes
  MD5 checksum 8cebd14f7907276e7978bf0e1dad251e
  Compiled from "ByteCode.java"
public class ByteCode
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #4                          // ByteCode
  super_class: #5                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 3, attributes: 1
Constant pool:
   #1 = Methodref          #5.#17         // java/lang/Object."<init>":()V
   #2 = Methodref          #4.#18         // ByteCode.increase:()V
   #3 = Fieldref           #4.#19         // ByteCode.value:I
   #4 = Class              #20            // ByteCode
   #5 = Class              #21            // java/lang/Object
   #6 = Utf8               value
   #7 = Utf8               I
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               increase
  #15 = Utf8               SourceFile
  #16 = Utf8               ByteCode.java
  #17 = NameAndType        #8:#9          // "<init>":()V
  #18 = NameAndType        #14:#9         // increase:()V
  #19 = NameAndType        #6:#7          // value:I
  #20 = Utf8               ByteCode
  #21 = Utf8               java/lang/Object
{
  public static int value;
    descriptor: I
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC

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
         0: invokestatic  #2                  // Method increase:()V
         3: return
      LineNumberTable:
        line 3: 0

  public static void increase();
    descriptor: ()V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: bipush        100
         2: putstatic     #3                  // Field value:I
         5: return
      LineNumberTable:
        line 4: 0
}
SourceFile: "ByteCode.java"

```

