---
layout:     post
title:      "Java switch不支持基本类型long"
date:       2018-06-11
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - JVM
---


### 原文
内容摘自  << The Java Virtual Machine Specification Java SE 10 Edition >> PDF - Page57

```
3.10. Compiling Switches

Compilation of switch statements uses the tableswitch and lookupswitch 
instructions. The tableswitch instruction is used when the cases of the 
switch can be efficiently represented as indices into a table of target 
offsets. The default target of the switch is used if the value of the 
expression of the switch falls outside the range of valid indices. For 
instance:

    int chooseNear(int i) {
        switch (i) {
            case 0:  return  0;
            case 1:  return  1;
            case 2:  return  2;
            default: return -1;
        }
    }

compiles to:

    Method int chooseNear(int)
    0   iload_1             // Push local variable 1 (argument i)
    1   tableswitch 0 to 2: // Valid indices are 0 through 2
          0: 28             // If i is 0, continue at 28
          1: 30             // If i is 1, continue at 30
          2: 32             // If i is 2, continue at 32
          default:34        // Otherwise, continue at 34
    28  iconst_0            // i was 0; push int constant 0...
    29  ireturn             // ...and return it
    30  iconst_1            // i was 1; push int constant 1...
    31  ireturn             // ...and return it
    32  iconst_2            // i was 2; push int constant 2...
    33  ireturn             // ...and return it
    34  iconst_m1           // otherwise push int constant -1...
    35  ireturn             // ...and return it

The Java Virtual Machine's tableswitch and lookupswitch instructions 
operate only on int data. Because operations on byte, char, or short 
values are internally promoted to int, a switch whose expression 
evaluates to one of those types is compiled as though it evaluated to 
type int. If the chooseNear method had been written using type short, the 
same Java Virtual Machine instructions would have been generated as when 
using type int. Other numeric types must be narrowed to type int for use 
in a switch.
```

### 解释

1. 上文明确指出JVM的`tableswitch`和`lookupswitch`指令只能操作int数据:

   __The Java Virtual Machine's tableswitch and lookupswitch instructions operate only on int data.__

2. 同时表明在`switch`中执行`byte`、`char`、`short`会隐式向上转型为基本类型`int`

### 参考链接

- [why-cant-java-switch-over-the-primitive-long](https://stackoverflow.com/questions/13951419/why-cant-java-switch-over-the-primitive-long)

- [Java Language and Virtual Machine Specifications](https://docs.oracle.com/javase/specs/)