---
layout:     post
title:      "魅族手机调试不打印Log日志"
date:       2017-11-07
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

平常都是用魅族MX5真机调试，有问题直接打断点看数据，不依赖控制台输出的Log。后来发现控制台应用Log只能打出 __Error__ 级别，其他级别没有任何输出。开始以为是应用Log的问题，换三星手机所有Log都能打印。最后找魅族 __开发者选项__ ，找到相关解决方案。


在`关于手机`里面多次点击下方的`版本号`开启`开发者选项`：

![关于手机](/img/android/mz_log/mz1.jpeg)

`开发者选项`在`辅助功能`中：

![辅助功能](/img/android/mz_log/mz2.jpeg)

然后拉到`开发者选项`最下面点击`性能优化`：

![性能优化](/img/android/mz_log/mz3.jpeg)

修改`高级日志输出`级别：

![高级日志输出](/img/android/mz_log/mz4.jpeg)

