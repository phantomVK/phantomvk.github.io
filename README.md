# Jekyll

Jekyll主题来自[Hux Blog](https://github.com/Huxpro/huxpro.github.io)(Powered by Jekyll with Hux Theme)，修改源码并增加新功能。

## 使用方法

### Fork or Clone

Fork工程到Github仓库或Clone到本地：

```bash
$ git clone https://github.com/phantomVK/phantomvk.github.io.git
```

项目文件夹名修改形如`GithubUsername.github.io`。

### 修改配置参数

打开配置文件`_config.yml`，按照以下示例把`Jekyll_Templet`改为`Github username`。

```
# Site settings
title: Jekyll_Templet
SEOTitle: Jekyll_Templet
header-img: img/home.jpg
email: Jekyll_Templet@gmail.com
description: ""
keyword: ""
#url: "http://"       # your host, for absolute URL
#baseurl: "/"         # for example, '/blog' if your blog hosted on 'host/blog'
```

### 文章编写

`_draft`存有文章模板`2016-10-01-demo.markdown`：

```
---
layout:     post
title:      ""
subtitle:   ""
date:       2016-10-01
author:     "Jekyll_Templet"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - tags
---
```

### 侧边栏

#### 头像和描述

用于配置是否打开侧边栏、自我描述和头像功能。

```
# Sidebar settings
sidebar: true                        # whether or not using Sidebar.
sidebar-about-description: ""
sidebar-avatar: /img/avatar.jpg      # use absolute URL, seeing it's used in both `/` and `/about/`
```


#### 打赏二维码

侧边栏添加二维码功能，如下设置支付二维码图片路径，图片分辨率建议`275*275`，格式`jpg`。

```
sidebar-wechat-pay: /img/wechat.jpg
sidebar-alipay-pay: /img/alipay.jpg
```

设置是否显示侧边栏二维码，可单独打开微信支付或支付宝支付。

```
# Pay Tags
pay-tags: false   # false为关闭，true为打开
pay-tags-wechat: false
pay-tags-alipay: false
```


#### 友情链接

全部注释可关闭友情链接功能，已知关闭此功能将导致文章阅读区域滚动卡顿。

```
# Friends
friends: [
    {
        title: "Foo Blog",
        href: "http://foo.github.io/"
    },
    {
        title: "Bar Blog",
        href: "http://bar.github.io"
    }
]
```


#### 链接

仅需输入链接对应用户名：

```
# SNS settings
RSS: false
#zhihu_username: username
#github_username:
#weibo_username:     
#twitter_username:   
#facebook_username:  
#linkedin_username:  firstname-lastname-idxxxx
```

### 文章归档

增加文章归档功能，自动按照月份归档文章，具体展示请参考[phantomVK - Archive](https://phantomvk.github.io/archives/)

![](./img/archive_img.jpg)

# License

    MIT License Copyright (c) 2013-2016 Blackrock Digital LLC.
    Apache License 2.0 Copyright(c) 2015-2016 Huxpro  
    Apache License 2.0 Copyright(c) 2016 phantomVK
