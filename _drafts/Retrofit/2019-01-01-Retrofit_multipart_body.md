---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

```kotlin
val requestBody = RequestBody.create(MediaType.parse("file/*"), File(filepath))
val body = MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("convertType", type)
        .addFormDataPart("file", fileName ?: "预览文件", requestBody)
        .build()

val request = Request.Builder()
        .url(urlString)
        .post(body)
        .build()
```

