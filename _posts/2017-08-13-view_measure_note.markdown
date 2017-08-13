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

```java
view.measure(0, 0);
view.getMeasuredHeight();

view.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
    @Override
    public void onGlobalLayout() {
        view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
        // view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
        view.getHeight();
    }
});
```


