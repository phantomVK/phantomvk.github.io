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

```bash
sudo apt-get install python3-pip python3-dev python-virtualenv
```

下载链接：__https://developer.nvidia.com/cuda-10.0-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1804&target_type=deblocal__

```bash
sudo dpkg -i cuda-repo-ubuntu1804-10-0-local-10.0.130-410.48_1.0-1_amd64.deb 
sudo apt-key add /var/cuda-repo-10-0-local-10.0.130-410.48/7fa2af80.pub
sudo apt-get update
sudo apt-get install cuda
```

注册账号登录到developer.nvidia.com，进入 __https://developer.nvidia.com/rdp/cudnn-archive__，[cuDNN Runtime Library for Ubuntu18.04 (Deb)](https://developer.nvidia.com/compute/machine-learning/cudnn/secure/v7.6.1.34/prod/10.0_20190620/Ubuntu18_04-x64/libcudnn7_7.6.1.34-1%2Bcuda10.0_amd64.deb)

```bash
sudo dpkg -i libcudnn7_7.6.1.34-1+cuda10.0_amd64.deb
```

```bash
virtualenv --system-site-packages -p python3 ~/tensorflow
```

```bash
source ~/tensorflow/bin/activate
(tensorflow)
```

```bash
easy_install -U pip
```

指定临时下载源： __https://pypi.tuna.tsin.edu.cn/simple__

```bash
pip3 install --upgrade tensorflow-gpu -i https://pypi.tuna.tsin.edu.cn/simple
```

检查驱动是否安装正常

```bash
nvidia-smi
Mon Jul 29 21:29:37 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 410.104      Driver Version: 410.104      CUDA Version: 10.0     |
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

最后运行执行测试检查即可