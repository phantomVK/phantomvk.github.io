---
layout:     post
title:      "LifecycleOwner类型转换异常"
date:       2018-04-17
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---



对客户集成SDK提供技术支持的时候，客户源码工程编译没有报错，但运行过程SDK出现Fragment没法转型为LifecycleOwner异常并导致Crash，意味着运行时Fragment没有实现LifecycleOwner接口，向上转型失败了。

Activity出现ClassCastException：

![错误日志](/img/images/LifecycleOwner_CCE_0.jpg)

带Fragment的Activity出现ClassCastException：

![错误日志](/img/images/LifecycleOwner_CCE_1.png)

由于BaseFragment是自行封装的，开始以为父类RxFragment继承的Fragment类不够新，追查RxFragment的源码用app.v4.Fragment。

怀疑是不是手机系统问题，但是查了两台不同品牌、不同系统版本的手机安装客户构建的安装包都出现崩溃，这也能排除手机偶然性。

最后翻阅[Android LifecycleOwner](https://developer.android.com/topic/libraries/support-library/revisions.html#26-1-0)发现LifecycleOwner有版本要求，appSupport需要到26.1.0，SDK产品开发平台是27.0.2，一问客户的工程仅用26.0.2，开始开始完全没有意料到的是这个原因。

