---
layout:     post
title:      "Android中Dalvik和ART的区别"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

__ART__ 在运行时的优点：

> 垃圾回收 (GC) 可能有损于应用性能，从而导致显示不稳定、界面响应速度缓慢以及其他问题。ART 通过以下几种方式对垃圾回收做了优化：
>
> - 只有一次（而非两次）GC 暂停
> - 在 GC 保持暂停状态期间并行处理
> - 在清理最近分配的短时对象这种特殊情况中，回收器的总 GC 时间更短
> - 优化了垃圾回收的工效，能够更加及时地进行并行垃圾回收，这使得 [`GC_FOR_ALLOC`](http://developer.android.com/tools/debugging/debugging-memory.html#LogMessages) 事件在典型用例中极为罕见
> - 压缩 GC 以减少后台内存使用和碎片

#### 推荐阅读

- [ART 下的方法内联策略及其对 Android 热修复方案的影响分析](https://cloud.tencent.com/developer/article/1005604)

- [微信Android热补丁实践演进之路](https://cloud.tencent.com/developer/article/1004335)

- [Android Runtime (ART) 和 Dalvik](https://source.android.com/devices/tech/dalvik)