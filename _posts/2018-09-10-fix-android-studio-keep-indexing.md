---
layout:     post
title:      "Android Studio不断indexing"
date:       2018-09-10
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

自从 __Android Studio 3.0__ 升级到 __Android Studio 3.1.3__ 后，每次打开工程持续不断indexing，一秒钟会indexing好几次，且一直持续不终止。这个问题直接影响到开发速度：因为indexing过程中，编辑代码的代码提示不出现，自动代码检查运行不正常，更不说编译运行。

开始还以为是 __Android Studio 3.1.3__ 存在bug，但是升级到 __Android Studio 3.1.4__ 后问题依旧。

最后通过以下方法解决，下面是MacOS的处理流程，Windows类似：

__"File" -> "Invalidate Caches / Restart..." -> 点击 "Invalidate and Restart" 确认__

IDE重启之后等重新索引完成即可。