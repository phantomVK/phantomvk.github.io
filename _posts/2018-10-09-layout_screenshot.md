---
layout:     post
title:      "Android布局截取"
date:       2018-10-09
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

布局 __activity_main.xml__

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="10dp"
    tools:ignore="ContentDescription,HardcodedText,UselessParent">

    <LinearLayout
        android:id="@+id/screenshotLayout"
        android:layout_width="310dp"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        android:layout_marginEnd="25dp"
        android:layout_marginStart="25dp"
        android:layout_marginTop="25dp"
        android:background="@drawable/bg_qr_code"
        android:orientation="vertical">

        <RelativeLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="20dp"
            android:layout_marginTop="17dp">

            <ImageView
                android:id="@+id/iv_avatar"
                android:layout_width="40dp"
                android:layout_height="40dp"
                android:layout_centerVertical="true"
                android:src="@drawable/qq" />

            <TextView
                android:id="@+id/tv_name"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginStart="10dp"
                android:layout_toEndOf="@id/iv_avatar"
                android:text="QQ"
                android:textColor="#000000"
                android:textSize="18sp" />

            <TextView
                android:id="@+id/tv_id"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignStart="@id/tv_name"
                android:layout_below="@id/tv_name"
                android:text="好友"
                android:textColor="#9b9b9b"
                android:textSize="14sp" />
        </RelativeLayout>

        <ImageView
            android:layout_width="250dp"
            android:layout_height="250dp"
            android:layout_gravity="center_horizontal"
            android:layout_marginTop="24dp"
            android:src="@drawable/generate" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_horizontal"
            android:layout_marginBottom="27dp"
            android:layout_marginTop="23dp"
            android:text="扫一扫加为好友"
            android:textColor="#6c6b6e"
            android:textSize="12sp"
            tools:ignore="HardcodedText" />
    </LinearLayout>
</FrameLayout>
```

执行逻辑

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        screenshotLayout.setOnClickListener { screenShot() }
    }

    private fun screenShot() {
        screenshotLayout.isDrawingCacheEnabled = true
        val bitmap = screenshotLayout.getDrawingCache(false)
        screenshotLayout.destroyDrawingCache()
        bitmap.toString() // Bitmap to save.
    }
}
```
