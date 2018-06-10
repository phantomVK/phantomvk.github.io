---
layout:     post
title:      "Glide 4.0 StringSignature 找不到类"
date:       2017-12-14
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android
---

## 一、起因

旧版本`Glide`中`.signature()`方法形参支持使用`StringSignature`。但在`Glide 4.0`里面这个方法不仅被移到`RequestOptions`中，而且形参改为`Key`。

## 二、解决办法

### 2.1 Key

不过`Key`是一个接口，需要实现抽象方法

```java
void updateDiskCacheKey(MessageDigest messageDigest);
```

### 2.2 ObjectKey

`Key`还有一个使用相对方便的子类`ObjectKey`，构造方法以`Object`为参数，下面是实际用法:

```java
public static RequestOptions userAvatarOptions = new RequestOptions()
            .placeholder(R.drawable.def_avatar)
            .error(R.drawable.def_avatar)
            .signature(new ObjectKey(System.currentTimeMillis()))
            .encodeQuality(70);
```

代码中创建了一个`ObjectKey`实例，并把当前时间戳整形值作为参数。

## 三、源码

顺便贴出`Key`和`ObjectKey`的源码，请自行查阅:

### 3.1 Key

```java
public interface Key {
  String STRING_CHARSET_NAME = "UTF-8";
  Charset CHARSET = Charset.forName(STRING_CHARSET_NAME);

  void updateDiskCacheKey(MessageDigest messageDigest);

  @Override
  boolean equals(Object o);

  @Override
  int hashCode();
}
```


### 3.2 ObjectKey

```java
public final class ObjectKey implements Key {
  private final Object object;

  public ObjectKey(Object object) {
    this.object = Preconditions.checkNotNull(object);
  }

  @Override
  public String toString() {
    return "ObjectKey{"
        + "object=" + object
        + '}';
  }

  @Override
  public boolean equals(Object o) {
    if (o instanceof ObjectKey) {
      ObjectKey other = (ObjectKey) o;
      return object.equals(other.object);
    }
    return false;
  }

  @Override
  public int hashCode() {
    return object.hashCode();
  }

  // Charset CHARSET = Charset.forName("UTF-8");
  @Override
  public void updateDiskCacheKey(MessageDigest messageDigest) {
    messageDigest.update(object.toString().getBytes(CHARSET));
  }
}
```

## 四、参考链接

* [StringSignature class not found #2692](https://github.com/bumptech/glide/issues/2692)

