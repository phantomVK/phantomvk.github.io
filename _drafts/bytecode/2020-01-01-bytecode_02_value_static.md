---
layout:     post
title:      "Java字节码 — 实例变量与静态变量"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - ByteCode
---

```java
public class ByteCode {
    public int A;

    public void putValue() {
        A = 100;
    }
}
```

```
public I A
public putValue()V
  ALOAD 0
  BIPUSH 100
  PUTFIELD ByteCode.A : I
```

```java
public class ByteCode {
    public static int A;

    public void putValue() {
        A = 100;
    }
}
```

```
public static I A
public putValue()V
  BIPUSH 100
  PUTSTATIC ByteCode.A : I
```

|        实例变量         |         静态变量         |
| :---------------------: | :----------------------: |
|         ALOAD 0         |                          |
|       BIPUSH 100        |        BIPUSH 100        |
| PUTFIELD ByteCode.A : I | PUTSTATIC ByteCode.A : I |

