---
layout:     post
title:      "Android Framework源码索引"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android Framework
---

### 编写目标

- 阅读较新版本源码：Android_8.0.0_r44；
- 详细描述展示的源码，并附带所有类文件所属路径；
- 在文章中加入上下文索引，并通过大量流程图和示意图对源码调用进行联系；
- 源码展示过程不会出现显著跳跃，导致阅读者思维断层；
- 删除不影响阅读的注释和日志代码；



### 已知缺点

- 不会对不同版本源码进行横向对比；
- 作者水平有限，编写文章时可能会出现错误解析；



### 参考来源

- 《Android进阶解密》--刘望舒
- 《深入理解Java虚拟机 第二版》 --周志明



### 源码下载

#### 配置repo

```shell
mkdir ~/bin
PATH=~/bin:$PATH
```

```shell
curl -sSL  'https://gerrit-googlesource.proxy.ustclug.org/git-repo/+/master/repo?format=TEXT' | base64 -d > ~/bin/repo
```

```shell
chmod a+x ~/bin/repo
```

#### 下载

创建保存源码的文件夹 __android-8.0.0_r44__

```shell
mkdir android-8.0.0_r44
cd android-8.0.0_r44
```

为文件夹初始化

```shell
repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-8.0.0_r44
```

如果提示无法连接到 gerrit.googlesource.com，可以编辑 ~/bin/repo，把 REPO_URL 变量替换成以下值

```
REPO_URL = 'https://gerrit-googlesource.proxy.ustclug.org/git-repo'
```

开始同步源码

```shell
repo sync
```

### 阅读工具



### 目录索引

