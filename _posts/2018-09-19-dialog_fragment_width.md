---
layout:     post
title:      "Android 自定义DialogFragment宽度控制"
date:       2018-09-19
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 一、现象

DialogFragment的xml布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/shape_dialog_fragment_message_forward"
    android:orientation="vertical">

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="10dp"
        android:layout_marginEnd="15dp"
        android:layout_marginStart="15dp"
        android:layout_marginTop="15dp"
        android:text="@string/detail"
        android:textColor="#333333"
        android:textSize="17sp" />

    <android.support.v7.widget.RecyclerView
        android:id="@+id/previewList"
        android:layout_width="match_parent"
        android:layout_height="30dp"
        android:layout_marginBottom="10dp"
        android:layout_marginEnd="15dp"
        android:layout_marginStart="15dp"
        tools:listitem="@layout/item_forward_info_preview" />

    <View
        android:id="@+id/dividerUpper"
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:layout_marginEnd="15dp"
        android:layout_marginStart="15dp"
        android:background="#d9d9d9" />

    <FrameLayout
        android:id="@+id/viewContainer"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginEnd="2dp"
        android:layout_marginStart="2dp" />

    <EditText
        android:id="@+id/leavedMessage"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginEnd="15dp"
        android:layout_marginStart="15dp"
        android:background="@drawable/bg_file_space_search"
        android:gravity="center_vertical"
        android:hint="@string/leaved_msg"
        android:inputType="text"
        android:paddingEnd="8dp"
        android:paddingStart="8dp"
        android:textColor="#333333"
        android:textSize="14sp" />

    <View
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:layout_marginEnd="1dp"
        android:layout_marginStart="1dp"
        android:layout_marginTop="20dp"
        android:background="#d9d9d9" />

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="1dp"
        android:layout_marginEnd="1dp"
        android:layout_marginStart="1dp">

        <View
            android:id="@+id/hook"
            android:layout_width="1dp"
            android:layout_height="wrap_content"
            android:layout_alignBottom="@+id/btnCancel"
            android:layout_alignTop="@+id/btnCancel"
            android:layout_centerHorizontal="true"
            android:background="#d9d9d9" />

        <Button
            android:id="@+id/btnCancel"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_toStartOf="@+id/hook"
            android:background="@drawable/selector_dialog_fragment_message_button_left"
            android:text="@string/cancel"
            android:textColor="#222222"
            android:textSize="17sp" />

        <Button
            android:id="@+id/btnConfirm"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_toEndOf="@+id/hook"
            android:background="@drawable/selector_dialog_fragment_message_button_right"
            android:text="@string/confirm"
            android:textColor="#222222"
            android:textSize="17sp" />
    </RelativeLayout>
</LinearLayout>
```

DialogFragment类代码：
```java
class CustomFragment : DialogFragment() {

    val data = ArrayList<Item>().apply {
        add(Item(R.drawable.baidu, "Baidu"))
        add(Item(R.drawable.alibaba, "Alibaba"))
        add(Item(R.drawable.qq, "Tencent"))
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        super.onCreateView(inflater, container, savedInstanceState)
        return inflater.inflate(R.layout.dialog_forward, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        val adapter = ItemAdapter(context!!)
        itemsListView.layoutManager = LinearLayoutManager(context).apply { orientation = LinearLayoutManager.HORIZONTAL }
        itemsListView.adapter = adapter
        adapter.setData(data)

        val v = layoutInflater.inflate(R.layout.view_forward_text, viewContainer, true)
        v.findViewById<TextView>(R.id.text).text = "[文本] 三间中国科技公司"
    }
}
```

在不做任何处理的前提下，运行效果如下。

显然布局全部被挤在一起，没有达到 __android:layout_width="match_parent"__ 的要求

![dialog_fragment_problem](/img/android/dialogFragment/dialog_fragment_problem.png)

下面针对这个问题介绍两种处理方法。

## 二、方法一

这种方法最简单，设置一个 __style__ 。

```java
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setStyle(DialogFragment.STYLE_NO_TITLE, android.R.style.Theme_Holo_Light_Dialog_MinWidth)
}
```

设置效果如下，布局宽度会自动延伸，并在左右两侧保留一定边距。从 __DialogFragment.STYLE_NO_TITLE__ 可知样式配置为不使用标题。

![dialog_fragment_method_1](/img/android/dialogFragment/dialog_fragment_method_1.png)

## 三、方法二

通过计算决定Window布局宽度。这种方法相比方法一具备一定灵活性，可以自定义两侧保留边距。

```java
override fun onStart() {
    super.onStart()
    val dm = DisplayMetrics()
    activity?.windowManager?.defaultDisplay?.getMetrics(dm)
    dialog.window?.setLayout(((dm.widthPixels * 0.9).toInt()), ViewGroup.LayoutParams.WRAP_CONTENT)
}
```

从代码可知，布局宽度设置为屏幕总宽度90%，剩下10%宽度被均分到两侧作为边距。或者，通过设计稿对两侧保留边距的像素密度，亦可反向计算主布局所需比例。

![dialog_fragment_method_2](/img/android/dialogFragment/dialog_fragment_method_2.png)

## 四、移除标题

需要注意的是，由于方法二没有设置关于 __title__ 的参数，所以上图的布局上方出现了一块空白区，需要定义 __Window__ 的特性移除标题。

```java
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    super.onCreateView(inflater, container, savedInstanceState)
    dialog.requestWindowFeature(Window.FEATURE_NO_TITLE)
    return inflater.inflate(R.layout.dialog_forward, container, false)
}
```

设置效果图：

![dialog_fragment_notitle](/img/android/dialogFragment/dialog_fragment_notitle.png)

## 五、背景透明

由于背景默认非透明，设置了圆角后边距会有非透明的区域。通过以下代码配置，可与移除标题的代码同时使用：

```java
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    super.onCreateView(inflater, container, savedInstanceState)
    dialog.window?.setBackgroundDrawableResource(android.R.color.transparent)
    return inflater.inflate(R.layout.dialog_forward, container, false)
}
```

设置效果图：

![dialog_fragment_transparent](/img/android/dialogFragment/dialog_fragment_transparent.png)