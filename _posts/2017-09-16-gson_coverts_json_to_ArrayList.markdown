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
public class UriInOut implements JsonSerializer<Uri>, JsonDeserializer<Uri> {

    @Override
    public JsonElement serialize(Uri src, Type typeOfSrc, JsonSerializationContext context) {
        return new JsonPrimitive(src.toString());
    }

    @Override
    public Uri deserialize(final JsonElement src, final Type srcType,
                           final JsonDeserializationContext context) throws JsonParseException {
        // Convert "https://abc.com" to https://abc.com
        return Uri.parse(src.toString().replace("\"", ""));
    }
}
```

构建Gson对象并注册类型转换器`UriDeserializer`，用fromJson转换字符串`list`为对应的ArrayList实例

```java
Gson mGson = new GsonBuilder()
        .registerTypeAdapter(Uri.class, new UriInOut())
        .create();
    
return mGson.fromJson(list, new TypeToken<ArrayList<HomeserverConnectionConfig>>(){}.getType());
```

参考：

<https://stackoverflow.com/questions/12384064/gson-convert-from-json-to-a-typed-arraylistt>

<https://stackoverflow.com/questions/22533432/create-object-from-gson-string-doesnt-work>

<https://stackoverflow.com/questions/22271779/is-it-possible-to-use-gson-fromjson-to-get-arraylistarrayliststring>

