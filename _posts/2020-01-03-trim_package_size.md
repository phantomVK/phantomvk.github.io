---
layout:     post
title:      "优雅地缩减APK体积"
date:       2020-01-03
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

为什么要优化APK包体积：

- 安装包下载体积，与安装后占用手机存储空间成正相关；
- 安装包体积较小，用户抗拒下载的心理越弱，可提高确认下载率；
- 体积较小的安装包令用户下载时更主动允许流量下载，而无需延迟到Wifi连接；
- 从用户体验上说，小体积安装包减少下载时间；
- 推广会按照流量收费，安装包体积越小，推广成本越低；
- 如果交付产物是SDK，会影响宿主集成后APK体积；

体积大的原因：

- 手抖把同一张图片放入 __xhdpi__ 和 __xxhdpi__ (同事犯过这错)；
- 存放的 __jpg__ 和 __png__ 图片没有压缩；无透明通道 __png__ 没有保存为 __jpg__；存放图片尺寸过大；
- 可以考虑把所有 __jpg__ 和 __png__ 图片资源转换成 __webp__ 或矢量资源；
- 内容相同的字符串使用了不用的资源名称，造成重复声明；
- 混淆过程没有启动 __shrinkResources true__ 和 __zipAlignEnabled true__；

处理方案：

- Lint检查工程资源是否重复；
- 构建安装包时自动化转换或压缩图片资源；
- 手动更换图片为矢量资源；
- 构建时脚本合并资源名称；
- 使用 __android.enableR8.fullMode=true__
- 使用腾讯的 __AndResGuard__ 压缩安装包资源id

#### 智图

[智图](https://zhitu.isux.us/)是腾讯ISUX前端团队开发的一个专门用于图片压缩和图片格式转换的平台，其功能包括针对png、jpeg、gif等各类格式图片的压缩，以及为上传图片自动选择最优的图片格式。同时，智图平台还会为用户转换一份webp格式的图片。

#### PNG

是否需要使用PNG，取决于图片是否透明。需要透明的图片只能使用PNG，这种情况只能对PNG进行有损压缩减少体积。如果图片不需要透明通道但使用了PNG格式，应先把PNG转换为JPG，再对JPG进行有损压缩。

#### JPEG和JPG

对JPG直接进行压缩或考虑利用WebP算法优势进一步减少体积。

#### WebP

需要注意 Android 对 WebP 支持有版本限制。

> WebP的有损压缩算法是基于[VP8](https://zh.wikipedia.org/wiki/VP8)视频格式的[帧内编码](https://zh.wikipedia.org/wiki/幀內編碼)[[17\]](https://zh.wikipedia.org/wiki/WebP#cite_note-f.glaser-17)，并以[RIFF](https://zh.wikipedia.org/wiki/資源交換檔案格式)作为[容器格式](https://zh.wikipedia.org/wiki/视频文件格式)。[[2\]](https://zh.wikipedia.org/wiki/WebP#cite_note-Announcement-in-chromium-2) 因此，它是一个具有八位[色彩深度](https://zh.wikipedia.org/wiki/色彩深度)和以1:2的比例进行[色度子采样](https://zh.wikipedia.org/wiki/色度抽样)的[亮度-色度模型](https://zh.wikipedia.org/wiki/YUV)（[YCbCr](https://zh.wikipedia.org/wiki/YCbCr) 4:2:0）的基于块的转换方案。[[18\]](https://zh.wikipedia.org/wiki/WebP#cite_note-vp8-bitstream-18) 不含内容的情况下，RIFF容器要求只需20字节的开销，依然能保存额外的 [元数据](https://zh.wikipedia.org/wiki/元数据)(metadata)。[[2\]](https://zh.wikipedia.org/wiki/WebP#cite_note-Announcement-in-chromium-2) WebP图像的边长限制为16383像素。[[5\]](https://zh.wikipedia.org/wiki/WebP#cite_note-faq-5)

#### 其他方案

- 使用原生代码代替xml文件的声明，如：drawable、anim、string、color、layout；
- 代码实现布局：__手写组件__、__Anko__、__Android JetPack__ 等；
- 压缩 raw、assets 资源文件；
- 减少类声明、内部类、抽象接口；
- 功能插件化；
- 原生界面和 __WebView__ 混合开发，功能转移到线上；

对 Android 来说，还有比较极端的技术方案：只留下一个尺寸的图片资源，例如：__drawable-xxxhdpi__。只保留一个尺寸可有效节省空间，但是对低端手机就需要读取更大的图才能缩放到合适展示尺寸。

印象里国内有不少大厂都用这种方法，我个人也推荐这种方法。当然，为了避免每次读取文件都是用原图，这里要用 __Glide__ 等图片加载框架磁盘缓存缩放后资源。

在只进行上述图片压缩的操作后，安装包体积变化如下：

![package_size](/img/android/performance/package_size.png)

链接：

- [配置编译版本](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)
- [压缩、混淆和优化您的应用](https://developer.android.com/studio/build/shrink-code?hl=zh-CN)
- [【腾讯Bugly干货分享】WebP原理和Android支持现状介绍](https://zhuanlan.zhihu.com/p/23648251)

