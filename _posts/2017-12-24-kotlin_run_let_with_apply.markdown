---
layout:     post
title:      "Kotlin - let, apply, with, run的差别和用法"
date:       2017-12-24
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Kotlin
---

## 前言

下面除了`with`之外，所有用例都来自投入到生产的代码。因为项目没有用`with`的语法，所以通过其他例子来示意。在不影响理解的情况下，用例移除了业务相关代码。

## 一、let

调用传入的闭包，并用`it`指代`T`，返回值与闭包的返回值一致。

```kotlin
/**
 * Calls the specified function [block] with `this` value as its argument and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

### 用例一

把`color:Int`颜色值设置到`text:TextView`。

```kotlin
color.let {
    try {
        text.setTextColor(Color.parseColor(it))
    } catch (e: Exception) {
        Log.e("Class.name", "setTextColor($color) -> ${e.message}")
    }
}
```
### 用例二

当`results:MutableList<String>`不为空时从`results`按`position`取对应值然后绑定。

```kotlin
results?.let {
    if (it.isNotEmpty()) {
        viewHolder.bind(holder, it[position])
    }
}
```

## 二、apply

`apply`使用`this`指代`T`，闭包返回值是`Unit`。但是`apply`方法会自动返回`T`，通过`return this`实现。

```kotlin
/**
 * Calls the specified function [block] with `this` value as its receiver and returns `this` value.
 */
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```


### 用例一

创建`LinearLayout`并利用`apply`设置参数，最后返回初始化完毕的`LinearLayout`实例。

```kotlin
val linearLayout = LinearLayout(itemView.context).apply {
    orientation = LinearLayout.VERTICAL
    layoutParams = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT)
}
```

上面的代码等价于：

```kotlin
val linearLayout = LinearLayout(itemView.context)
linearLayout.orientation = LinearLayout.VERTICAL
linearLayout.layoutParams = LinearLayout.LayoutParams(
        LinearLayout.LayoutParams.MATCH_PARENT,
        LinearLayout.LayoutParams.WRAP_CONTENT)
```

### 用例二

根据`newProgress:Int`来修改`progressBar`的状态。

```kotlin
progressBar.apply {
    progress = newProgress
    visibility = if (newProgress in 1..99) View.VISIBLE else View.GONE
}
```

等价下列代码：

```kotlin
progressBar.progress = newProgress
progressBar.visibility = if (newProgress in 1..99) View.VISIBLE else View.GONE
```

## 三、with

接受一个`receiver`和一个函数，通过`this`调用`receiver`，返回值根据函数中最后一个返回值为准。

一般会在传入`receiver`的时候就地创建实例，否则可以使用`apply`来替代`with`。

```kotlin
/**
 * Calls the specified function [block] with the given [receiver] as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

### 用例一

使用`ArrayList<String>()`作为闭包的参数，返回类型为`println()`的返回值`Unit`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    println("this = " + this)
}
```
### 用例二

使用`ArrayList<String>()`做为闭包的参数，返回值类型为`ArrayList<String>`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    this
}
```

### 用例三

使用`ArrayList<String>()`做为闭包的参数，返回值类型为`3.1415926`的类型`Double`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    3.1415926
}
```

## 四、run

执行传入的函数，并返回函数的返回值。`run`的主要目的是强调需要执行函数里面的逻辑。

根据函数签名可见，`run`不运行在任何变量上，既不能使用`it`，也不能使用`this`指代任何参数。

```kotlin
/**
 * Calls the specified function [block] and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

/**
 * Calls the specified function [block] with `this` value as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

### 用例

从`intent`取`EXTRA_URL`的值，不为非空且内容不为空，赋值给`url`。否则弹出提示并关闭页面。

```kotlin
// Init url:String
url = intent.getStringExtra(EXTRA_URL)?.takeIf { it.isNotEmpty() } ?: run {
    toast("不能浏览一个空链接哦")
    activity.finish()
}

// If url is not null, continues...
```

