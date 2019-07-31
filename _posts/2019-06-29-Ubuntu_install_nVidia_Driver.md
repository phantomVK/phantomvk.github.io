---
layout:     post
title:      "Ubuntu18.04安装NVIDIA显卡驱动"
date:       2019-06-29
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Linux
---

#### 开始安装

添加PPA，过程中回车确认询问的信息

```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
```

依次执行以下命令更新下载源

```bash
sudo apt update
```

检查可用驱动

```bash
ubuntu-drivers devices
```

根据下列结果，可见 __nvidia-driver-430__ 为推荐驱动安装版本

```bash
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

直接安装最新驱动，安装完成后重启电脑

```bash
sudo apt install nvidia-driver-430
```

重启后用命令 __nvidia-smi__ 检查显卡是否被正常识别：显卡P106-100，显存6G，驱动430.26，CUDA10.2

```bash
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 430.26       Driver Version: 430.26       CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  P106-100            Off  | 00000000:01:00.0 Off |                  N/A |
| 39%   40C    P0    27W / 120W |      0MiB /  6080MiB |      4%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

#### 错误处理

如果安装 __nvidia-driver-410__ 或以上版本提示 __packages__ 无法安装，请执行以下步骤：

移除已添加的 __PPA__

```bash
sudo apt-add-repository -r ppa:graphics-drivers/ppa
```

更新 __apt__

```bash
sudo apt update
```

移除 __NVIDIA__ 显卡驱动文件

```bash
sudo apt remove nvidia*
```

执行自动清理

```bash
sudo apt autoremove
```

然后重新回到本文初步骤重新安装

#### 参考链接

[Unable to install nvidia drivers on Ubuntu 18.04](https://askubuntu.com/questions/1077493/unable-to-install-nvidia-drivers-on-ubuntu-18-04)

[Unable to correct problems, you have held broken packages](https://askubuntu.com/questions/223237/unable-to-correct-problems-you-have-held-broken-packages)