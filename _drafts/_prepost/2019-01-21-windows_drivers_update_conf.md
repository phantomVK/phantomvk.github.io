---
layout:     post
title:      "禁止Windows10自动安装驱动"
date:       2010-01-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Windows
---

操作步骤：

1、右键开始菜单，选择 __"运行"__ ，输入命令 __"gpedit.msc"__ 并确定；

2、弹出本地组策略编辑器，按照下图路径展开：

![Windows_update_dir](/img/Windows/Windows_update_dir.PNG)

3、进入 __"Windows更新不包括驱动程序"__ 选项。双击打开，修改为已启动并确定即可。

![Windows_update_config](/img/Windows/Windows_update_config.PNG)