---
layout:     post
title:      "Retrofit2请求添加Header和Query参数"
date:       2017-10-11
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

在`Retrofit2`里一般通过`@Query`注解来给单个请求增加`query`参数，也可能需要给每个请求都加上`query`参数的场景。

如：`Kong`通过`Authorization`的`JWT`值验证请求合法性。

如果有很多接口，每个接口手动添加`@Query`注解是不合理的。最好的方式是拦截所有网络请求，添加`query`参数后再发送。

请看下列代码：

```java
OkHttpClient.Builder client = new OkHttpClient.Builder();
        
client.addInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        
        // 添加Query参数
        HttpUrl httpUrl = request.url()
                .newBuilder()
                .addQueryParameter("QueryNameA", "queryValueA")
                .addQueryParameter("QueryNameB", "queryValueB")
                .build();
        
        // 添加Header参数
        Request request = original.newBuilder()
                .addHeader("HeaderKeyA", "headerValueA")
                .addHeader("HeaderKeyB", "headerValueB")
                .url(httpUrl)
                .build();
        
        return chain.proceed(request);
});
```

通过`.addQueryParameter(key, val)`，把单个甚至多个键值对添加到请求中，`build()`构建`HttpUrl`加入`Request.Builder`中即可。
