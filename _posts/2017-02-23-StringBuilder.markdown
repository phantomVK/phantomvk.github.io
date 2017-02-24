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

`StringBuilder`是可变的字符串序列，API上和`StringBuffer`兼容。`StringBuilder`不保证线程安全，而`StringBuffer`可以。

在单线程中，由于没有线程同步的要求，所以`StringBuilder`在单线程中比`StringBuffer`拥有更好的性能。同样道理，多线程中应该使用`StringBuffer`保证字符串修改的安全。

# 二、类签名

`StringBuilder`继承`AbstractStringBuilder`，实现了`Serializable`和`CharSequence`接口。在`AbstractStringBuilder`父类中，构造方法、字符串增删查改等方法都已经完成实现，`StringBuilder`只是在自己的方法中调用父类方法。

```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

而`StringBuffer`同样继承自`AbstractStringBuilder`，在调用父类的过程中增加了`synchronized`代码块同步锁，仅此而已。这也是为什么`StringBuilder`和`StringBuffer`在API互相兼容、同步支持存在差异的原因。

只有阅读`AbstractStringBuilder`才能了解可变字符串是如何实现，仅看`StringBuffer`和`StringBuilder`没有太大帮助。`AbstractStringBuilder`源码将在`Java源码系列`后续篇章出现。


# 三、数据成员


```java
static final long serialVersionUID = 4383685877147921099L;
```

# 四、构造方法

可通过数值来初始化一个StringBuilder，或使用默认构造方法，或使用一个字符创来创建StringBuilder。使用默认构造方法的字符串长度是16，使用自定义字符串则会在原字符串尾加16个字符的缓冲长度。

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

# 五、方法

### 5.1 添加

尾添加对象，准确来说应该是obj.toString()

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

// IndexOutOfBoundsException
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

// IndexOutOfBoundsException
@Override
public StringBuilder append(char[] str, int offset, int len) {
    super.append(str, offset, len);
    return this;
}
```

```java
@Override
public StringBuilder append(boolean b) {
    super.append(b);
    return this;
}
```

尾添加布尔值，"true"或"false"的字符串

```java
@Override
public StringBuilder append(char c) {
    super.append(c);
    return this;
}
```

尾添加数值

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

### 5.2 删除

删除指定下标或范围的字符，都可能抛出`StringIndexOutOfBoundsException`

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


### 5.3 替换
替换字符串

```java
// StringIndexOutOfBoundsException
@Override
public StringBuilder replace(int start, int end, String str) {
    super.replace(start, end, str);
    return this;
}
```

### 5.4 插入

插入字符串

```java
// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int index, char[] str, int offset,
                            int len)
{
    super.insert(index, str, offset, len);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, Object obj) {
        super.insert(offset, obj);
        return this;
}

// StringIndexOutOfBoundsException@Override
public StringBuilder insert(int offset, String str) {
    super.insert(offset, str);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, char[] str) {
    super.insert(offset, str);
    return this;
}

// IndexOutOfBoundsException
@Override
public StringBuilder insert(int dstOffset, CharSequence s) {
        super.insert(dstOffset, s);
        return this;
}

// IndexOutOfBoundsException
@Override
public StringBuilder insert(int dstOffset, CharSequence s,
                            int start, int end)
{
    super.insert(dstOffset, s, start, end);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, boolean b) {
    super.insert(offset, b);
    return this;
}

// IndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, char c) {
    super.insert(offset, c);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, int i) {
    super.insert(offset, i);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, long l) {
    super.insert(offset, l);
    return this;
}

// IndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, float f) {
    super.insert(offset, f);
    return this;
}

// StringIndexOutOfBoundsException
@Override
public StringBuilder insert(int offset, double d) {
    super.insert(offset, d);
    return this;
}
```

### 5.5 查找

查找字符在字符串中的序号

```java
@Override
public int indexOf(String str) {
    return super.indexOf(str);
}

@Override
public int indexOf(String str, int fromIndex) {
    return super.indexOf(str, fromIndex);
}

@Override
public int lastIndexOf(String str) {
    return super.lastIndexOf(str);
}

@Override
public int lastIndexOf(String str, int fromIndex) {
    return super.lastIndexOf(str, fromIndex);
}
```

### 5.6 翻转

反转字符串

```java
@Override
public StringBuilder reverse() {
    super.reverse();
    return this;
}
```
### 5.7 toString

`toString()`方法返回字符串的拷贝，而不是字符串数组本身

```java
@Override
public String toString() {
    return new String(value, 0, count);
}
```

# 六、实现序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    s.defaultWriteObject();
    s.writeInt(count);
    s.writeObject(value);
}

private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    count = s.readInt();
    value = (char[]) s.readObject();
}
```

