---
layout:     post
title:      使用Gson转换Json为ArrayList<T>
date:       2017-09-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

如果Json中有Uri相关的字符串，先构建类`UriDeserializer`

```java
public class UriDeserializer implements JsonDeserializer<Uri> {

    @Override
    public Uri deserialize(final JsonElement src, final Type srcType,
                           final JsonDeserializationContext context) throws JsonParseException {

        return Uri.parse(src.toString());
    }
}
```

构建一个Gson对象并注册类型转换器，用于转换Uri的`UriDeserializer`。然后fromJson转换字符串`list`为对应的ArrayList实例

```java
Gson mGson = new GsonBuilder()
        .registerTypeAdapter(Uri.class, new UriDeserializer())
        .create();
    
return mGson.fromJson(list, new TypeToken<ArrayList<HomeserverConnectionConfig>>() {
}.getType());
```




