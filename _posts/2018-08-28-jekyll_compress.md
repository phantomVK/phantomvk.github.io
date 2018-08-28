---
layout:     post
title:      "压缩Jekyll的HTML代码"
date:       2018-08-28
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Jekyll
---

从16年10月至今，坚持每月写博客快2年了，也是使用GitHub Pages和Jekyll的2年。当时，取了别人的开源Jekyll主题，修修补补增加一些简陋的自定制功能，算是满足基本需求。但是，HTML代码优化的问题直到最近才找到解决方法。

[compress.html](https://github.com/penibelst/jekyll-compress-html/blob/master/site/_layouts/compress.html)使用Jekyll layout 进行代码压缩，不需依赖任何第三方工具，不依赖具体部署平台，配置简单、方便易用。

首先，把上述[compress.html](https://github.com/penibelst/jekyll-compress-html/blob/master/site/_layouts/compress.html)文件下载到本地，存放到Jekyll根目录的layout文件夹中。修改layout文件夹下的`default.html`文件，在文件头部增加并保存以下代码：

```html
---
layout: compress
---
```

然后，修改根目录的`_config.yml`文件，在最后增加并保存以下配置：

```
# compress
compress_html:
  clippings:      all
  comments:       ["<!--", "-->"]
  endings:        all
  startings:      [html, head, body]
```

最后，启动本地服务，在浏览器内检查页面HTML源码，即可发现代码已经开启压缩。