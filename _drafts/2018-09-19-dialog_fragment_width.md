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

## 现象

以下是DialogFragment的布局：

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

以下是DialogFragment代码：
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

## 方法一

```java
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setStyle(DialogFragment.STYLE_NO_TITLE, android.R.style.Theme_Holo_Light_Dialog_MinWidth)
}
```

## 方法二

```java
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    super.onCreateView(inflater, container, savedInstanceState)
    return inflater.inflate(R.layout.dialog_forward, container, false)
}

override fun onStart() {
    super.onStart()
    val dm = DisplayMetrics()
    activity?.windowManager?.defaultDisplay?.getMetrics(dm)
    dialog.window?.setLayout(((dm.widthPixels * 0.9).toInt()), ViewGroup.LayoutParams.WRAP_CONTENT)
}
```

## 去掉标签

基于方法二

```java
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    super.onCreateView(inflater, container, savedInstanceState)
    dialog.requestWindowFeature(Window.FEATURE_NO_TITLE)
    return inflater.inflate(R.layout.dialog_forward, container, false)
}
```

## 背景透明

```java
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    super.onCreateView(inflater, container, savedInstanceState)
    dialog.window?.setBackgroundDrawableResource(android.R.color.transparent)
    return inflater.inflate(R.layout.dialog_forward, container, false)
}
```

