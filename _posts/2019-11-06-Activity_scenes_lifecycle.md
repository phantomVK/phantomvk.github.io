---
layout:     post
title:      "基本生命周期"
date:       2019-11-06
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

设备：__Samsung Galaxy S4__
系统：__Android 5.0.1__

```java
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Log.i("MainActivity", "onCreate")
    }

    override fun onStart() {
        super.onStart()
        Log.i("MainActivity", "onStart")
    }

    override fun onRestoreInstanceState(savedInstanceState: Bundle?) {
        super.onRestoreInstanceState(savedInstanceState)
        Log.i("MainActivity", "onRestoreInstanceState")
    }

    override fun onResume() {
        super.onResume()
        Log.i("MainActivity", "onResume")
    }

    override fun onPause() {
        super.onPause()
        Log.i("MainActivity", "onPause")
    }

    override fun onSaveInstanceState(outState: Bundle?) {
        super.onSaveInstanceState(outState)
        Log.i("MainActivity", "onSaveInstanceState")
    }

    override fun onStop() {
        super.onStop()
        Log.i("MainActivity", "onStop")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.i("MainActivity", "onDestroy")
    }
}
```

启动进入前台

```
com.phantomvk.playground I/MainActivity: onCreate
com.phantomvk.playground I/MainActivity: onStart
com.phantomvk.playground I/MainActivity: onResume
```

按Home键

```
com.phantomvk.playground I/MainActivity: onPause
com.phantomvk.playground I/MainActivity: onSaveInstanceState
com.phantomvk.playground I/MainActivity: onStop
```

点击图标回到前台

```
com.phantomvk.playground I/MainActivity: onStart
com.phantomvk.playground I/MainActivity: onResume
```

竖屏切换到横屏、横屏回到竖屏

```
com.phantomvk.playground I/MainActivity: onPause
com.phantomvk.playground I/MainActivity: onSaveInstanceState
com.phantomvk.playground I/MainActivity: onStop
com.phantomvk.playground I/MainActivity: onDestroy

com.phantomvk.playground I/MainActivity: onCreate
com.phantomvk.playground I/MainActivity: onStart
com.phantomvk.playground I/MainActivity: onRestoreInstanceState
com.phantomvk.playground I/MainActivity: onResume
```

前台点击返回键

```
com.phantomvk.playground I/MainActivity: onPause
com.phantomvk.playground I/MainActivity: onStop
com.phantomvk.playground I/MainActivity: onDestroy
```

电源键关闭屏幕

```
com.phantomvk.playground I/MainActivity: onPause
com.phantomvk.playground I/MainActivity: onSaveInstanceState
com.phantomvk.playground I/MainActivity: onStop
```

电源键解锁回到前台

```
com.phantomvk.playground I/MainActivity: onStart
com.phantomvk.playground I/MainActivity: onResume
```

