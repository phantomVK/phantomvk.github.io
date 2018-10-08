---
layout:     post
title:      "Java源码系列(2) -- StringBuilder"
date:       2017-02-23
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

# 一、前言

`StringBuilder`是可变字符串序列，API和[StringBuffer](http://phantomvk.github.io/2017/03/06/StringBuffer/)兼容。`StringBuilder`不保证线程安全，而[StringBuffer](http://phantomvk.github.io/2017/03/06/StringBuffer/)可以。

由于单线程没有线程同步的要求，所以`StringBuilder`在单线程中比`StringBuffer`拥有更好的性能。同样道理，多线程中应该使用`StringBuffer`保证字符串修改的安全。

# 二、类签名

`StringBuilder`继承`AbstractStringBuilder`，实现了`Serializable`和`CharSequence`接口。构造方法、字符串增删查改在`AbstractStringBuilder`父类中实现，`StringBuilder`只负责调用父类方法。

```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

`StringBuffer`同样继承自`AbstractStringBuilder`，调用方法时增加了`synchronized`同步锁。这是`StringBuilder`和`StringBuffer`在API互相兼容、线程安全存在差异的原因。由此可知，阅读[AbstractStringBuilder](https://phantomvk.github.io/2017/03/25/AbstractStringBuilder/)才能了解可变字符串如何实现，仅看`StringBuffer`和`StringBuilder`没有太大帮助。

# 三、构造方法

可通过字符串长度、默认构造方法，或一个字符创作为参数创建StringBuilder。默认构造方法的字符串长度是16，自定义字符串则会在原字符串尾加16个字符作为缓冲长度。

```java
public StringBuilder() {
    super(16);
}

public StringBuilder(int capacity) {
    super(capacity);
}

public StringBuilder(String str) {
    super(str.length() + 16);
    append(str);
}

public StringBuilder(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```

# 四、方法

### 4.1 添加

尾添加对象，准确来说应该是obj.toString()或“null”

```java
@Override
public StringBuilder append(Object obj) {
    return append(String.valueOf(obj));
}
```

尾添加字符串

```java
@Override
public StringBuilder append(String str) {
    super.append(str);
    return this;
}

public StringBuilder append(StringBuffer sb) {
    super.append(sb);
    return this;
}

@Override
public StringBuilder append(CharSequence s) {
    super.append(s);
    return this;
}

// 越界异常
@Override
public StringBuilder append(CharSequence s, int start, int end) {
    super.append(s, start, end);
    return this;
}

@Override
public StringBuilder append(char[] str) {
    super.append(str);
    return this;
}

// 越界异常
@Override
public StringBuilder append(char[] str, int offset, int len) {
    super.append(str, offset, len);
    return this;
}
```

尾添加布尔值，`true`或`false`的字符串

```java
@Override
public StringBuilder append(boolean b) {
    super.append(b);
    return this;
}
```

加入一个字符

```java
@Override
public StringBuilder append(char c) {
    super.append(c);
    return this;
}
```

尾添加数值，`byte`和`short`类型提升到`int`

```java
@Override
public StringBuilder append(int i) {
    super.append(i);
    return this;
}

@Override
public StringBuilder append(long lng) {
    super.append(lng);
    return this;
}

@Override
public StringBuilder append(float f) {
    super.append(f);
    return this;
}

@Override
public StringBuilder append(double d) {
    super.append(d);
    return this;
}
```

尾添加字符编码对应字符

```java
@Override
public StringBuilder appendCodePoint(int codePoint) {
    super.appendCodePoint(codePoint);
    return this;
}
```

### 4.2 删除

删除指定下标或范围的字符，可能抛出`StringIndexOutOfBoundsException`异常

```java
@Override
public StringBuilder delete(int start, int end) {
    super.delete(start, end);
    return this;
}

@Override
public StringBuilder deleteCharAt(int index) {
    super.deleteCharAt(index);
    return this;
}
```

### 4.3 替换
替换字符串

```java
@Override
public StringBuilder replace(int start, int end, String str) {
    super.replace(start, end, str);
    return this;
}
```

### 4.4 插入

插入字符串

```java
@Override
public StringBuilder insert(int index, char[] str, int offset, int len) {
    super.insert(index, str, offset, len);
    return this;
}

@Override
public StringBuilder insert(int offset, Object obj) {
        super.insert(offset, obj);
        return this;
}

@Override
public StringBuilder insert(int offset, String str) {
    super.insert(offset, str);
    return this;
}

@Override
public StringBuilder insert(int offset, char[] str) {
    super.insert(offset, str);
    return this;
}

@Override
public StringBuilder insert(int dstOffset, CharSequence s) {
        super.insert(dstOffset, s);
        return this;
}

@Override
public StringBuilder insert(int dstOffset, CharSequence s,
                            int start, int end)
{
    super.insert(dstOffset, s, start, end);
    return this;
}

@Override
public StringBuilder insert(int offset, boolean b) {
    super.insert(offset, b);
    return this;
}

@Override
public StringBuilder insert(int offset, char c) {
    super.insert(offset, c);
    return this;
}

@Override
public StringBuilder insert(int offset, int i) {
    super.insert(offset, i);
    return this;
}

@Override
public StringBuilder insert(int offset, long l) {
    super.insert(offset, l);
    return this;
}

@Override
public StringBuilder insert(int offset, float f) {
    super.insert(offset, f);
    return this;
}

@Override
public StringBuilder insert(int offset, double d) {
    super.insert(offset, d);
    return this;
}
```

### 4.5 查找

查找字符在字符串中的序号，可以指定字符串开始的下标

```java
@Override
public int indexOf(String str) {
    return super.indexOf(str);
}

@Override
public int indexOf(String str, int fromIndex) {
    return super.indexOf(str, fromIndex);
}
```

最后一次出现的下标

```java
@Override
public int lastIndexOf(String str) {
    return super.lastIndexOf(str);
}

@Override
public int lastIndexOf(String str, int fromIndex) {
    return super.lastIndexOf(str, fromIndex);
}
```

### 4.6 翻转

翻转字符串

```java
@Override
public StringBuilder reverse() {
    super.reverse();
    return this;
}
```
### 4.7 toString

返回字符串拷贝，而不是字符串数组本身

```java
@Override
public String toString() {
    return new String(value, 0, count);
}
```