---
layout:     post
title:      "Dart配置运行"
date:       2019-01-28
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Dart
---

#### 简介

相比 __Java__、__Kotlin__、__Go__ 或 __JavaScript__ 预览，__Dart__ 更小众也更冷门。而人们对 __Dart__ 的关注度逐渐提升，主要是 __Flutter__ 的主要变成语言为 __Dart__。__Dart__ 和 __Go__ 均为谷歌的亲儿子，我个人短时间所产生的熟悉感，主要是来自 __Dart__ 的语法和 __Kotlin__ 高度相像。__Dart__ 和 __Kotlin__ 均为面向对象的语言，基于单继承的类进行开发，都是强类型语言且支持自动类型推断。

现时 __Dart__ 可参阅的资料并不多，所以参考官方文档能保证内容的正确性。此外，也其他了一些其他不错的参考资料：

- [A Tour of the Dart Language](https://www.dartlang.org/guides/language/language-tour)
- [Dart  - wikipedia](https://zh.wikipedia.org/wiki/Dart)
- [Dartlang译文：Dart快速入门](https://jarontai.github.io/blog/2014/12/18/translation-dart-quick-start/)

和学习一般的语言的方式一样，初期学习 __Dart__ 只需要学会如何运行工程，并了解基本语法。以后可以在学习 __Flutter__ 时一并提升 __Dart__ 的水平。

#### 安装Dart

如果电脑已安装 __Homebrew__ ，通过以下代码就能安装 __Dart__ 环境

```shell
$ brew tap dart-lang/dart
$ brew install dart
```

安装完成的 __Dart__ 在 MacOS 中保存在`/usr/local/opt/dart/libexec`

#### IDE配置

使用 __Idea Intellij__ 需要配置 __Dart__ 语言的插件，其他IDE需要配置各自的支持插件。到 __Preferences__ - __Plugins__ 界面，选择下方的  __Install JetBrains plugins__

![dart_plugin](/img/dart/dart_plugin.png)

搜索 __Dart__ 并安装对应插件，安装成功后重启IDE。由于本机应安装插件，所以下图按钮显示 __installed__

![dart_plugin_install](/img/dart/dart_plugin_install.png)

#### 运行

在IDE中创建新工程

![dart_idea_new](/img/dart/dart_idea_new.png)

左边的类型选择 __Dart__ 并进入下一步，下一步里面填写工程名称

![dart_new_project](/img/dart/dart_new_project.png)

运行前记得配置工程的主文件，就能显示 Hello world.

![dart_hello_world](/img/dart/dart_hello_world.png)