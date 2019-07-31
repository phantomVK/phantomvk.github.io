---
layout:     post
title:      "macOS安装Java8"
date:       2019-07-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Programming Language
---

开发库如 __Gradle__ 和 __Java11__ 之间存在兼容问题需用旧版本 __Java__，但安装旧版本无法用以下命令获取

```bash
brew install java8
```

__Homebrew__ 会找不到该库而报错

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

得先获取 __cask-versions__

```bash
brew tap homebrew/cask-versions
```

从 __cask-versions__ 安装 __JDK8__

```bash
brew cask install homebrew/cask-versions/adoptopenjdk8
```
