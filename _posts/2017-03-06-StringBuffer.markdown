---
layout:     post
title:      "Java源码系列 -- StringBuffer"
date:       2017-03-06
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

相信看过 [Java源码系列(2) -- StringBuilder](http://phantomvk.coding.me/2017/02/23/StringBuilder/) 的读者都了解`StringBuilder`和`StringBuffer`的异同，这里我们再复习一次加深印象。

```java
public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

`StringBuilder`和`StringBuffer`同样继承自`AbstractStringBuilder`父类，字符串序列增删查改等主要操作均在父类中实现。`StringBuilder`调用父类方法的时候，不会对正在进行的操作进行同步处理，而`StringBuffer`相反。一个同步锁的差异能影响多线程修改字符串序列的行为方式和线程处理过程中的性能。除此之外，他们俩都支持序列化。

## 二、数据成员

```java
// 当字符串被修改之后，值最新缓存会被toString返回
private transient char[] toStringCache;
static final long serialVersionUID = 3388685877147921107L;
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

为了保证线程安全，几乎所有方法都使用`synchronized`修饰符。同步锁加在这个字符串序列实例上，只要有一个线程已经占用这个实例，其他线程只能等待上一个线程释放同步再去竞争。

值得一提的是，`synchronied`锁有三种类别，分别是`对象锁`、`实例锁`、`类锁`。`对象锁`是类中有一个静态常量Object对象作代码块锁的锁对象，只要同步代码块已经持有该对象，其他同步代码块只能等待上一个锁释放；`实例锁`的实例方法使用`synchronied`作为锁目标，只要该实例有一个同步方法在使用，其他线程对同一个实例同步操作只能等待上一个同步结束；`类锁`针对类静态方法需要使用线程同步的情况。由于静态类不需要初始化构造就能被访问，所以需要上锁的地方只能把整个类作为锁目标。

### 4.1 序列长

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


### 4.2 查找

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
    // Note, synchronization achieved via invocations of other StringBuffer methods
    return super.indexOf(str);
}

@Override
public synchronized int indexOf(String str, int fromIndex) {
    return super.indexOf(str, fromIndex);
}

@Override
public int lastIndexOf(String str) {
    // Note, synchronization achieved via invocations of other StringBuffer methods
    return lastIndexOf(str, count);
}

@Override
public synchronized int lastIndexOf(String str, int fromIndex) {
    return super.lastIndexOf(str, fromIndex);
}
```

### 4.3 追加

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

/**
 * Appends the specified {@code StringBuffer} to this sequence.
 * <p>
 * The characters of the {@code StringBuffer} argument are appended,
 * in order, to the contents of this {@code StringBuffer}, increasing the
 * length of this {@code StringBuffer} by the length of the argument.
 * If {@code sb} is {@code null}, then the four characters
 * {@code "null"} are appended to this {@code StringBuffer}.
 * <p>
 * Let <i>n</i> be the length of the old character sequence, the one
 * contained in the {@code StringBuffer} just prior to execution of the
 * {@code append} method. Then the character at index <i>k</i> in
 * the new character sequence is equal to the character at index <i>k</i>
 * in the old character sequence, if <i>k</i> is less than <i>n</i>;
 * otherwise, it is equal to the character at index <i>k-n</i> in the
 * argument {@code sb}.
 * <p>
 * This method synchronizes on {@code this}, the destination
 * object, but does not synchronize on the source ({@code sb}).
 *
 * @param   sb   the {@code StringBuffer} to append.
 * @return  a reference to this object.
 * @since 1.4
 */
public synchronized StringBuffer append(StringBuffer sb) {
    toStringCache = null;
    super.append(sb);
    return this;
}

/**
 * @since 1.8
 */
@Override
synchronized StringBuffer append(AbstractStringBuilder asb) {
    toStringCache = null;
    super.append(asb);
    return this;
}

/**
 * Appends the specified {@code CharSequence} to this
 * sequence.
 * <p>
 * The characters of the {@code CharSequence} argument are appended,
 * in order, increasing the length of this sequence by the length of the
 * argument.
 *
 * <p>The result of this method is exactly the same as if it were an
 * invocation of this.append(s, 0, s.length());
 *
 * <p>This method synchronizes on {@code this}, the destination
 * object, but does not synchronize on the source ({@code s}).
 *
 * <p>If {@code s} is {@code null}, then the four characters
 * {@code "null"} are appended.
 *
 * @param   s the {@code CharSequence} to append.
 * @return  a reference to this object.
 * @since 1.5
 */
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

### 4.4 替换

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

### 4.5 子序列

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

### 4.6 插入

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

### 4.7 翻转

```java
@Override
public synchronized StringBuffer reverse() {
    toStringCache = null;
    super.reverse();
    return this;
}
```

### 4.8 删除

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

### 4.9 toString
```java
@Override
public synchronized String toString() {
    if (toStringCache == null) {
        toStringCache = Arrays.copyOfRange(value, 0, count);
    }
    return new String(toStringCache, true);
}
```

## 五、序列化

```java
private static final java.io.ObjectStreamField[] serialPersistentFields =
{
    new java.io.ObjectStreamField("value", char[].class),
    new java.io.ObjectStreamField("count", Integer.TYPE),
    new java.io.ObjectStreamField("shared", Boolean.TYPE),
};

private synchronized void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    java.io.ObjectOutputStream.PutField fields = s.putFields();
    fields.put("value", value);
    fields.put("count", count);
    fields.put("shared", false);
    s.writeFields();
}

private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    java.io.ObjectInputStream.GetField fields = s.readFields();
    value = (char[])fields.get("value", null);
    count = fields.get("count", 0);
}
```

