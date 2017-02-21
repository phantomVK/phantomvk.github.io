---
layout:     post
title:      "Ubuntu 16.04安装Oracle JDK8"
date:       2016-11-23
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java
---

## 简述

虽然在Ubuntu上安装Oracle JDK不是什么难事，但是偶尔配置一次就到处找命令相当令人讨厌。所以这里主要用来记录需要的命令，做个备忘录。

我自己试过在JDK官网下载二进制编译包，也试过apt-get的方式。因为我个人比较懒，而且使用的电脑已经联网，就直接使用apt-get的方式，相比前者方便不少。

## 安装Oracle JDK8

```bash
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```

## 设置环境变量

```bash
$ sudo apt-get install oracle-java8-set-default
```

## 检查是否成功

```bash
$ java -version
java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)
```





