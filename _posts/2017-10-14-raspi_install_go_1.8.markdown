---
layout:     post
title:      "RaspberryPi 安装 Go1.8"
date:       2017-10-14
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - RaspberryPi
---


树莓派使用Linux内核，想必能用来运行Go的程序。但树莓派孱弱的性能去编译Go源码不现实，所以使用官方已经编译好的二进制来安装。

在`https://golang.org/dl/`找到 `armv6l`指令集的安装包并下载

```bash
pi@raspberrypi:~ $ cd /usr/local
pi@raspberrypi:/usr $ wget https://storage.googleapis.com/golang/go1.8.4.linux-armv6l.tar.gz
```

在`/usr/local`下解压

```bash
pi@raspberrypi:/usr $ sudo tar -zxvf go1.8.4.linux-armv6l.tar.gz
```


设置环境变量

```bash
pi@raspberrypi:/etc $ sudo nano /etc/profile
```

在最后加上以下内容

```
PATH=/usr/local/go/bin:$PATH
GO_HOME=/usr/local/go
export PATH
export GO_HOME
```

保存之后

```bash
pi@raspberrypi:/usr $ source profile
```

验证安装：

```bash
pi@raspberrypi:~ $ uname -a
Linux raspberrypi 4.9.41-v7+ #1023 SMP Tue Aug 8 16:00:15 BST 2017 armv7l GNU/Linux
pi@raspberrypi:~ $ go version
go version go1.8.3 linux/arm
```

