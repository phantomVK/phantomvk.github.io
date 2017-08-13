---
layout:     post
title:      "View测量代码笔记"
date:       2017-08-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

在网上看见两种对View测量大小的方法。初步测量的数值没有问题，所以先做个笔记记录，以后深入研究。

```java
// 方法一
view.measure(0, 0);
view.getMeasuredHeight();

// 方法二
view.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
    @Override
    public void onGlobalLayout() {
        view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
        // view.getViewTreeObserver().removeOnGlobalLayoutListener(this);
        view.getHeight();
    }
});
```


