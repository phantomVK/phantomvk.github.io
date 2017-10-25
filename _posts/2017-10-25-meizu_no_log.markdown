---
layout:     post
title:      "魅族手机调试不打印Log日志"
date:       2017-10-25
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

平常都是用自己的魅族MX5真机调试的，不过有问题都是直接打断点看数据，不太依赖控制台输出的Log。后来发现怎么控制台里面关于应用的Log只能打出 __Error__ 级别的，其他的统统没有。

开始还以为是应用Log的问题，换了台三星，所有Log一应俱全。然后就找魅族 __开发者选项__ ，终于找到了解决方案。


在`关于手机`里面，多次点击下方的`版本号`，开启`开发者选项`

![关于手机](/img/mz_log/mz1.ipeg)

`开发者选项`在`辅助功能`中

![辅助功能](/img/mz_log/mz2.ipeg)

然后拉到`开发者选项`最下面点击`性能优化`

![性能优化](/img/mz_log/mz3.ipeg)

修改`高级日志输出`的级别即可

![高级日志输出](/img/mz_log/mz4.ipeg)

