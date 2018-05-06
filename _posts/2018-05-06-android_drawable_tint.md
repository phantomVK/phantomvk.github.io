---
layout:     post
title:      "Android Drawable tint方法"
date:       2018-05-06
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

### 方法

通过`PorterDuffColorFilter`和`setTint`兼容各版本系统，实测Android 4.3和Android 5.1。

```java
// Tints the drawable by color int.
fun Drawable?.tint(@ColorInt tint: Int): Drawable? {
    return this?.apply {
        if (android.os.Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
            colorFilter = PorterDuffColorFilter(tint, PorterDuff.Mode.SRC_IN)
        } else {
            setTint(tint)
        }
    }
}

// Tints the drawable by color String.
fun Drawable?.tint(@Size(min = 1) colorString: String): Drawable? {
    return tint(Color.parseColor(colorString))
}
```

### 实例代码

#### MainActivity.kt

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Origin black
        val drawable = ContextCompat.getDrawable(this, R.drawable.ic_close)?.mutate()
        image.setImageDrawable(drawable)

        // Red
        val drawableRed = ContextCompat.getDrawable(this, R.drawable.ic_close)?.mutate().tint(Color.RED)
        imageRed.setImageDrawable(drawableRed)

        // Green
        val drawableGreen = ContextCompat.getDrawable(this, R.drawable.ic_close)?.mutate().tint(Color.GREEN)
        imageGreen.setImageDrawable(drawableGreen)

        // Blue
        val drawableBlue = ContextCompat.getDrawable(this, R.drawable.ic_close)?.mutate().tint(Color.BLUE)
        imageBlue.setImageDrawable(drawableBlue)
    }
}
```

#### R.layout.activity_main

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/image"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <ImageView
        android:id="@+id/imageRed"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <ImageView
        android:id="@+id/imageGreen"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <ImageView
        android:id="@+id/imageBlue"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</LinearLayout>
```

### 运行效果

![错误日志](/img/android/android_drawable_tint.jpg)
