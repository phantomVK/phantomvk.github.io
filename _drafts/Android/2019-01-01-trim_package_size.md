---
layout:     post
title:      "优雅地缩减APK包体积"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

为什么要优化APK包体积：

- 安装包下载体积，与安装后占用手机存储空间成正相关；

- 安装包体积较小，用户抗拒下载的心理越弱，可提高确认下载率；

- 体积较小的安装包令用户下载时更主动允许流量下载，而无需延迟到Wifi连接，提高下载成功率；

- 从用户体验上说，体积小的安装包减少下载等待时间；

- 推广会按照流量收费，安装包体积越小，推广成本越低；

- 如果交付产物是SDK，会影响宿主集成SDK后最终APK体积；

可能存在存现问题：

- 相同图片资源同时防在 __xhdpi__ 和 __xxhdpi__；
- 存放的 __jpg__ 和 __png__ 图片没有压缩；无透明通道 __png__ 没有保存为 __jpg__；存放图片尺寸过大；
- 可以考虑把所有 __jpg__ 和 __png__ 图片资源转换成 __webp__ 或矢量资源；
- 内容相同的字符串使用了不用的资源名称，造成重复声明；
- 混淆过程同时启动  __shrinkResources true__ 和 __zipAlignEnabled true__；

处理方案：

- Lint检查工程所有资源是否出现重复；
- 构建安装包时自动化转换或压缩图片资源；
- 手动更换图片为矢量资源；
- 构建时脚本合并资源名称；
- 使用 __android.enableR8.fullMode=true__
- 使用腾讯的 __AndResGuard__ 压缩安装包的资源id

#### PNG

#### JPEG和JPG

#### WebP



