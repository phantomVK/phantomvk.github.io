---
layout:     post
title:      "Android注解处理器入门"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

annotation

创建名为 __XAnnotation__

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface XAnnotation {
}
```

annotation_processor

添加 __javapoet__ 或 __kotlinpoet__，推荐使用 __javapoet__


```groovy
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.squareup:javapoet:1.11.1'
    implementation project(":annotation")
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

