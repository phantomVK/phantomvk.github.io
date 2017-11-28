---
layout:     post
title:      "ToolBar修改标题截断位置"
date:       2017-11-28
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

下面通过反射`ToolBar`中`TextView`的`setEllipsize`方法，设置标题中间为截断位置

```java
    private void setEllipsize(Toolbar toolBar) {
        try {
            Field field = Toolbar.class.getDeclaredField("mTitleTextView");
            field.setAccessible(true);
            TextView textView = (TextView) field.get(toolBar);
            textView.setEllipsize(TextUtils.TruncateAt.MIDDLE);
        } catch (Exception ignored) {
            Log.e(LOG_TAG, ignored.getMessage());
        }
    }
```


