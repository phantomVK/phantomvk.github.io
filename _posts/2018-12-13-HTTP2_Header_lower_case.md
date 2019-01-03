---
layout:     post
title:      "HTTP/2约束Header大小写"
date:       2018-12-13
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Network
---

#### 起因

晚上 __Android__ 客户端遇到奇怪的问题：在某台新配的服务器上，出现应用层获取 __Header__ 自定义键 __Authorization__ 时出现其值为空，但存在键 __authorization__ 。

#### 调查

通过抓包发现几个疑点：

- 问题以前没有出现，业务也是正常的，唯独连接到这台服务器会出现异常；

- 同一套代码，仅对这台服务器发出的请求行没有HTTP版本号，如：HTTP/1.1；
- 所有请求体、响应体的字符全是小写；
- iOS业务运行没有问题，所以没有抓包查看；

先修复客户端 HTTP/1.1 __大小写不敏感__ 实现为 __大小写敏感__ 的严重问题。调查的脚步不能就此终止，还要明确就为啥这台服务器如此突出。

由于晚上问题发现时后端同事下班了，又一直按照HTTP/1.1的定义查找问题，丝毫没有进展。最后想到HTTP/2才逐渐有头绪，后翻查RFC文档。

#### RFC定义

文档 [RFC7504](https://tools.ietf.org/html/rfc7540#section-8.1.2) 明确指出Header的定义：

```
8.1.2.  HTTP Header Fields

   HTTP header fields carry information as a series of key-value pairs.
   For a listing of registered HTTP headers, see the "Message Header
   Field" registry maintained at <https://www.iana.org/assignments/
   message-headers>.

   Just as in HTTP/1.x, header field names are strings of ASCII
   characters that are compared in a case-insensitive fashion.  However,
   header field names MUST be converted to lowercase prior to their
   encoding in HTTP/2.  A request or response containing uppercase
   header field names MUST be treated as malformed (Section 8.1.2.6).
```

从上文可知，__HTTP/2__ 和 __HTTP/1.x__ 同样使用 __ASCII__ 字符集，但 __HTTP/2__ 头部必须使用小写，而不像 __HTTP/1.x__ 大小写均可。也正是碰上终端业务代码实现不严谨，引发上述问题。

最终确认这台服务端 __Nginx__ 开启了 __HTTP/2__，且客户端问题已修复提交，调查结束。

#### 总结

对于任意网络协议，客户端在上层跟踪数据包有一定难度，所以先用抓包工具辅助是很好的习惯。

尽管各种协议更新演进，如果前端(业务代码、依赖库版本)或后端任意一方出现协议不兼容或实现不严谨，排查问题难度较高，所以应尽可能做到细致严谨。

对客户端来说，把所有HTTP的Header键字符串转换为小写，业务匹配时也用小写键名称(即equalsIgnoreCase)可同时适配 __HTTP/2__ 和 __HTTP/1.x__。注意，如果转小写后出现多个相同Header，必须要求服务端修正，同时前端调整代码。

