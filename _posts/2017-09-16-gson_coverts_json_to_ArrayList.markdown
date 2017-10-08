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

用Gson.fromJson()转换字符串`list`为对应的ArrayList实例

```java
return mGson.fromJson(list, new TypeToken<ArrayList<HomeServerConfig>>(){}.getType());
```

参考：

<https://stackoverflow.com/questions/12384064/gson-convert-from-json-to-a-typed-arraylistt>

<https://stackoverflow.com/questions/22271779/is-it-possible-to-use-gson-fromjson-to-get-arraylistarrayliststring>

