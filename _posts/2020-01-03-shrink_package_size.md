---
layout:     post
title:      "优雅地缩减APK体积"
date:       2020-01-03
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Optimization
---

#### 为何优化体积

- 安装包体积越小，则用户抗拒下载心理越弱，可提高确认下载率；
- 小安装包让用户下载时更主动允许流量下载，而无需延迟到Wifi；
- 从用户体验层面来说，小体积安装包减少下载时间，同时减少安装耗时；
- 安装包大小与安装后占用存储空间成正相关；
- 推广按照流量收费，安装包体积越小，推广成本越低；
- 如果交付产物是SDK，更会影响宿主APK体积；

#### 原因

- 手抖把同一张图片同时放入 __xhdpi__ 和 __xxhdpi__ (同事犯过这错)；
- 资源图片没有压缩；不需要透明通道的 __png__ 没有保存为 __jpg__；
- 存放图片实际尺寸过大；
- 相同字符串使用不同资源名称造成重复声明；
- 废弃、冗余代码没有及时清理，依然被其他代码引用令 __Proguard__ 无法自动裁剪；

#### 基础方案

- 用 __Android Lint__ 检查开发资源是否重复；
- 构建安装包时用脚本自动化压缩图片资源；
- 构建时脚本合并资源名称；
- 把所有 __jpg__ 和 __png__ 图片资源转换成 __webp__ 或矢量资源；
- 启用 __android.enableR8.fullMode=true__；
- 启动 __shrinkResources true__ 和 __zipAlignEnabled true__；
- 使用腾讯的 __AndResGuard__ 压缩安装包资源id名称；

#### 智图

[智图](https://zhitu.isux.us/)是腾讯ISUX前端团队开发的一个专门用于图片压缩和图片格式转换的平台，其功能包括针对png、jpeg、gif等各类格式图片的压缩，以及为上传图片自动选择最优的图片格式。同时，智图平台还会为用户转换一份webp格式的图片。

#### PNG

是否需要使用 __PNG__ 取决于应用场景的图片是否需要透明。透明通道的图片需使用 __PNG__ (或 __WebP__)，这种情况只能对图片进行有损压缩减少体积。如果图片不需要透明通道但使用了 __PNG__ 格式，应先把 __PNG__ 转换为 __JPG__，再对 __JPG__ 进行有损压缩。

#### JPEG和JPG

对 __JPG__ 直接进行压缩或考虑利用 __WebP__ 算法优势进一步减少体积。

#### WebP

需要注意 __Android__ 对 __WebP__ 支持有版本限制，和 __PNG__ 一样支持透明通道。

> WebP的有损压缩算法是基于[VP8](https://zh.wikipedia.org/wiki/VP8)视频格式的[帧内编码](https://zh.wikipedia.org/wiki/幀內編碼)[[17\]](https://zh.wikipedia.org/wiki/WebP#cite_note-f.glaser-17)，并以[RIFF](https://zh.wikipedia.org/wiki/資源交換檔案格式)作为[容器格式](https://zh.wikipedia.org/wiki/视频文件格式)。[[2\]](https://zh.wikipedia.org/wiki/WebP#cite_note-Announcement-in-chromium-2) 因此，它是一个具有八位[色彩深度](https://zh.wikipedia.org/wiki/色彩深度)和以1:2的比例进行[色度子采样](https://zh.wikipedia.org/wiki/色度抽样)的[亮度-色度模型](https://zh.wikipedia.org/wiki/YUV)（[YCbCr](https://zh.wikipedia.org/wiki/YCbCr) 4:2:0）的基于块的转换方案。[[18\]](https://zh.wikipedia.org/wiki/WebP#cite_note-vp8-bitstream-18) 不含内容的情况下，RIFF容器要求只需20字节的开销，依然能保存额外的 [元数据](https://zh.wikipedia.org/wiki/元数据)(metadata)。[[2\]](https://zh.wikipedia.org/wiki/WebP#cite_note-Announcement-in-chromium-2) WebP图像的边长限制为16383像素。[[5\]](https://zh.wikipedia.org/wiki/WebP#cite_note-faq-5)

#### 其他方案

- 使用原生代码代替文件声明，如：__drawable__、__anim__、__string__、__color__、__layout__；
- 图片资源迁移到云端，运行时通过图片加载框架加载资源；
- __手写组件__、__Anko__、__Android JetPack__ 等代码实现UI布局；
- 手动压缩 __raw__、__assets__ 文件夹的图片、音频、JS文件，这些文件构建时默认不会压缩；
- 减少类声明、内部类、抽象接口；
- 使用频率低的功能在使用时下载插件再启动；
- 原生界面和 __WebView__ 混合开发，功能转移到线上；
- 矢量背景、图形用自定义 __View__ 在 __Canvas__ 绘制；

对 Android 来说，还有比较极端的技术方案：只留下一个尺寸的图片资源，例如：__drawable-xxxhdpi__。只保留一个尺寸可有效节省空间，但是对低端手机就需要读取更大的图才能缩放到合适展示尺寸。

印象里国内有不少大厂都用这种方法，我个人也推荐这种方法。为了避免每次读取文件都用原图，要用 __Glide__ 等图片加载框架的磁盘策略，缓存缩放后的资源。

#### 效果

对工程图片进行压缩、部分非重要图标只保留一份资源的处理，结果如下：

![package_size](/img/android/performance/package_size.png)

后期再把数张 __PNG__ 海报转换为 __JPG__，体积更是减少到19.6MB，累计压缩20%体积。

#### 参考链接

- [配置编译版本](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)
- [压缩、混淆和优化您的应用](https://developer.android.com/studio/build/shrink-code?hl=zh-CN)
- [\[腾讯Bugly干货分享\]WebP原理和Android支持现状介绍](https://zhuanlan.zhihu.com/p/23648251)

