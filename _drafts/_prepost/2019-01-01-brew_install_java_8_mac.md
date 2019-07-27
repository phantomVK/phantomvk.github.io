---
layout:     post
title:      "Homwbrew安装Java8"
date:       2019-07-27
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

部分开发库例如 __Gradle__ 和最新版本 __Java11__ 存在兼容性问题，需要安装旧版本。但安装旧版本不能通过以下命令获取：

```basg
brew install java8
```

报错：

```bash
Error: No available formula with the name "java8"
==> Searching for a previously deleted formula (in the last month)...
Error: No previously deleted formula found.
==> Searching for similarly named formulae...
Error: No similarly named formulae found.
==> Searching taps...
==> Searching taps on GitHub...
Error: No formulae found in taps.
```

先获取 __cask-versions__

```bash
brew tap homebrew/cask-versions
```

从 __cask-versions__ 安装 __JDK8__ 即可，安装完成本地调整环境变量即可

```bash
brew cask install homebrew/cask-versions/adoptopenjdk8
```
