---
layout:     post
title:      "Fix: Only fullscreen activities can request orientation"
date:       2019-09-27
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

根据运行系统版本号重写 __Activity.setRequestedOrientation(Int)__。当版本号为 __Android O__ 时不调用父类方法，避免父类检测条件并抛出异常。

```kotlin
override fun setRequestedOrientation(requestedOrientation: Int) {
    if (Build.VERSION.SDK_INT == Build.VERSION_CODES.O && isTranslucentOrFloating()) return
    super.setRequestedOrientation(requestedOrientation)
}
```

根据文章 [Only fullscreen opaque activities can request orientation](/2019/06/26/fullscreen_orientation/) 编写过滤条件。

```kotlin
private fun isTranslucentOrFloating(): Boolean {
    var result = false
    try {
        val r = Class.forName("com.android.internal.R\$styleable")
            .getField("Window").get(null) as IntArray
        val ta = obtainStyledAttributes(r)
        val m = ActivityInfo::class.java
            .getMethod("isTranslucentOrFloating", TypedArray::class.java)
        m.isAccessible = true
        result = m.invoke(null, ta) as Boolean
        m.isAccessible = false
    } catch (ignore: Exception) {
    }
    return result
}
```
