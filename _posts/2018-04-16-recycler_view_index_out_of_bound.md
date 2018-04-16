---
layout:     post
title:      "RecyclerView索引溢出异常"
date:       2018-04-16
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---



使用RecyclerView过程中遇到异常：

```
java.lang.IndexOutOfBoundsException: Inconsistency detected. Invalid view holder adapter positionViewHolder
```

继承并重写LinearLayoutManager.onLayoutChildren()方法

```kotlin
class WrappedLinearLayoutManager : LinearLayoutManager {

    constructor(context: Context) : super(context)

    constructor(context: Context, orientation: Int, reverseLayout: Boolean) : super(context, orientation, reverseLayout)

    constructor(context: Context, attrs: AttributeSet, defStyleAttr: Int, defStyleRes: Int) : super(context, attrs, defStyleAttr, defStyleRes)

    override fun onLayoutChildren(recycler: RecyclerView.Recycler?, state: RecyclerView.State) {
        try {
            super.onLayoutChildren(recycler, state)
        } catch (e: IndexOutOfBoundsException) {
            e.printStackTrace()
        }
    }
}
```

调用时使用WrappedLinearLayoutManager代替LinearLayoutManager

```kotlin
val recyclerAdapter = RecyclerViewAdapter(activity)
val manager = WrapContentLinearLayoutManager(context).apply { orientation = LinearLayoutManager.VERTICAL }

val recyclerView = view.findViewById<RecyclerView>(R.id.recycler_view).apply {
    layoutManager = manager
    adapter = recyclerAdapter
}
```

