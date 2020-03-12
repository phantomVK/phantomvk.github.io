---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---

```java
package access_inline;

public class AccessInline {

    private int mValue;

    private int doStuff(int value) {
        return value;
    }

    private class Inner {
        void stuff() {
            AccessInline.this.doStuff(0);
        }
    }
}
```

```shell
$  javac -source 1.7 -target 1.7  AccessInline.java
```

生成实例变量 __mValue__ 的 __access$000__  方法

```
static int access$000(access_inline.AccessInline);
  descriptor: (Laccess_inline/AccessInline;)I
  flags: (0x1008) ACC_STATIC, ACC_SYNTHETIC
  Code:
    stack=1, locals=1, args_size=1
       0: aload_0
       1: getfield      #2                  // Field mValue:I
       4: ireturn
```

生成实例方法 __doStuff(int)__ 的 __access$100__  方法

```
static int access$100(access_inline.AccessInline, int);
  descriptor: (Laccess_inline/sAccessInline;I)I
  flags: (0x1008) ACC_STATIC, ACC_SYNTHETIC
  Code:
    stack=2, locals=2, args_size=2
       0: aload_0
       1: iload_1
       2: invokespecial #1                  // Method doStuff:(I)I
       5: ireturn
```

