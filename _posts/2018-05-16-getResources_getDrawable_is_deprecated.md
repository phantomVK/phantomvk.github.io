---
layout:     post
title:      "getResources().getDrawable() is deprecated"
date:       2018-05-16
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

### 方案A (推荐)：

获取带主题属性的drawable，主题配置来自当前Activity:

```java
ContextCompat.getDrawable(getActivity(), R.drawable.name);
```

### 方案B：

获取不带主题属性的drawable:

```java
ResourcesCompat.getDrawable(getResources(), R.drawable.name, null);
```

### 方案C：

获取带自定义主题属性的drawable:

```java
ResourcesCompat.getDrawable(getResources(), R.drawable.name, anotherTheme);
```

