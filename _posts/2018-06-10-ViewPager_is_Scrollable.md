---
layout:     post
title:      "ViewPager禁止滚动"
date:       2018-06-10
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---



```java
// Override to catch pointerIndex out of range exception
// when ViewPager used with a multi-touch PhotoView to display images.
class StrongViewPager : ViewPager {

    // If ViewPagers can be scrolled.
    private var isScrollable = true

    constructor(context: Context) : super(context)
    constructor(context: Context, attrs: AttributeSet) : super(context, attrs)

    @SuppressLint("ClickableViewAccessibility")
    override fun onTouchEvent(ev: MotionEvent): Boolean {
        return try {
            if (isScrollable) super.onTouchEvent(ev) else false
        } catch (ignore: IllegalArgumentException) {
            false
        }
    }

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        return try {
            if (isScrollable) super.onInterceptTouchEvent(ev) else false
        } catch (e: IllegalArgumentException) {
            false
        }
    }

    // Set ViewPager if is scrollable. Default is true, means it is scrollable.
    fun setScrollable(scrollable: Boolean) {
        isScrollable = scrollable
    }

    // Get ViewPager if is scrollable.
    fun isScrollable() = isScrollable
}
```

