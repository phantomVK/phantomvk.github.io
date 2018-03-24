---
layout:     post
title:      "pip 下载超时 镜像源配置"
date:       2016-10-28
author:     "phantomVK"
header-img: "img/main_img.jpg"
tags:
    - Python
---

国内访问Python pip官方镜像源速度非常慢，matplotlib下载速度只有10KB/s，而且下载没一会连接超时。浪费时间不止，几次下载没有一次成功过。最好的解决办法就是给pip增加国内镜像源。按照下面步骤设置好后，matplotlib下载有超百倍提升，从15KB/s提升到3.6MB/s。体积比较小的库下载速度实测满速。

先拿Mac OSX来详细说，终端进入目录：

```bash
$ cd ~/
```

如果没有.pip这个文件夹就手动新建

```bash
$ mkdir .pip
```

然后进入.pip，在文件夹内新建一个文件 

```bash
$ cd .pip
$ touch pip.conf
```

编辑 pip.conf 文件，写入阿里云镜像源

```
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

同样可以选择豆瓣镜像源

```
[global]
index-url = http://pypi.douban.com/simple

[install]
trusted-host=pypi.douban.com
```


## 其他系统pip.conf保存位置

### Linux/Unix:

```
~/.pip/pip.conf
```


### Windows:

```
%APPDATA%\pip\pip.ini
%HOME%\pip\pip.ini
C:\Documents and Settings\All Users\Application Data\PyPA\pip\pip.conf (For Windows XP)
C:\ProgramData\PyPA\pip\pip.conf (For Windows 7)
```

其他镜像源:

- 华中理工大学     http://pypi.hustunique.com/
- 山东理工大学     http://pypi.sdutlinux.org/
- 中国科学技术大学  http://pypi.mirrors.ustc.edu.cn/




