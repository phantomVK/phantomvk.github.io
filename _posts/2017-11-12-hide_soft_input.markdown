---
layout:     post
title:      "Android点击空白区域隐藏键盘"
date:       2017-11-12
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

重写`dispatchTouchEvent`拦截点击事件，如果点击的区域不是`EditText`则隐藏键盘。

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }

    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        if (ev.action == MotionEvent.ACTION_DOWN && shouldHideInput(currentFocus, ev)) {
            hideSoftInput(currentFocus.windowToken)
        }
        return super.dispatchTouchEvent(ev)
    }

    // Check if click on the EditText, return true to hide soft input.
    private fun shouldHideInput(view: View?, event: MotionEvent): Boolean {
        if (view != null && view is EditText) {
            val array = intArrayOf(0, 0)
            view.getLocationInWindow(array)
            val left = array[0]
            val top = array[1]
            val bottom = top + view.getHeight()
            val right = left + view.getWidth()
            return event.x <= left || event.x >= right || event.y <= top || event.y >= bottom
        }
        return false
    }

    // Hide system soft input.
    private fun hideSoftInput(token: IBinder?) {
        token?.let {
            val im = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
            im.hideSoftInputFromWindow(token, InputMethodManager.HIDE_NOT_ALWAYS)
        }
    }
}
```


