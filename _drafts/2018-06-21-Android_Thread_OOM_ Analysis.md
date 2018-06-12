---
layout:     post
title:      "Android OOM源码分析"
date:       2018-06-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---



#### 错误类型A

根据bugly上报统计一共发生8次

```
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Out of memory

java.lang.Thread.nativeCreate(Native Method)
java.lang.Thread.start(Thread.java:745)
java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:941)
java.util.concurrent.ThreadPoolExecutor.ensurePrestart(ThreadPoolExecutor.java:1582)
....
```

#### 错误类型A

根据bugly上报统计一共发生1次

```
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Out of memory

java.lang.Thread.nativeCreate(Native Method)
java.lang.Thread.start(Thread.java:1078)
java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:921)
java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1339)
....
```



#### 错误类型C

根据bugly上报统计一共发生1次

```
java.lang.OutOfMemoryError: Could not allocate JNI Env

java.lang.Thread.nativeCreate(Native Method)
java.lang.Thread.start(Thread.java:745)
java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:941)
java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1359)
....
```

