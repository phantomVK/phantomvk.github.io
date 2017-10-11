---
layout:     post
title:      "Retrofit2为所有请求添加Query参数"
date:       2017-10-11
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

在Retrofit2里一般通过`@Query`注解来给单个请求增加`query`参数，也可能需要给每个请求都加上`query`参数的场景：如`Kong`通过`Authorization`的`JWT`值验证每个请求的合法性。

如果很多接口，给每个接口单独使用`@Query`注解是不合理的。所以这种情况下最好的方式是拦截所有网络请求，添加`query`参数后再发送。

请看下列代码：

```java
OkHttpClient.Builder client = new OkHttpClient.Builder();
        
client.addInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
    
        Request request = chain.request();
        
        HttpUrl url = request.url()
                .newBuilder()
                .addQueryParameter("query_name", "query_value")
                .addQueryParameter("query_name", "query_value")
                .build();
        
        Request request = original.newBuilder()
                .url(httpUrl)
                .build();
        
        return chain.proceed(request);
});
```

通过`.addQueryParameter(key, val)`，把单对甚至多对`Key-Value`添加到请求中。之后`build()`构建出`HttpUrl`加入`Request.Builder`中即可。


