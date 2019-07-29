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

开始以下步骤之前，强烈建议先参考 [Ubuntu18.04安装nVidia显卡驱动]()正确安装显卡驱动。如果显卡驱动已经安装，则下面安装 __CUDA10__ 流程可以省略，因为 __CUDA10__ 在显卡驱动安装过程会自动装好。

__Ubuntu18.04__ 同时包含 __Python2__ 和 __Python3__，但 __Python3__ 相关环境例如 __pip__ 不一定有，所以需要安装

```bash
sudo apt-get install python3-pip python3-dev python-virtualenv
```

从 [cuda-10.0-download-archive](https://developer.nvidia.com/cuda-10.0-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1804&target_type=deblocal) 获取 __CUDA10__，如果已经安装可以跳过。

![cuda_10](/img/tensorflow/cuda_10.png)

```bash
sudo dpkg -i cuda-repo-ubuntu1804-10-0-local-10.0.130-410.48_1.0-1_amd64.deb 
sudo apt-key add /var/cuda-repo-10-0-local-10.0.130-410.48/7fa2af80.pub
sudo apt-get update
sudo apt-get install cuda
```

注册账号登录到 __developer.nvidia.com__，下载 [cuDNN Runtime Library for Ubuntu18.04 (Deb)](https://developer.nvidia.com/compute/machine-learning/cudnn/secure/v7.6.1.34/prod/10.0_20190620/Ubuntu18_04-x64/libcudnn7_7.6.1.34-1%2Bcuda10.0_amd64.deb)

![cuDNN_7_6_1](/img/tensorflow/cuDNN_7_6_1.png)

安装

```bash
sudo dpkg -i libcudnn7_7.6.1.34-1+cuda10.0_amd64.deb
```

调用 __virtualenv__ 

```bash
virtualenv --system-site-packages -p python3 ~/tensorflow
```

调用脚本

```bash
source ~/tensorflow/bin/activate
(tensorflow)
```

在同一个终端下更新 __pip__

```bash
easy_install -U pip
```

下载 __tensorflow-gpu__，国内指定临时下载源： __https://pypi.tuna.tsin.edu.cn/simple__ 可以让下载速度更快

```bash
pip3 install --upgrade tensorflow-gpu -i https://pypi.tuna.tsin.edu.cn/simple
```

最后运行 TensorFlow 即可