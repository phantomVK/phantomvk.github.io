---
layout:     post
title:      ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

设备：__Samsung Galaxy S4__
系统：__Android 5.0.1__

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Log.i(TAG, "onCreate")
    }

    override fun onStart() {
        super.onStart()
        Log.i(TAG, "onStart")
    }

    override fun onRestoreInstanceState(savedInstanceState: Bundle?) {
        super.onRestoreInstanceState(savedInstanceState)
        Log.i(TAG, "onRestoreInstanceState")
    }

    override fun onResume() {
        super.onResume()
        Log.i(TAG, "onResume")
    }

    override fun onPause() {
        super.onPause()
        Log.i(TAG, "onPause")
    }

    override fun onSaveInstanceState(outState: Bundle?) {
        super.onSaveInstanceState(outState)
        Log.i(TAG, "onSaveInstanceState")
    }

    override fun onStop() {
        super.onStop()
        Log.i(TAG, "onStop")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.i(TAG, "onDestroy")
    }

    companion object {
        const val TAG = "MainActivity"
    }
}
```

启动进入前台

```
03-18 18:19:07.116 6213-6213/com.phantomvk.playground I/MainActivity: onCreate
03-18 18:19:07.121 6213-6213/com.phantomvk.playground I/MainActivity: onStart
03-18 18:19:07.121 6213-6213/com.phantomvk.playground I/MainActivity: onResume
```

按Home键

```
03-18 18:20:12.216 6213-6213/com.phantomvk.playground I/MainActivity: onPause
03-18 18:20:12.676 6213-6213/com.phantomvk.playground I/MainActivity: onSaveInstanceState
03-18 18:20:12.676 6213-6213/com.phantomvk.playground I/MainActivity: onStop
```

点击图标回到前台

```
03-18 18:20:51.976 6213-6213/com.phantomvk.playground I/MainActivity: onStart
03-18 18:20:51.976 6213-6213/com.phantomvk.playground I/MainActivity: onResume
```

从竖屏切换到横屏

```
03-18 18:21:43.616 6213-6213/com.phantomvk.playground I/MainActivity: onPause
03-18 18:21:43.621 6213-6213/com.phantomvk.playground I/MainActivity: onSaveInstanceState
03-18 18:21:43.621 6213-6213/com.phantomvk.playground I/MainActivity: onStop
03-18 18:21:43.621 6213-6213/com.phantomvk.playground I/MainActivity: onDestroy

03-18 18:21:43.786 6213-6213/com.phantomvk.playground I/MainActivity: onCreate
03-18 18:21:43.786 6213-6213/com.phantomvk.playground I/MainActivity: onStart
03-18 18:21:43.786 6213-6213/com.phantomvk.playground I/MainActivity: onRestoreInstanceState
03-18 18:21:43.786 6213-6213/com.phantomvk.playground I/MainActivity: onResume
```

横屏回到竖屏

```
03-18 18:22:08.761 6213-6213/com.phantomvk.playground I/MainActivity: onPause
03-18 18:22:08.761 6213-6213/com.phantomvk.playground I/MainActivity: onSaveInstanceState
03-18 18:22:08.761 6213-6213/com.phantomvk.playground I/MainActivity: onStop
03-18 18:22:08.761 6213-6213/com.phantomvk.playground I/MainActivity: onDestroy

03-18 18:22:08.876 6213-6213/com.phantomvk.playground I/MainActivity: onCreate
03-18 18:22:08.876 6213-6213/com.phantomvk.playground I/MainActivity: onStart
03-18 18:22:08.876 6213-6213/com.phantomvk.playground I/MainActivity: onRestoreInstanceState
03-18 18:22:08.876 6213-6213/com.phantomvk.playground I/MainActivity: onResume
```

前台点击返回键

```
03-18 18:23:49.211 6213-6213/com.phantomvk.playground I/MainActivity: onPause
03-18 18:23:49.831 6213-6213/com.phantomvk.playground I/MainActivity: onStop
03-18 18:23:49.831 6213-6213/com.phantomvk.playground I/MainActivity: onDestroy
```

 电源键关闭屏幕

```

```

电源键解锁回到前台

```

```

