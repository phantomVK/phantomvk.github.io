---
layout:     post
title:      "RecyclerView优化"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 一、数据优化

#### 1.1 缓存数据

__Adapter__ 缓存 __ViewHolder__ 公用数据

#### 1.2 本地缓存

先从本地缓存读取，同时从网络加载

#### 1.3 子线程

子线程计算数据，避免在绑定时实时计算

## 二、布局优化

#### 2.1 减少过渡绘制

#### 2.2 降低布局填充事件

原生代码、__Anko__、__Litho__

#### 2.3 减少对象生成

## 三、配置参数



#### 参考链接

- [[RecyclerView Scrolling Performance](https://stackoverflow.com/questions/27188536/recyclerview-scrolling-performance)](https://stackoverflow.com/q/27188536/8750399)
- [How to optimize Recyclerview in Android](https://mobikul.com/how-to-optimize-recyclerview-in-android/)
- [RecyclerView item optimizations](https://medium.com/@programmerr47/recyclerview-item-optimizations-cae1aed0c321)
- [Improve RecyclerView Performance](https://blog.usejournal.com/improve-recyclerview-performance-ede5cec6c5bf)
- [基本功 | Litho的使用及原理剖析](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)
- [Litho在美团动态化方案MTFlexbox中的实践](https://tech.meituan.com/2019/09/19/litho-practice-in-dynamic-program-mtflexbox.html)
- [jetpack compose - Android’s modern toolkit for building native UI](https://developer.android.com/jetpack/compose)

