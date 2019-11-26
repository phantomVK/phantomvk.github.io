---
layout:     post
title:      "应用启动速度提升"
date:       2019-11-25
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

1. 可以延迟的第三方库放在SplashActivity内，或使用前才初始化；
2. 第三方库在子线程初始化，使用线程池大小为2的线程；
3. 不要在Application启动过程调用网络（框架需初始化、其有线程池开销、网络访问操作开销）；
4. __Application__ 读取开关全部放在一张SharedPreferences表内；
5. 移除无效代码；
6. 避免开发版代码进入线上环境；
7. 减少需要加载的类也能降低加载时间；
8. 