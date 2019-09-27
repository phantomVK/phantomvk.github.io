---
layout:     post
title:      "创建Flutter工程卡死"
date:       2019-08-27
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Flutter
---

首次完成配置 __Flutter__ 环境后，通过 __flutter doctor__ 检查 __Android Studio__ 插件已正常安装。

```
> flutter doctor
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, v1.7.8+hotfix.4, on Mac OS X 10.14.6 18G87, locale zh-Hans-CN)

[✓] Android toolchain - develop for Android devices (Android SDK version 28.0.3)
....
....
[✓] Android Studio (version 3.5)
[!] IntelliJ IDEA Ultimate Edition (version 2019.1.3)
    ✗ Flutter plugin not installed; this adds Flutter specific functionality.
[✓] Connected device (1 available)

! Doctor found issues in 4 categories.
```

但是用 __Android Studio__ 的 __New Flutter project__ 时IDE卡死，因为 __Android Studio__ 配置插件后不会默认导入 __Flutter__ 的环境变量。首先进入 __Android Studio__ 设置：

![preferences_macox](/img/flutter/preferences_macox.png)

搜索关键字 __flutter__，然后手动指定本地的 __Flutter SDK path__：

![preferences_flutter_macox](/img/flutter/preferences_flutter_macox.png)

设置完毕后重启IDE，在尝试创建工程。如果依然卡死，请在环境变量内配置下载 __Flutter__ 的中国依赖源。

![new_flutter_project_page](/img/flutter/new_flutter_project_page.png)