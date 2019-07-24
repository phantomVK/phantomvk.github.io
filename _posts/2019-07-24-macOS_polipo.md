---
layout:     post
title:      "macOS使用polipo"
date:       2019-07-24
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - network
---

__macOS__ 先通过 __Homebrew__ 安装 polipo

```bash
brew services start polipo
```

在用户根目录创建或修改配置文件 __~/.polipo__。有两个参数需要修改：

- __proxyPort__：__polipo__ 提供服务的端口，默认为8123；
- __socksParentProxy__：本地 __Socks5__ 地址，具体看 __ShadowSocks__ 配置，一般为 __"127.0.0.1:1080"__；

```
proxyAddress = "0.0.0.0"
proxyPort = 8123
allowedClients = 127.0.0.1, 10.0.1.0/24
allowedPorts = 1-65535
tunnelAllowedPorts = 1-65535
proxyName = "localhost"
cacheIsShared = false
socksParentProxy = "127.0.0.1:1080"
socksProxyType = socks5
# chunkHighMark = 33554432
# diskCacheRoot = ""
# localDocumentRoot = ""
disableLocalInterface = true
disableConfiguration = true
dnsUseGethostbyname = yes
disableVia = true
censoredHeaders = from,accept-language,x-pad,link
censorReferer = maybe
# maxConnectionAge = 5m
# maxConnectionRequests = 120
# serverMaxSlots = 8
# serverSlots = 2
```

启动 __polipo__

```bash
brew services start polipo
brew services restart polipo # 重启
brew services stop polipo    # 关闭
```

如果需要引导全部终端流量，可以把配置写到bash：

```bash
export http_proxy="http://127.0.0.1:8123"
export https_proxy="http://127.0.0.1:8123"
```

如果只针对某个命令走 __polipo__，执行命令时指定proxy：

```bash
http_proxy="http://127.0.0.1:8119" curl ip.gs
https_proxy="http://127.0.0.1:8119" curl ip.gs
```

用 __curl__ 检查ip

```bash
> http_proxy="http://127.0.0.1:8119" curl ip.gs

Current IP / 当前 IP: 194.156.***.***
ISP / 运营商:  thinkhuge.net
City / 城市: Tokyo Tokyo
Country / 国家: Japan
IP.GS is now IP.SB, please visit https://ip.sb/ for more information. / IP.GS 已更改为 IP.SB ，请访问 https://ip.sb/ 获取更详细 IP 信息！
Please join Telegram group https://t.me/sbfans if you have any issues. / 如有问题，请加入 Telegram 群 https://t.me/sbfans

  /\_/\
=( °w° )=
  )   (  //
 (__ __)//
```

__https__ 导流可以直接使用 __http_proxy__

```bash
> https_proxy="http://127.0.0.1:8119" curl https://google.com

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="https://www.google.com/">here</A>.
</BODY></HTML>
```