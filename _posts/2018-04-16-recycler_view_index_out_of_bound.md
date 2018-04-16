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



实际开发出现以下异常：

```
RecyclerView - java.lang.IndexOutOfBoundsException: Inconsistency detected. Invalid view holder adapter positionViewHolder
```

继承并重写LinearLayoutManager.onLayoutChildren()方法

```java
public class WrappedLinearLayoutManager extends LinearLayoutManager {

    public WrappedLinearLayoutManager(Context context) {
        super(context);
    }

    public WrappedLinearLayoutManager(Context context, int orientation, boolean reverseLayout) {
        super(context, orientation, reverseLayout);
    }

    public WrappedLinearLayoutManager(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }

    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        try {
            super.onLayoutChildren(recycler, state);
        } catch (IndexOutOfBoundsException e) {
            e.printStackTrace();
        }
    }
}
```

使用WrappedLinearLayoutManager代替LinearLayoutManager即可

```kotlin
val mAdapter = RecyclerViewAdapter(activity)
val manager = WrapContentLinearLayoutManager(context).apply { orientation = LinearLayoutManager.VERTICAL }

val mRecyclerList = view.findViewById<RecyclerView>(R.id.recycler_view).apply {
    layoutManager = manager
    adapter = mAdapter
}
```

