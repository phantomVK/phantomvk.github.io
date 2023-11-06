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

> 2020.9.15更新：
>
> 经过长期测试后，确定压缩脚本的处理逻辑存在大量问题。如<pre>和</head>标签会被错误移除、空行没有正确移除等等。即使更新为最新发布版本，以上问题依然存在，因此不建议使用。

从16年10月至今，坚持每月写博客快2年了，也是使用GitHub Pages和Jekyll的2年。当时，用了第三方开源Jekyll主题，修修补补增加一些简陋的自定制功能，算是满足基本需求。但是，HTML代码优化的问题直到最近才找到解决方法。

[compress.html](https://github.com/penibelst/jekyll-compress-html/blob/master/site/_layouts/compress.html)使用Jekyll layout进行代码压缩，不需依赖任何第三方工具，不依赖具体部署平台，配置简单、方便易用。

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

最后，启动本地服务，在浏览器内检查页面HTML源码，即可发现代码已经开启压缩。最好再通过浏览器的控制台查看压缩后代码是否存在问题，尤其是 __JavaScript__，如果编写的时候不够仔细压缩后会出现语法问题。