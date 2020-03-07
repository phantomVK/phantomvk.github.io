---
layout:     post
title:      "Java字节码 — 父类与接口"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - ByteCode
---

```java
public class Apple extends Food {
}
```

```java
public class Food {
}
```

```
public class Apple extends Food
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // Apple
  super_class: #3                         // Food
  interfaces: 0, fields: 0, methods: 1, attributes: 1
Constant pool:
   #1 = Methodref          #3.#10         // Food."<init>":()V
   #2 = Class              #11            // Apple
   #3 = Class              #12            // Food
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               SourceFile
   #9 = Utf8               Apple.java
  #10 = NameAndType        #4:#5          // "<init>":()V
  #11 = Utf8               Apple
  #12 = Utf8               Food
{
  public Apple();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method Food."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
}
```

```java
public class Apple extends Food implements InterfaceA, InterfaceB {
}
```

```
public class Apple extends Food implements InterfaceA,InterfaceB
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // Apple
  super_class: #3                         // Food
  interfaces: 2, fields: 0, methods: 1, attributes: 1
Constant pool:
   #1 = Methodref          #3.#12         // Food."<init>":()V
   #2 = Class              #13            // Apple
   #3 = Class              #14            // Food
   #4 = Class              #15            // InterfaceA
   #5 = Class              #16            // InterfaceB
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               SourceFile
  #11 = Utf8               Apple.java
  #12 = NameAndType        #6:#7          // "<init>":()V
  #13 = Utf8               Apple
  #14 = Utf8               Food
  #15 = Utf8               InterfaceA
  #16 = Utf8               InterfaceB
{
  public Apple();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method Food."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
}
```

