---
layout:     post
title:      "Android Retrofit使用MultipartBody"
date:       2019-07-04
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

指定 __RequestBody__ 上传的文件路径

```kotlin
val requestBody = RequestBody.create(MediaType.parse("file/*"), File(filepath))
```

构建 __MultipartBody__ 消息体

```kotlin
val body = MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("convertType", type)
        .addFormDataPart("file", fileName, requestBody)
        .build()
```

发出请求即可

```kotlin
val request = Request.Builder()
        .url(urlString)
        .post(body)
        .build()
```

