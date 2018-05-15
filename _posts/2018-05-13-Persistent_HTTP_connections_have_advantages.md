---
layout:     post
title:      "HTTP长连接的优点 - RCF2616"
date:       2018-05-13
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Network
---



## 原文

[RCF 2616: Hypertext Transfer Protocol -- HTTP/1.1](https://tools.ietf.org/html/rfc2616#section-8.1.1)

```
Persistent HTTP connections have a number of advantages:

  - By opening and closing fewer TCP connections, CPU time is saved
    in routers and hosts (clients, servers, proxies, gateways,
    tunnels, or caches), and memory used for TCP protocol control
    blocks can be saved in hosts.

  - HTTP requests and responses can be pipelined on a connection.
    Pipelining allows a client to make multiple requests without
    waiting for each response, allowing a single TCP connection to
    be used much more efficiently, with much lower elapsed time.

  - Network congestion is reduced by reducing the number of packets
    caused by TCP opens, and by allowing TCP sufficient time to
    determine the congestion state of the network.

  - Latency on subsequent requests is reduced since there is no time
    spent in TCP's connection opening handshake.

  - HTTP can evolve more gracefully, since errors can be reported
    without the penalty of closing the TCP connection. Clients using
    future versions of HTTP might optimistically try a new feature,
    but if communicating with an older server, retry with old
    semantics after an error is reported.

HTTP implementations SHOULD implement persistent connections.
```

## 翻译

- 通过开启和关闭更少HTTP连接，可节省路由器和主机 (包括：客户端、服务器、代理、网关、隧道和缓存) 的CPU时间，且主机中TCP协议控制块使用内存也有所节省。
- 在同一个连接中，可管道化地传输HTTP请求和HTTP响应。管道能让客户端在不等待到每个响应的情况下，同时发出多个HTTP请求，令一个独立的TCP连接在更短的经历时间中得到更高效的重用。
- 减少了来自TCP建立所需数据包数量，缓解网络拥塞的情况，也给TCP留出充足时间去探测网络拥堵的状况。
- 没有TCP开启连接握手的时间消耗，可降低后续请求的延时时间。
- 错误报告不再受到HTTP连接关闭带来的不利后果，让HTTP可以更加优雅地演进。客户端使用一个将来的HTTP版本可能会乐观地尝试一个新特性，但如果与之建立通讯的是一个更老的服务器，报告错误之后会重试旧版的语义。