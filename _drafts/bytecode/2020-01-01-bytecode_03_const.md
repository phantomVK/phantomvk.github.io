---
layout:     post
title:      "Java字节码 — 常量"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - ByteCode
---

```java
public class ByteCode {
    private static final char CONST_CHAR = 0;
    private static final byte CONST_BYTE = 0;
    private static final short CONST_SHORT = 0;
    private static final int CONST_INT = 0;
    private static final long CONST_LONG = 0L;
    private static final float CONST_FLOAT = 10.0F;
    private static final double CONST_DOUBLE = 10.0;
    private static final String CONST_STR = "Const";
}
```

```
public class ByteCode
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ByteCode
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 8, methods: 1, attributes: 1
Constant pool:
   #1 = Methodref          #3.#34         // java/lang/Object."<init>":()V
   #2 = Class              #35            // ByteCode
   #3 = Class              #36            // java/lang/Object
   #4 = Utf8               CONST_CHAR
   #5 = Utf8               C
   #6 = Utf8               ConstantValue
   #7 = Integer            0
   #8 = Utf8               CONST_BYTE
   #9 = Utf8               B
  #10 = Utf8               CONST_SHORT
  #11 = Utf8               S
  #12 = Utf8               CONST_INT
  #13 = Utf8               I
  #14 = Utf8               CONST_LONG
  #15 = Utf8               J
  #16 = Long               0l
  #18 = Utf8               CONST_FLOAT
  #19 = Utf8               F
  #20 = Float              10.0f
  #21 = Utf8               CONST_DOUBLE
  #22 = Utf8               D
  #23 = Double             10.0d
  #25 = Utf8               CONST_STR
  #26 = Utf8               Ljava/lang/String;
  #27 = String             #37            // ConstValue
  #28 = Utf8               <init>
  #29 = Utf8               ()V
  #30 = Utf8               Code
  #31 = Utf8               LineNumberTable
  #32 = Utf8               SourceFile
  #33 = Utf8               ByteCode.java
  #34 = NameAndType        #28:#29        // "<init>":()V
  #35 = Utf8               ByteCode
  #36 = Utf8               java/lang/Object
  #37 = Utf8               ConstValue
{
  public static final char CONST_CHAR;
    descriptor: C
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 0

  public static final byte CONST_BYTE;
    descriptor: B
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 0

  public static final short CONST_SHORT;
    descriptor: S
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 0

  public static final int CONST_INT;
    descriptor: I
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 0

  public static final long CONST_LONG;
    descriptor: J
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: long 0l

  public static final float CONST_FLOAT;
    descriptor: F
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: float 10.0f

  public static final double CONST_DOUBLE;
    descriptor: D
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: double 10.0d

  public static final java.lang.String CONST_STR;
    descriptor: Ljava/lang/String;
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: String ConstValue

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
}
SourceFile: "ByteCode.java"
```

```
public class ByteCode {
  public final static C CONST_CHAR = 0
  public final static B CONST_BYTE = 0
  public final static S CONST_SHORT = 0
  public final static I CONST_INT = 0
  public final static J CONST_LONG = 0
  public final static F CONST_FLOAT = 10.0
  public final static D CONST_DOUBLE = 10.0
  public final static Ljava/lang/String; CONST_STR = "ConstValue"

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 1 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this LByteCode; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1
}
```

