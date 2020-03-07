---
layout:     post
title:      "Java字节码 — 变量可访问修饰"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - ByteCode
---

```java
public class ByteCode {
    int B = 0; // package access
    public int D = 0;
    private int A = 0;
    protected int C = 0;
}
```

```
Classfile /Users/tanwenkang/java/ByteCode/src/ByteCode.class
  Last modified 2020年3月6日; size 318 bytes
  MD5 checksum f1e8e9e1fa2f6132a2e763a4fa289a2b
  Compiled from "ByteCode.java"
public class ByteCode
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #6                          // ByteCode
  super_class: #7                         // java/lang/Object
  interfaces: 0, fields: 4, methods: 1, attributes: 1
Constant pool:
   #1 = Methodref          #7.#19         // java/lang/Object."<init>":()V
   #2 = Fieldref           #6.#20         // ByteCode.B:I
   #3 = Fieldref           #6.#21         // ByteCode.D:I
   #4 = Fieldref           #6.#22         // ByteCode.A:I
   #5 = Fieldref           #6.#23         // ByteCode.C:I
   #6 = Class              #24            // ByteCode
   #7 = Class              #25            // java/lang/Object
   #8 = Utf8               B
   #9 = Utf8               I
  #10 = Utf8               D
  #11 = Utf8               A
  #12 = Utf8               C
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               SourceFile
  #18 = Utf8               ByteCode.java
  #19 = NameAndType        #13:#14        // "<init>":()V
  #20 = NameAndType        #8:#9          // B:I
  #21 = NameAndType        #10:#9         // D:I
  #22 = NameAndType        #11:#9         // A:I
  #23 = NameAndType        #12:#9         // C:I
  #24 = Utf8               ByteCode
  #25 = Utf8               java/lang/Object
{
  int B;
    descriptor: I
    flags: (0x0000)

  public int D;
    descriptor: I
    flags: (0x0001) ACC_PUBLIC

  private int A;
    descriptor: I
    flags: (0x0002) ACC_PRIVATE

  protected int C;
    descriptor: I
    flags: (0x0004) ACC_PROTECTED

  public ByteCode();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_0
         6: putfield      #2                  // Field B:I
         9: aload_0
        10: iconst_0
        11: putfield      #3                  // Field D:I
        14: aload_0
        15: iconst_0
        16: putfield      #4                  // Field A:I
        19: aload_0
        20: iconst_0
        21: putfield      #5                  // Field C:I
        24: return
      LineNumberTable:
        line 1: 0
        line 2: 4
        line 3: 9
        line 4: 14
        line 5: 19
}
SourceFile: "ByteCode.java"
```

```
// class version 55.0 (55)
// access flags 0x21
public class ByteCode {

  // compiled from: ByteCode.java

  // access flags 0x0
  I B

  // access flags 0x1
  public I D

  // access flags 0x2
  private I A

  // access flags 0x4
  protected I C

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 1 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
   L1
    LINENUMBER 2 L1
    ALOAD 0
    ICONST_0
    PUTFIELD ByteCode.B : I
   L2
    LINENUMBER 3 L2
    ALOAD 0
    ICONST_0
    PUTFIELD ByteCode.D : I
   L3
    LINENUMBER 4 L3
    ALOAD 0
    ICONST_0
    PUTFIELD ByteCode.A : I
   L4
    LINENUMBER 5 L4
    ALOAD 0
    ICONST_0
    PUTFIELD ByteCode.C : I
    RETURN
   L5
    LOCALVARIABLE this LByteCode; L0 L5 0
    MAXSTACK = 2
    MAXLOCALS = 1
}
```

