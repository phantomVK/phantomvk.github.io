---
layout:     post
title:      ""
subtitle:   ""
date:       2017-01-01
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - tags
---

## supportInvalidateOptionsMenu() deprecated

```
'supportInvalidateOptionsMenu(): Unit' is deprecated. Overrides deprecated member in 'android.support.v4.app.FragmentActivity'. Deprecated in Java
:projectName:javaPreCompileDebug
```

__Fix:__

```java
invalidateOptionsMenu()
```

