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

下面除了`with`之外，所有用例都来自`Android`生产代码。因项目没有实际使用`with`语法，所以通过其他例子来示意。在不影响理解的情况下，所有用例移除业务相关代码。

文章中使用的`Kotlin`版本是`1.2.10`，不同版本的`Kotlin`标准库实现可能会有差异。

## 一、let

调用传入的函数式。接收者为`T`，且用`it`指代`T`，返回值与函数式返回值一致。

```kotlin
// 用`this`作为调用块的实参，并返回块执行结果
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

### 1.1 用例

把`color:Int`颜色值设置到`text:TextView`。

```kotlin
color.let {
    try {
        text.setTextColor(Color.parseColor(it))
    } catch (e: Exception) {
        Log.e("ClazzName", "setTextColor($color) -> ${e.message}")
    }
}
```
### 1.2 用例

当`results:MutableList<String>`不为空时，从`results`按`position`取对应值并绑定到`viewHolder`。

```kotlin
// results进行了判空操作
results?.let {
    if (it.isNotEmpty()) {
        viewHolder.bind(holder, it[position])
    }
}
```

## 二、apply

`apply`使用`this`指代`T`，函数值返回值是`Unit`。但`apply`通过`return this`主动返回`T`的实例。

```kotlin
// 用`this`作为执行块的接收者，并返回`this`的实例
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```


### 2.1 用例

创建`LinearLayout`并利用`apply`设置初始化参数，最后返回初始化完毕的`LinearLayout`实例。

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

如果构造过程中需要初始化的变量较多，使用`apply`形成的代码块会非常直观。

### 2.2 用例

根据`newProgress:Int`来修改`progressBar`的状态。

```kotlin
progressBar.apply {
    progress = newProgress
    visibility = if (newProgress in 1..99) View.VISIBLE else View.GONE
}
```

等价于下列代码：

```kotlin
progressBar.progress = newProgress
progressBar.visibility = if (newProgress in 1..99) View.VISIBLE else View.GONE
```

## 三、with

接收一个`receiver`和一个函数式，通过`this`调用`receiver`，返回值根据函数式最后一个返回值为准。

一般会在传入`receiver`的时候就地创建实例，不然使用`apply`来替代`with`会是更好的选择。

```kotlin
// 用给定的[receiver]作为执行块的接收者，并返回接收者的块执行结果
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

### 3.1 用例

使用`ArrayList<String>()`作为闭包的参数，返回类型为`println()`的返回值`Unit`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    println("this = " + this)
}
```

### 3.2用例

使用`ArrayList<String>()`做为闭包的参数，返回值类型为`this:ArrayList<String>`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    this
}
```

### 3.3 用例

使用`ArrayList<String>()`做为闭包的参数，返回值类型为`3.1415926`的默认类型`Double`，注意不是`Float`。

```kotlin
with(ArrayList<String>()) {
    add("a")
    add("b")
    add("c")
    3.1415926
}
```

## 四、run

执行传入的函数式，并返回函数的执行结果。`run`的主要目的是强调需要执行的函数。

```kotlin
// 执行具有指定功能的[block]，并返回执行结果
@kotlin.internal.InlineOnly
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

// 用`this`作为执行块的接收者，并返回块执行的结果
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

### 4.1 用例

从`intent`取`EXTRA_URL`的值，不为非空且内容不为空，赋值给`url`。否则弹出提示并关闭页面。

```kotlin
// Init url:String
url = intent.getStringExtra(EXTRA_URL)?.takeIf { it.isNotEmpty() } ?: run {
    toast("不能浏览一个空链接哦")
    activity.finish()
}
```

