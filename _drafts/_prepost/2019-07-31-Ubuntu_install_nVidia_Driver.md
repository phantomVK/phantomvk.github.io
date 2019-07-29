---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

输入以下命令，然后回车确认询问的信息

```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
```

依次执行

```bash
sudo apt update
sudo apt-get update
```

```bash
ubuntu-drivers devices

== /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
modalias : pci:v000010DEd00001C07sv00001462sd00008C99bc03sc02i00
vendor   : NVIDIA Corporation
model    : GP106 [P106-100]
driver   : nvidia-driver-415 - third-party free
driver   : nvidia-driver-430 - third-party free recommended
driver   : nvidia-driver-410 - third-party free
driver   : nvidia-driver-390 - distro non-free
driver   : nvidia-driver-396 - third-party free
driver   : nvidia-driver-418 - third-party free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

```bash
sudo apt install nvidia-driver-430
```

```bash
sudo reboot
```

