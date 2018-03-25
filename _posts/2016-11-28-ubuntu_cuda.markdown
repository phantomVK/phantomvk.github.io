---
layout:     post
title:      "深度学习环境配置: Ubuntu16.04 & CUDA8.0"
date:       2016-11-28
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Tools
---

深度学习系统用Ubuntu 16.04，显卡是一张nVidia GTS450，主要用来做简单实验。用比较新的nVidia显卡驱动和CUDA8。

__请看完文章再尝试哦__

# 添加下载源

```bash
$ cd /etc/apt/
$ sudo cp sources.list sources.list.bak
$ sudo vi sources.list 
```

把源添加到文件前面，保存

```bash
deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse 
```

更新一下

```bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

添加显卡驱动的PPA

```bash
$ sudo add-apt-repository ppa:graphics-drivers/ppa
```

然后显示下列信息

```shell
Fresh drivers from upstream, currently shipping Nvidia.

## Current Status

Current official release: `nvidia-370` (370.28)
Current long-lived branch release: `nvidia-367` (367.57)

For GeForce 8 and 9 series GPUs use `nvidia-340` (340.98)
For GeForce 6 and 7 series GPUs use `nvidia-304` (304.132)

## What we're working on right now:

- Normal driver updates
- Help Wanted: Mesa Updates for Intel/AMD users, ping us if you want to help do this work, we're shorthanded.

## WARNINGS:

This PPA is currently in testing, you should be experienced with packaging before you dive in here:

Volunteers welcome! See also: https://github.com/mamarley/nvidia-graphics-drivers/

### How you can help:

## Install PTS and benchmark your gear:

    sudo apt-get install phoronix-test-suite

Run the benchmark:

    phoronix-test-suite default-benchmark openarena xonotic tesseract gputest unigine-valley

and then say yes when it asks you to submit your results to openbechmarking.org. Then grab a cup of coffee, it takes a bit for the benchmarks to run. Depending on the version of Ubuntu you're using it might preferable for you to grabs PTS from upstream directly: http://www.phoronix-test-suite.com/?k=downloads

## Share your results with the community:

Post a link to your results (or any other feedback to): https://launchpad.net/~graphics-drivers-testers

Remember to rerun and resubmit the benchmarks after driver upgrades, this will allow us to gather a bunch of data on performance that we can share with everybody.

If you run into old documentation referring to other PPAs, you can help us by consolidating references to this PPA.

If someone wants to go ahead and start prototyping on `software-properties-gtk` on what the GUI should look like, please start hacking!

## Help us Help You!

We use the donation funds to get the developers hardware to test and upload these drivers, please consider donating to the "community" slider on the donation page if you're loving this PPA:

http://www.ubuntu.com/download/desktop/contribute
 More info: https://launchpad.net/~graphics-drivers/+archive/ubuntu/ppa
Press [ENTER] to continue or ctrl-c to cancel adding it
```

在这里回车确认

```shell
gpg: keyring `/tmp/tmp_4l44wuj/secring.gpg' created
gpg: keyring `/tmp/tmp_4l44wuj/pubring.gpg' created
gpg: requesting key 1118213C from hkp server keyserver.ubuntu.com
gpg: /tmp/tmp_4l44wuj/trustdb.gpg: trustdb created
gpg: key 1118213C: public key "Launchpad PPA for Graphics Drivers Team" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
OK
```

下面就可以开始安装显卡驱动了，下载速度有点慢。

```bash
$ sudo apt-get update
$ sudo apt-get install nvidia-367
$ sudo apt-get install mesa-common-dev
$ sudo apt-get install freeglut3-dev 
```

驱动安装完成一定要重启系统


# 下载并安装CUDA


进入[CUDA官网](https://developer.nvidia.com/cuda-release-candidate-download)，注册并登陆账号，按照下图配置下载Ubuntu的安装文件，文件约1.4GB。

![img](/img/cuda.png)

下载完成进入下载目录，创建一个临时文件，使用命令解压到临时文件并安装。

```bash
$ sudo mkdir /opt/temp
$ sudo sh cuda_8.0.44_linux.run --tmpdir=/opt/temp/ 
```


安装协议可以使用`q`跳过，第一个问题询问是否安装驱动。因为我们已经安装驱动，所以选择不安装。

```bash
....
....

Do you accept the previously read EULA?
accept/decline/quit:accept

Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 367.48?
(y)es/(n)o/(q)uit: n
```

问题选择`y`，询问路径回车默认设置。

```bash
Install the CUDA 8.0 Toolkit?
(y)es/(n)o/(q)uit: y

Enter Toolkit Location
 [ default is /usr/local/cuda-8.0 ]: y

Toolkit location must be an absolute path.
Enter Toolkit Location
 [ default is /usr/local/cuda-8.0 ]: 

Do you want to install a symbolic link at /usr/local/cuda?
(y)es/(n)o/(q)uit: y

Install the CUDA 8.0 Samples?
(y)es/(n)o/(q)uit: y

Enter CUDA Samples Location
 [ default is /home/mike ]: 

Installing the CUDA Toolkit in /usr/local/cuda-8.0 ...
```

这里根据不同系统会有2个或4个文件缺失，下面显示缺了两个文件。

```
Missing recommended library: libXi.so
Missing recommended library: libXmu.so
```

使用apt补全文件，参考自 [stackoverflow](http://stackoverflow.com/questions/22360771/missing-recommended-library-libglu-so)。

```bash
$ sudo apt-get install nvidia-modprobe freeglut3-dev libx11-dev libxmu-dev libxi-dev libglu1-mesa-dev
```

安装完成要`重启系统`再重新安装CUDA包，正常的话就不会显示文件缺失，继续安装成功。

```bash
Installing the CUDA Samples in /home/textminer ...
Installing the CUDA Samples in /home/mike ...
Copying samples to /home/mike/NVIDIA_CUDA-8.0_Samples now...
Finished copying samples.

===========
= Summary =
===========

Driver:   Not Selected
Toolkit:  Installed in /usr/local/cuda-8.0
Samples:  Installed in /home/mike, but missing recommended libraries

Please make sure that
 -   PATH includes /usr/local/cuda-8.0/bin
 -   LD_LIBRARY_PATH includes /usr/local/cuda-8.0/lib64, or, add /usr/local/cuda-8.0/lib64 to /etc/ld.so.conf and run ldconfig as root

To uninstall the CUDA Toolkit, run the uninstall script in /usr/local/cuda-8.0/bin

Please see CUDA_Installation_Guide_Linux.pdf in /usr/local/cuda-8.0/doc/pdf for detailed information on setting up CUDA.

***WARNING: Incomplete installation! This installation did not install the CUDA Driver. A driver of version at least 361.00 is required for CUDA 8.0 functionality to work.
To install the driver using this installer, run the following command, replacing <CudaInstaller> with the name of this run file:
    sudo <CudaInstaller>.run -silent -driver

Logfile is /opt/temp//cuda_install_2299.log
```


安装完毕后声明环境变量，并将其写入`~/.bashrc`最后

```
export PATH=/usr/local/cuda-8.0/bin${PATH:+:${PATH}}export LD_LIBRARY_PATH=/usr/local/cuda-8.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}} 
```


# 测试

使用`nvidia-smi`检查显卡状况。

```bash
mike@mike-P5Q-PRO:~/NVIDIA_CUDA-8.0_Samples$ nvidia-smi
Fri Nov 25 17:07:01 2016
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 367.57                 Driver Version: 367.57                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTS 450     Off  | 0000:01:00.0     N/A |                  N/A |
| 40%   35C   P12    N/A /  N/A |    225MiB /   962MiB |     N/A      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|    0                  Not Supported                                         |
+-----------------------------------------------------------------------------+
```

测试显卡带宽，使用之前先把文件用`make`编译

```bash
mike@mike-P5Q-PRO:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/bandwidthTest$ ./bandwidthTest 
[CUDA Bandwidth Test] - Starting...
Running on...

 Device 0: GeForce GTS 450
 Quick Mode

 Host to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(MB/s)
   33554432			5390.0

 Device to Host Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(MB/s)
   33554432			5935.1

 Device to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(MB/s)
   33554432			43900.2

Result = PASS

NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.
```


