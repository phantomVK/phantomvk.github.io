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

```java
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

```java
val recyclerAdapter = RecyclerViewAdapter(activity)
val manager = WrapContentLinearLayoutManager(context)
manager.orientation = LinearLayoutManager.VERTICAL

val recyclerView = view.findViewById<RecyclerView>(R.id.recycler_view)
recyclerView.layoutManager = manager
recyclerView.adapter = recyclerAdapter
```

注：出现这个问题的主要原因，应该是 __notifyDataSetChanged__ 后又调用 __notifyItemInserted__、__notifyItemMoved__ 等CRUD方法，造成数据的不一致。请仔细检查对数据集操作的事件流是否存在问题。