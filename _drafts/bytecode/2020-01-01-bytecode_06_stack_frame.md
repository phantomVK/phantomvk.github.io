---
layout:     post
title:      "Java字节码 — 栈帧"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - ByteCode
---

### 关于locals, args_size

```java
static void staticMethod(){ }
static void staticMethod(int a){ }
static void staticMethod(int a, int b){ }
static void staticMethod(int a, int b,int c){ }
```

由于方法内没有运算实体，所有方法的stack值均为0

|     方法名(无局部变量)      | locals | args_size |
| :-------------------------: | :----: | :-------: |
|       staticMethod()        |   0    |     0     |
|      staticMethod(int)      |   1    |     1     |
|   staticMethod(int, int)    |   2    |     2     |
| staticMethod(int, int, int) |   3    |     3     |

```java
static void staticMethod() { int z = 0; }
static void staticMethod(int a) { int z = 0; }
static void staticMethod(int a, int b) { int z = 0; }
static void staticMethod(int a, int b, int c) { int z = 0; }
```

|   方法名(含局部变量一个)    | locals | args_size |
| :-------------------------: | :----: | :-------: |
|       staticMethod()        |   1    |     0     |
|      staticMethod(int)      |   2    |     1     |
|   staticMethod(int, int)    |   3    |     2     |
| staticMethod(int, int, int) |   4    |     3     |


```java
void method(){ }
void method(int a){ }
void method(int a, int b){ }
void method(int a, int b,int c){ }
```

|  方法名(无局部变量)   | locals | args_size |
| :-------------------: | :----: | :-------: |
|       method()        |   1    |     1     |
|      method(int)      |   2    |     2     |
|   method(int, int)    |   3    |     3     |
| method(int, int, int) |   4    |     4     |

## 参考链接

- [轻松看懂Java字节码](https://juejin.im/post/5aca2c366fb9a028c97a5609)
