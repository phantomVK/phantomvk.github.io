---
layout:     post
title:      "Linux挂载exFat分区"
date:       2017-12-03
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Linux
---

先安装`exfat-fuse`和`exfat-utils`：

```bash
sudo apt-get install exfat-fuse exfat-utils
```

安装完之后直接插入`exFat`的设备，`exFat`分区就会像`NTFS`或其他支持的类型一样自动加载。用完后卸载或弹出就可以。


