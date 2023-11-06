---
layout:     post
title:      "FireFox配置PAC"
date:       2017-03-21
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - 问题解决
---


# 使用范围

这个插件及相应设置适合Linux、MacOSX、Windows下的FireFox浏览器。这个插件必须配合已经连通的SS服务使用，单独使用插件无效。

下面的设置是在Ubuntu 16.04 LTS x86_64演示。


# 安装插件

进入FireFox的`附加组件` - `插件中`，搜索`Autoproxy`。选择下面这个插件，安装之后重启FireFox。

![img](/img/firefox_pac/autoproxy.png)

# 设置AutoProxy

重启后会出现工具，进入首选项

![img](/img/firefox_pac/setting.png)


## 代理服务器

在里面选择`代理服务器` - `编辑代理服务器`

![img](/img/firefox_pac/edit.png)

把里面的首选项全部选中`删除`，只需要剩下一条修改

![img](/img/firefox_pac/delete.png)

给剩下的首选项取名字叫`ss`，端口设置为`1080`，使用`socks5`，确定保存

![img](/img/firefox_pac/add_ss.png)

## 代理规则

在`代理规则`里面`添加订阅`

![img](/img/firefox_pac/add_rule.png)

选择`gfwList`那项保存

![img](/img/firefox_pac/confirm.png)

确认完成设置

![img](/img/firefox_pac/sub.png)

当`Shadowsocks`开启之后就会自动代理，自动选择合适的访问规则。如果遇到不在规则里面的URL，请使用全局模式或自行添加访问规则。

