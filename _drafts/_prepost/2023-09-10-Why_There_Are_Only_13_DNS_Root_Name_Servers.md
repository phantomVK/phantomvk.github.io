---
layout:     post
title:      "为什么DNS根服务器只有13台?"
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Network
---



RFC 791

```
Every internet destination must be able to receive a datagram of 576
octets either in one piece or in fragments to be reassembled.
```





```shell
# dig -t ns

; <<>> DiG 9.10.6 <<>> -t ns
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9678
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;.				IN	NS

;; ANSWER SECTION:
.			164657	IN	NS	m.root-servers.net.
.			164657	IN	NS	k.root-servers.net.
.			164657	IN	NS	f.root-servers.net.
.			164657	IN	NS	i.root-servers.net.
.			164657	IN	NS	g.root-servers.net.
.			164657	IN	NS	a.root-servers.net.
.			164657	IN	NS	j.root-servers.net.
.			164657	IN	NS	c.root-servers.net.
.			164657	IN	NS	d.root-servers.net.
.			164657	IN	NS	e.root-servers.net.
.			164657	IN	NS	b.root-servers.net.
.			164657	IN	NS	h.root-servers.net.
.			164657	IN	NS	l.root-servers.net.

;; ADDITIONAL SECTION:
l.root-servers.net.	164657	IN	A	199.7.83.42
b.root-servers.net.	164657	IN	A	199.9.14.201
a.root-servers.net.	164657	IN	A	198.41.0.4
f.root-servers.net.	275820	IN	A	192.5.5.241
h.root-servers.net.	164657	IN	A	198.97.190.53
d.root-servers.net.	164657	IN	A	199.7.91.13
e.root-servers.net.	164657	IN	A	192.203.230.10
g.root-servers.net.	277097	IN	A	192.112.36.4
k.root-servers.net.	164657	IN	A	193.0.14.129
j.root-servers.net.	164657	IN	A	192.58.128.30
i.root-servers.net.	277442	IN	A	192.36.148.17
c.root-servers.net.	164657	IN	A	192.33.4.12
m.root-servers.net.	277086	IN	A	202.12.27.33
l.root-servers.net.	164657	IN	AAAA	2001:500:9f::42
b.root-servers.net.	276038	IN	AAAA	2001:500:200::b
a.root-servers.net.	276605	IN	AAAA	2001:503:ba3e::2:30
f.root-servers.net.	164657	IN	AAAA	2001:500:2f::f
h.root-servers.net.	164657	IN	AAAA	2001:500:1::53
d.root-servers.net.	164657	IN	AAAA	2001:500:2d::d
e.root-servers.net.	164657	IN	AAAA	2001:500:a8::e
g.root-servers.net.	164657	IN	AAAA	2001:500:12::d0d
k.root-servers.net.	276861	IN	AAAA	2001:7fd::1
j.root-servers.net.	164657	IN	AAAA	2001:503:c27::2:30
i.root-servers.net.	164657	IN	AAAA	2001:7fe::53
c.root-servers.net.	279939	IN	AAAA	2001:500:2::c
m.root-servers.net.	164657	IN	AAAA	2001:dc3::35

;; Query time: 52 msec
;; SERVER: 172.22.233.180#53(172.22.233.180)
;; WHEN: Sun Sep 10 18:16:04 CST 2023
;; MSG SIZE  rcvd: 811
```



### 参考资料

- https://datatracker.ietf.org/doc/html/rfc791
