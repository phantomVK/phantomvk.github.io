---
layout:     post
title:      "计算月天数"
date:       2017-04-25
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Algorithm
---

计算月有多少天，已包括闰年的情况

```java
days = ( month == 2 ) ? ( 28 + isLeapYear ) : 31 - ( month - 1 ) % 7 % 2;
```

