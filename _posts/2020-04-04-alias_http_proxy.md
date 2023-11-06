---
layout:     post
title:      "Linux终端走本地代理端口"
date:       2020-04-04
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Linux
---

```
alias HTTP_PROXY=" export  http_proxy=http://localhost:1087"
alias HTTPS_PROXY="export https_proxy=http://localhost:1087"
alias UN_HTTP_PROXY="unset http_proxy"
alias UN_HTTPS_PROXY="unset https_proxy"
```
