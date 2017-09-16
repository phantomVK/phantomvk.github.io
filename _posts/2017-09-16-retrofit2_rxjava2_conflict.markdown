---
layout:     post
title:      "RxJava2与Retrofit2冲突"
date:       2017-09-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

如果同时引入以下库

```groovy
implementation 'io.reactivex.rxjava2:rxjava:2.1.3'
implementation 'com.squareup.retrofit2:retrofit:2.3.0'
implementation 'com.squareup.retrofit2:converter-gson:2.3.0'
implementation 'com.squareup.retrofit2:adapter-rxjava:2.3.0'
```

就会出现

```
Error:Execution failed for task ':app:transformResourcesWithMergeJavaResForDebug'.
com.android.build.api.transform.TransformException: com.android.builder.packaging.DuplicateFileException: Duplicate files copied in APK META-INF/rxjava.properties
```

即使以下方式忽略`META-INF/rxjava.properties`，重新编译后还是会出现出现`Unable to merge dex`

```
packagingOptions {  
    exclude 'META-INF/rxjava.properties'
} 
``` 



报错的原因是`com.squareup.retrofit2:adapter-rxjava`只能支`Retrofit1`，不支持`Retrofit2`

只要引入以下的库替代即可：

```
implementation 'com.jakewharton.retrofit:retrofit2-rxjava2-adapter:1.0.0'
```


参考： 

<http://blog.csdn.net/bingducaijun/article/details/53584449>
<http://winkyqin.com/2017/01/21/Android%E5%B8%B8%E8%A7%81%E5%BC%82%E5%B8%B8%E8%A7%A3%E5%86%B3/2017-02-07/>



