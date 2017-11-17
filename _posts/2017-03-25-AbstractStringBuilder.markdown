---
layout:     post
title:      "Java源码系列(4) -- AbstractStringBuilder"
date:       2017-03-25
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

`AbstractStringBuilder`是`StringBuilder`和`StringBuffer`的父类，包含字符串操作的实现逻辑，子类根据各自需求对方法调用做同步处理，本身是线程不安全的。作为可变字符串的父类，`AbstractStringBuilder`借助字符数组的形式实现可变字符串，类中大部分方法是字符修改、插入、追加等操作。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence
```

实现`CharSequence`，所以两个子类`StringBuilder`和`StringBuffer`的实例可以直接赋值给`CharSequence`。

## 二、数据成员

```java
char[] value; // 保存字符串的数组
int count;    // 记录字符长度
```

## 三、构造方法

```java
// 必要的无参构造方法，用于序列化和子类
AbstractStringBuilder() { }

// 通过指定容量构造字符串数组
AbstractStringBuilder(int capacity) {
    value = new char[capacity];
}
```

## 四、成员方法

### 4.1 长度相关

已保存在字符串数组的字符个数

```java
@Override
public int length() {
    return count;
}
```

字符串数组的空间长度，包含可插入新字符的空间。在同一时刻，总有`length()`小于等于`capacity()`

```java
public int capacity() {
    return value.length;
}
```

### 4.2 增加容量

保证字符数组大小至少要和具体的最小值相等。若当前容量小于参数具体值，则创建一个更大的数组，数值取下列之大者：

   * minimumCapacity的具体大小；
   * 原容量值的两倍

若minimumCapacity不可用，下列直接退出。本类有方法减少字符数组的长度

```java
public void ensureCapacity(int minimumCapacity) {
    if (minimumCapacity > 0)
        ensureCapacityInternal(minimumCapacity);
}

// 包含参数检查，最小容量不得小于已有字符串数组长度，没有同步特性
private void ensureCapacityInternal(int minimumCapacity) {
    // 仅minimumCapacity大于数组长度可用
    if (minimumCapacity - value.length > 0)
        expandCapacity(minimumCapacity);
}

// 容量扩增方法，没有大小检查或加入同步特性
void expandCapacity(int minimumCapacity) {
    int newCapacity = value.length * 2 + 2;
    if (newCapacity - minimumCapacity < 0)
        newCapacity = minimumCapacity;
    if (newCapacity < 0) {
        if (minimumCapacity < 0) // 下溢
            throw new OutOfMemoryError();
        newCapacity = Integer.MAX_VALUE;
    }
    value = Arrays.copyOf(value, newCapacity);
}
```

### 4.3 裁剪、增加

可变字符数组有空余用于插入更多字符，可以通过裁剪剩余空间达到提高内存使用效率的目的。

如果内存空间不紧张或没有特殊要求，不要使用这个方法，因为向裁剪后的字符数组添加字符一定会引起扩容。而且裁剪也是一次数据内容拷贝的过程，超大数组的情况下可能会引起性能问题。

实现方式：把字符串拷贝到长度刚好合适的字符数组中返回，释放原数组空间

```java
public void trimToSize() {
    if (count < value.length) {
        value = Arrays.copyOf(value, count);
    }
}
```

通过添加字符'\0'，令字符串长度达到`newLength`，仅在`newLength`大于数组内字符串长度有效。

```java
public void setLength(int newLength) {
    if (newLength < 0)
        throw new StringIndexOutOfBoundsException(newLength);
        
    ensureCapacityInternal(newLength);

    if (count < newLength) {
        Arrays.fill(value, count, newLength, '\0');
    }

    count = newLength;
}
```

### 4.4 查找

返回指定索引值下的字符

```java
@Override
public char charAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    return value[index];
}
```

返回指定索引下字符的编码值

```java
public int codePointAt(int index) {
    if ((index < 0) || (index >= count)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointAtImpl(value, index, count);
}
```

返回指定索引前一位字符的编码值

```java
public int codePointBefore(int index) {
    int i = index - 1;
    if ((i < 0) || (i >= count)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointBeforeImpl(value, index, 0);
}
```

```java
public int codePointCount(int beginIndex, int endIndex) {
    if (beginIndex < 0 || endIndex > count || beginIndex > endIndex) {
        throw new IndexOutOfBoundsException();
    }
    return Character.codePointCountImpl(value, beginIndex, endIndex-beginIndex);
}
```

```java
public int offsetByCodePoints(int index, int codePointOffset) {
    if (index < 0 || index > count) {
        throw new IndexOutOfBoundsException();
    }
    return Character.offsetByCodePointsImpl(value, 0, count,
                                            index, codePointOffset);
}
```

复制指定范围的字符串到给定的字符数组中

```java
public void getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)
{
    if (srcBegin < 0)
        throw new StringIndexOutOfBoundsException(srcBegin);
    if ((srcEnd < 0) || (srcEnd > count))
        throw new StringIndexOutOfBoundsException(srcEnd);
    if (srcBegin > srcEnd)
        throw new StringIndexOutOfBoundsException("srcBegin > srcEnd");
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```


### 4.5 append

```java

public AbstractStringBuilder append(Object obj) {
    return append(String.valueOf(obj));
}

public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}

// 因为同步特性的差异，具体请看子类的文档说明(注:StringBuilder和StringBuffer)
public AbstractStringBuilder append(StringBuffer sb) {
    if (sb == null)
        return appendNull();
    int len = sb.length();
    ensureCapacityInternal(count + len);
    sb.getChars(0, len, value, count);
    count += len;
    return this;
}

AbstractStringBuilder append(AbstractStringBuilder asb) {
    if (asb == null)
        return appendNull();
    int len = asb.length();
    ensureCapacityInternal(count + len);
    asb.getChars(0, len, value, count);
    count += len;
    return this;
}

// 因为同步特性的差异，具体请看子类的文档说明(注:StringBuilder和StringBuffer)
@Override
public AbstractStringBuilder append(CharSequence s) {
    if (s == null)
        return appendNull();
    if (s instanceof String)
        return this.append((String)s);
    if (s instanceof AbstractStringBuilder)
        return this.append((AbstractStringBuilder)s);

    return this.append(s, 0, s.length());
}

private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l';
    count = c;
    return this;
}

@Override
public AbstractStringBuilder append(CharSequence s, int start, int end) {
    if (s == null)
        s = "null";
    if ((start < 0) || (start > end) || (end > s.length()))
        throw new IndexOutOfBoundsException(
            "start " + start + ", end " + end + ", s.length() "
            + s.length());
    int len = end - start;
    ensureCapacityInternal(count + len);
    for (int i = start, j = count; i < end; i++, j++)
        value[j] = s.charAt(i);
    count += len;
    return this;
}

public AbstractStringBuilder append(char[] str) {
    int len = str.length;
    ensureCapacityInternal(count + len);
    System.arraycopy(str, 0, value, count, len);
    count += len;
    return this;
}

public AbstractStringBuilder append(char str[], int offset, int len) {
    if (len > 0)
        ensureCapacityInternal(count + len);
    System.arraycopy(str, offset, value, count, len);
    count += len;
    return this;
}

public AbstractStringBuilder append(boolean b) {
    if (b) {
        ensureCapacityInternal(count + 4);
        value[count++] = 't';
        value[count++] = 'r';
        value[count++] = 'u';
        value[count++] = 'e';
    } else {
        ensureCapacityInternal(count + 5);
        value[count++] = 'f';
        value[count++] = 'a';
        value[count++] = 'l';
        value[count++] = 's';
        value[count++] = 'e';
    }
    return this;
}

@Override
public AbstractStringBuilder append(char c) {
    ensureCapacityInternal(count + 1);
    value[count++] = c;
    return this;
}

public AbstractStringBuilder append(int i) {
    if (i == Integer.MIN_VALUE) {
        append("-2147483648");
        return this;
    }
    int appendedLength = (i < 0) ? Integer.stringSize(-i) + 1
                                 : Integer.stringSize(i);
    int spaceNeeded = count + appendedLength;
    ensureCapacityInternal(spaceNeeded);
    Integer.getChars(i, spaceNeeded, value);
    count = spaceNeeded;
    return this;
}

public AbstractStringBuilder append(long l) {
    if (l == Long.MIN_VALUE) {
        append("-9223372036854775808");
        return this;
    }
    int appendedLength = (l < 0) ? Long.stringSize(-l) + 1
                                 : Long.stringSize(l);
    int spaceNeeded = count + appendedLength;
    ensureCapacityInternal(spaceNeeded);
    Long.getChars(l, spaceNeeded, value);
    count = spaceNeeded;
    return this;
}

public AbstractStringBuilder append(float f) {
    FloatingDecimal.appendTo(f,this);
    return this;
}
eturn  a reference to this object.
 */
public AbstractStringBuilder append(double d) {
    FloatingDecimal.appendTo(d,this);
    return this;
}

public AbstractStringBuilder appendCodePoint(int codePoint) {
    final int count = this.count;

    if (Character.isBmpCodePoint(codePoint)) {
        ensureCapacityInternal(count + 1);
        value[count] = (char) codePoint;
        this.count = count + 1;
    } else if (Character.isValidCodePoint(codePoint)) {
        ensureCapacityInternal(count + 2);
        Character.toSurrogates(codePoint, value, count);
        this.count = count + 2;
    } else {
        throw new IllegalArgumentException();
    }
    return this;
}
```


### 4.6 移除字符

移除字符串

```java
public AbstractStringBuilder delete(int start, int end) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        end = count;
    if (start > end)
        throw new StringIndexOutOfBoundsException();
    int len = end - start;
    if (len > 0) {
        System.arraycopy(value, start+len, value, start, count-end);
        count -= len;
    }
    return this;
}
```

移除一个字符

```java
public AbstractStringBuilder deleteCharAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    System.arraycopy(value, index+1, value, index, count-index-1);
    count--;
    return this;
}
```

### 4.7 替换

```java
public AbstractStringBuilder replace(int start, int end, String str) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (start > count)
        throw new StringIndexOutOfBoundsException("start > length()");
    if (start > end)
        throw new StringIndexOutOfBoundsException("start > end");

    if (end > count)
        end = count;
    int len = str.length();
    int newCount = count + len - (end - start);
    ensureCapacityInternal(newCount);

    System.arraycopy(value, end, value, start + len, count - end);
    str.getChars(value, start);
    count = newCount;
    return this;
}
```

```java
public void setCharAt(int index, char ch) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    value[index] = ch;
}
```

### 4.8 获取子序列

从指定起点获取后面字符序列，结果是`String`

```java
public String substring(int start) {
    return substring(start, count);
}
```

获取给定范围内的字符序列，结果是`CharSequence`

```java
@Override
public CharSequence subSequence(int start, int end) {
    return substring(start, end);
}
```

获取给定范围内的字符序列，结果是`String`

```java
public String substring(int start, int end) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        throw new StringIndexOutOfBoundsException(end);
    if (start > end)
        throw new StringIndexOutOfBoundsException(end - start);
    return new String(value, start, end - start);
}
```

### 4.9 insert

```java
public AbstractStringBuilder insert(int index, char[] str, int offset,
                                    int len)
{
    if ((index < 0) || (index > length()))
        throw new StringIndexOutOfBoundsException(index);
    if ((offset < 0) || (len < 0) || (offset > str.length - len))
        throw new StringIndexOutOfBoundsException(
            "offset " + offset + ", len " + len + ", str.length "
            + str.length);
    ensureCapacityInternal(count + len);
    System.arraycopy(value, index, value, index + len, count - index);
    System.arraycopy(str, offset, value, index, len);
    count += len;
    return this;
}

public AbstractStringBuilder insert(int offset, Object obj) {
    return insert(offset, String.valueOf(obj));
}

public AbstractStringBuilder insert(int offset, String str) {
    if ((offset < 0) || (offset > length()))
        throw new StringIndexOutOfBoundsException(offset);
    if (str == null)
        str = "null";
    int len = str.length();
    ensureCapacityInternal(count + len);
    System.arraycopy(value, offset, value, offset + len, count - offset);
    str.getChars(value, offset);
    count += len;
    return this;
}

public AbstractStringBuilder insert(int offset, char[] str) {
    if ((offset < 0) || (offset > length()))
        throw new StringIndexOutOfBoundsException(offset);
    int len = str.length;
    ensureCapacityInternal(count + len);
    System.arraycopy(value, offset, value, offset + len, count - offset);
    System.arraycopy(str, 0, value, offset, len);
    count += len;
    return this;
}

public AbstractStringBuilder insert(int dstOffset, CharSequence s) {
    if (s == null)
        s = "null";
    if (s instanceof String)
        return this.insert(dstOffset, (String)s);
    return this.insert(dstOffset, s, 0, s.length());
}

 public AbstractStringBuilder insert(int dstOffset, CharSequence s,
                                     int start, int end) {
    if (s == null)
        s = "null";
    if ((dstOffset < 0) || (dstOffset > this.length()))
        throw new IndexOutOfBoundsException("dstOffset "+dstOffset);
    if ((start < 0) || (end < 0) || (start > end) || (end > s.length()))
        throw new IndexOutOfBoundsException(
            "start " + start + ", end " + end + ", s.length() "
            + s.length());
    int len = end - start;
    ensureCapacityInternal(count + len);
    System.arraycopy(value, dstOffset, value, dstOffset + len,
                     count - dstOffset);
    for (int i=start; i<end; i++)
        value[dstOffset++] = s.charAt(i);
    count += len;
    return this;
}

public AbstractStringBuilder insert(int offset, boolean b) {
    return insert(offset, String.valueOf(b));
}

public AbstractStringBuilder insert(int offset, char c) {
    ensureCapacityInternal(count + 1);
    System.arraycopy(value, offset, value, offset + 1, count - offset);
    value[offset] = c;
    count += 1;
    return this;
}

public AbstractStringBuilder insert(int offset, int i) {
    return insert(offset, String.valueOf(i));
}

public AbstractStringBuilder insert(int offset, long l) {
    return insert(offset, String.valueOf(l));
}

public AbstractStringBuilder insert(int offset, float f) {
    return insert(offset, String.valueOf(f));
}

public AbstractStringBuilder insert(int offset, double d) {
    return insert(offset, String.valueOf(d));
}
```


### 4.10 查索引

```java
public int indexOf(String str) {
    return indexOf(str, 0);
}

public int indexOf(String str, int fromIndex) {
    return String.indexOf(value, 0, count, str, fromIndex);
}

public int lastIndexOf(String str) {
    return lastIndexOf(str, count);
}

public int lastIndexOf(String str, int fromIndex) {
    return String.lastIndexOf(value, 0, count, str, fromIndex);
}
```

### 4.11 字符序列翻转

```java
public AbstractStringBuilder reverse() {
    boolean hasSurrogates = false;
    int n = count - 1;
    for (int j = (n-1) >> 1; j >= 0; j--) {
        int k = n - j;
        char cj = value[j];
        char ck = value[k];
        value[j] = ck;
        value[k] = cj;
        if (Character.isSurrogate(cj) ||
            Character.isSurrogate(ck)) {
            hasSurrogates = true;
        }
    }
    if (hasSurrogates) {
        reverseAllValidSurrogatePairs();
    }
    return this;
}

private void reverseAllValidSurrogatePairs() {
    for (int i = 0; i < count - 1; i++) {
        char c2 = value[i];
        if (Character.isLowSurrogate(c2)) {
            char c1 = value[i + 1];
            if (Character.isHighSurrogate(c1)) {
                value[i++] = c1;
                value[i] = c2;
            }
        }
    }
}
```

### 4.12 其他

获取字符串序列

```java
final char[] getValue() {
    return value;
}
```

