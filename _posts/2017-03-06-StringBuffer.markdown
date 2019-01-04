---
layout:     post
title:      "Java源码系列(3) -- StringBuffer"
date:       2017-03-06
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

相信看过 [StringBuilder](/2017/02/23/StringBuilder/) 的读者都了解`StringBuilder`和`StringBuffer`的异同，这里我们再重温一次。

```java
public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

`StringBuilder`和`StringBuffer`同样继承自[AbstractStringBuilder](/2017/03/25/AbstractStringBuilder/)父类，字符串序列增删查改等主要操作由父类实现。`StringBuilder`调用父类方法线程不安全，而`StringBuffer`操作线程安全，均支持序列化。

## 二、数据成员

当字符串被修改之后，此值会被更新

```java
private transient char[] toStringCache;
```

## 三、构造方法

默认创建长度为16的字符串序列，也可以自定义字符串序列长度。若用一个字符串为构造参数的话，构造方法会在原字符串上再加上16的缓冲长度。

```java
public StringBuffer() {
    super(16);
}

public StringBuffer(int capacity) {
    super(capacity);
}

public StringBuffer(String str) {
    super(str.length() + 16);
    append(str);
}

public StringBuffer(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```


## 四、成员方法

为了保证线程安全，几乎所有方法都使用`synchronized`修饰符。同步锁加在字符串序列实例上，只要线程已经占用这个实例，其他线程只能等待线程释放同步锁再去竞争。

__synchronied__ 锁有三种类别：

- __对象锁__ 通常由一个常量实例作代码块的锁对象，只要同步代码块已经持有该对象，其他同步代码块只能等待上一个锁释放；
- __实例锁__ 的实例方法使用`synchronied`作为锁目标，只要该实例有一个同步方法在使用，其他线程对同一个实例同步操作只能等待上一个同步结束；
- __类锁__ 针对类静态方法需要使用线程同步的情况。由于静态类不需要new就能访问，所以只能把整个类作为锁目标。

#### 4.1 序列长度

```java
@Override
public synchronized int length() {
    return count;
}

@Override
public synchronized int capacity() {
    return value.length;
}

@Override
public synchronized void ensureCapacity(int minimumCapacity) {
    if (minimumCapacity > value.length) {
        expandCapacity(minimumCapacity);
    }
}

@Override
public synchronized void trimToSize() {
    super.trimToSize();
}

@Override
public synchronized void setLength(int newLength) {
    toStringCache = null;
    super.setLength(newLength);
}
```


#### 4.2 查找

```java
@Override
public synchronized char charAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    return value[index];
}

@Override
public synchronized int codePointAt(int index) {
    return super.codePointAt(index);
}

@Override
public synchronized int codePointBefore(int index) {
    return super.codePointBefore(index);
}

@Override
public synchronized int codePointCount(int beginIndex, int endIndex) {
    return super.codePointCount(beginIndex, endIndex);
}

@Override
public synchronized int offsetByCodePoints(int index, int codePointOffset) {
    return super.offsetByCodePoints(index, codePointOffset);
}

@Override
public synchronized void getChars(int srcBegin, int srcEnd, char[] dst,
                                  int dstBegin)
{
    super.getChars(srcBegin, srcEnd, dst, dstBegin);
}

@Override
public int indexOf(String str) {
    // 备注:这个方法的同步依赖于其他StringBuffer方法的实现
    return super.indexOf(str);
}

@Override
public synchronized int indexOf(String str, int fromIndex) {
    return super.indexOf(str, fromIndex);
}

@Override
public int lastIndexOf(String str) {
    // 备注:这个方法的同步依赖于其他StringBuffer方法的实现
    return lastIndexOf(str, count);
}

@Override
public synchronized int lastIndexOf(String str, int fromIndex) {
    return super.lastIndexOf(str, fromIndex);
}
```

#### 4.3 追加

```java
@Override
public synchronized StringBuffer append(Object obj) {
    toStringCache = null;
    super.append(String.valueOf(obj));
    return this;
}

@Override
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}

public synchronized StringBuffer append(StringBuffer sb) {
    toStringCache = null;
    super.append(sb);
    return this;
}

@Override
synchronized StringBuffer append(AbstractStringBuilder asb) {
    toStringCache = null;
    super.append(asb);
    return this;
}

@Override
public synchronized StringBuffer append(CharSequence s) {
    toStringCache = null;
    super.append(s);
    return this;
}

@Override
public synchronized StringBuffer append(CharSequence s, int start, int end)
{
    toStringCache = null;
    super.append(s, start, end);
    return this;
}

@Override
public synchronized StringBuffer append(char[] str) {
    toStringCache = null;
    super.append(str);
    return this;
}

@Override
public synchronized StringBuffer append(char[] str, int offset, int len) {
    toStringCache = null;
    super.append(str, offset, len);
    return this;
}

@Override
public synchronized StringBuffer append(boolean b) {
    toStringCache = null;
    super.append(b);
    return this;
}

@Override
public synchronized StringBuffer append(char c) {
    toStringCache = null;
    super.append(c);
    return this;
}

@Override
public synchronized StringBuffer append(int i) {
    toStringCache = null;
    super.append(i);
    return this;
}

@Override
public synchronized StringBuffer appendCodePoint(int codePoint) {
    toStringCache = null;
    super.appendCodePoint(codePoint);
    return this;
}

@Override
public synchronized StringBuffer append(long lng) {
    toStringCache = null;
    super.append(lng);
    return this;
}

@Override
public synchronized StringBuffer append(float f) {
    toStringCache = null;
    super.append(f);
    return this;
}

@Override
public synchronized StringBuffer append(double d) {
    toStringCache = null;
    super.append(d);
    return this;
}
```

#### 4.4 替换

```java
@Override
public synchronized StringBuffer replace(int start, int end, String str) {
    toStringCache = null;
    super.replace(start, end, str);
    return this;
}

@Override
public synchronized void setCharAt(int index, char ch) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    toStringCache = null;
    value[index] = ch;
}
```

#### 4.5 子序列

```java
@Override
public synchronized String substring(int start) {
    return substring(start, count);
}

@Override
public synchronized CharSequence subSequence(int start, int end) {
    return super.substring(start, end);
}

@Override
public synchronized String substring(int start, int end) {
    return super.substring(start, end);
}
```

#### 4.6 插入

```java
@Override
public synchronized StringBuffer insert(int index, char[] str, int offset,
                                        int len)
{
    toStringCache = null;
    super.insert(index, str, offset, len);
    return this;
}

@Override
public synchronized StringBuffer insert(int offset, Object obj) {
    toStringCache = null;
    super.insert(offset, String.valueOf(obj));
    return this;
}

@Override
public synchronized StringBuffer insert(int offset, String str) {
    toStringCache = null;
    super.insert(offset, str);
    return this;
}

@Override
public synchronized StringBuffer insert(int offset, char[] str) {
    toStringCache = null;
    super.insert(offset, str);
    return this;
}


@Override
public StringBuffer insert(int dstOffset, CharSequence s) {
    // Note, synchronization achieved via invocations of other StringBuffer methods
    // after narrowing of s to specific type
    // Ditto for toStringCache clearing
    super.insert(dstOffset, s);
    return this;
}

@Override
public synchronized StringBuffer insert(int dstOffset, CharSequence s,
        int start, int end)
{
    toStringCache = null;
    super.insert(dstOffset, s, start, end);
    return this;
}

@Override
public  StringBuffer insert(int offset, boolean b) {
    // Note, synchronization achieved via invocation of StringBuffer insert(int, String)
    // after conversion of b to String by super class method
    // Ditto for toStringCache clearing
    super.insert(offset, b);
    return this;
}

@Override
public synchronized StringBuffer insert(int offset, char c) {
    toStringCache = null;
    super.insert(offset, c);
    return this;
}

@Override
public StringBuffer insert(int offset, int i) {
    // Note, synchronization achieved via invocation of StringBuffer insert(int, String)
    // after conversion of i to String by super class method
    // Ditto for toStringCache clearing
    super.insert(offset, i);
    return this;
}

@Override
public StringBuffer insert(int offset, long l) {
    // Note, synchronization achieved via invocation of StringBuffer insert(int, String)
    // after conversion of l to String by super class method
    // Ditto for toStringCache clearing
    super.insert(offset, l);
    return this;
}

@Override
public StringBuffer insert(int offset, float f) {
    // Note, synchronization achieved via invocation of StringBuffer insert(int, String)
    // after conversion of f to String by super class method
    // Ditto for toStringCache clearing
    super.insert(offset, f);
    return this;
}

@Override
public StringBuffer insert(int offset, double d) {
    // Note, synchronization achieved via invocation of StringBuffer insert(int, String)
    // after conversion of d to String by super class method
    // Ditto for toStringCache clearing
    super.insert(offset, d);
    return this;
}
```

#### 4.7 翻转

```java
@Override
public synchronized StringBuffer reverse() {
    toStringCache = null;
    super.reverse();
    return this;
}
```

#### 4.8 删除

```java
@Override
public synchronized StringBuffer delete(int start, int end) {
    toStringCache = null;
    super.delete(start, end);
    return this;
}

@Override
public synchronized StringBuffer deleteCharAt(int index) {
    toStringCache = null;
    super.deleteCharAt(index);
    return this;
}
```

#### 4.9 toString
```java
@Override
public synchronized String toString() {
    if (toStringCache == null) {
        toStringCache = Arrays.copyOfRange(value, 0, count);
    }
    return new String(toStringCache, true);
}
```
