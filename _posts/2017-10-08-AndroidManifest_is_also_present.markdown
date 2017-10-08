---
layout:     post
title:      "Error: AndroidManifest.xml is also present at ..."
date:       2017-10-08
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

导入多个第三方依赖库的时候报错：

```
Error:Execution failed for task ':app:processDebugManifest'.
> Manifest merger failed : Attribute meta-data#android.support.VERSION@value value=(26.0.2) from [com.android.support:preference-v7:26.0.2] AndroidManifest.xml:25:13-35
  	is also present at [com.android.support:appcompat-v7:26.1.0] AndroidManifest.xml:28:13-35 value=(26.1.0).
  	Suggestion: add 'tools:replace="android:value"' to <meta-data> element at AndroidManifest.xml:23:9-25:38 to override.
```

在`build.gradle(project:...)`的`buildscript {...}`内添加添加代码。`${versions.support}`是一个全局变量，为"26.1.0"，请根据具体情况修改。

```groovy
subprojects {
    project.configurations.all {
        resolutionStrategy.eachDependency { details ->
            if (details.requested.group == 'com.android.support'
                    && !details.requested.name.contains('multidex')) {
                details.useVersion "${versions.support}"
            }
        }
    }
}
```


