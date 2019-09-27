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

```kotlin
override fun setRequestedOrientation(requestedOrientation: Int) {
    if (Build.VERSION.SDK_INT == Build.VERSION_CODES.O && isTranslucentOrFloating()) return
    super.setRequestedOrientation(requestedOrientation)
}
```

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



