---
layout:     post
title:      "Kotlin lazy特性"
date:       2018-03-18
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Koltin
---

## 一、用法

Koltin的lazy懒加载主要有以下两种用法，根据具体实现看来只有方法二是懒加载的。

用法一：

```kotlin
val strLazyOf by lazyOf("LazyString")
```


用法二：

```kotlin
val strLazy by lazy { "LazyString" }
```

## 二、Lazy接口

`SynchronizedLazyImpl`和`InitializedLazyImpl`均实现Lazy接口

```kotlin
public interface Lazy<out T> {
    // 从当前懒加载实例中获取需加载值
    // 一旦值被初始化，该值在懒加载实例剩余生命周期都不应被修改
    public val value: T

    // 懒加载实例被初始化后此方法返回true，否则返回false
    public fun isInitialized(): Boolean
}
```

## 三、lazyOf源码


### 3.1 lazyOf

从实参value构造`InitializedLazyImpl`实例

```kotlin
public fun <T> lazyOf(value: T): Lazy<T> = InitializedLazyImpl(value)
```

### 3.2 InitializedLazyImpl

没有任何关于懒加载的逻辑

```kotlin
private class InitializedLazyImpl<out T>(override val value: T) : Lazy<T>, Serializable {

    override fun isInitialized(): Boolean = true

    override fun toString(): String = value.toString()
}
```

## 四、lazy {}源码

### 4.1 lazy

lazy线程安全，不需要在外层包加同步代码。

```kotlin
@kotlin.jvm.JvmVersion
public fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
```

### 4.2 SynchronizedLazyImpl

单例`UNINITIALIZED_VALUE`用于表示值未初始化

```kotlin
private object UNINITIALIZED_VALUE
```

实现`Lazy<T>`和`Serializable`两个接口

```kotlin
@JvmVersion
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    // 指定的初始值
    private var initializer: (() -> T)? = initializer

    // 值默认没有初始化，为UNINITIALIZED_VALUE，用Volatile关键字修饰
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE

    // 如果不传入锁对象，默认把当前实例作为上锁对象
    private val lock = lock ?: this
    
    // 重写父类值value的行为
    override val value: T
        get() {
            // 值已被初始化就返回该值，并进行类型转换
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }
            
            // 值没有经过初始化，先上同步锁
            return synchronized(lock) {
                val _v2 = _value
                // 双重校检，第一次是上方的检验
                // 下面是第二次校验
                if (_v2 !== UNINITIALIZED_VALUE) {
                    // aka: return value;
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                }
                else {
                    // 从initializer实例中获取输入值
                    val typedValue = initializer!!()
                    // 把值赋值给_value，_value保存在SynchronizedLazyImpl
                    _value = typedValue
                    // 使用过的initializer实例置null并回收
                    initializer = null
                    // aka: return typedValue;
                    typedValue
                }
            }
        }
    
    // 默认为false，经过初始化后变为true
    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```