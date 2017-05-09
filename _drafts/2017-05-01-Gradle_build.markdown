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


# Gradle构建


## 一、默认配置

下面是android域初次构建的变量参数，包括编译版本、包名、应用版本和编译类型等配置。

```groovy
android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "com.phantomvk.app"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

## 二、构建类型

`buildType`默认是`debug`模式，提供并支持`release`模式。由于`debug`模式下，很少甚至不需要使用配置项，所以`debug`配置项是不显示出来，可以手动添加。在`release`里能增加代码混淆和资源压缩的支持。代码混淆需要在`proguard-android.txt`里编写规则。

```groovy
buildTypes {
    debug {
        minifyEnabled false
    }
    release {
        minifyEnabled true
        shrinkResources true
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
}
```


## 三、渠道配置

### 3.1 设置不同渠道

不同版本使用不同链接

```
productFlavors {
    internal {
        buildConfigField "String", "TYPE", "\"Internal\""
        buildConfigField "String", "URL", "\"https://api.phantomvk.com\int\""
    }
    external {
        buildConfigField "String", "TYPE", "\"External\""
        buildConfigField "String", "URL", "\"https://api.phantomvk.com\ext\""
    }
}
```

### 3.2 渠道包名

不同版本，androidManifest中package也名会被替换

```
internal {
    applicationId "com.phantomvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

同理

```
external {
    applicationId "com.phantomvk.app.external"
    buildConfigField "String", "TYPE", "\"External\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\ext\""
}
```

如果不想在applicationId重复长字符串，可以使用`applicationIdSuffix`，构建时自动拼接`defaultConfig.applicationId`和`applicationIdSuffix`。

```
internal {
    applicationIdSuffix ".internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

两者个各有优点，`applicationIdSuffix`长度短，易于阅读；applicationId可以指定其他和本工程完全不同的包名，如“com.pvk.app.internal”。这个报名可以和工程没有任何关系，可以没有出现在工程的其他地方中。

```
internal {
    applicationId "com.pvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

### 3.3 资源配置

更多配置,resValue小写， 删除res/values/string.xml中app_name


```
internal {
    applicationId "com.pvk.app.internal"
    resValue "string", "AppName", "PhantomvkInt"
    manifestPlaceholders = [app_key: "phantomvk"]
    manifestPlaceholders = [app_secret: "jnce3r93n"]
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

```xml
<meta-data android:name="app_key" android:value="${app_key}" />
<meta-data android:name="app_secret" android:value="${app_secret}" />
```

### 3.4 安装包重命名

默认

```
app-external-debug.apk
app-external-release-unsigned.apk
app-internal-debug.apk
app-internal-release-unsigned.apk
```

```
applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def apkName = "Phantom_" + "v${defaultConfig.versionName}_" + variant.productFlavors[0].name
        if (variant.buildType.name == ('release')) {
            apkName += '.apk'
        } else if (variant.buildType.name == ('debug')) {
            apkName += '_debug.apk'
        }
        output.outputFile = new File(output.outputFile.parent, apkName)
    }
}
```

```
Phantom_v1.0_external_debug.apk
Phantom_v1.0_external.apk
Phantom_v1.0_internal_debug.apk
Phantom_v1.0_internal.apk
```


## 四、全局配置

Project build.gradle

```
ext {
    versionCodeProp = 1
    versionNameProp = "1.0"
}
```



Module build.gradle

```
versionCode 1
versionName "1.0"
        
versionCode rootProject.ext.versionCodeProp
versionName rootProject.ext.versionNameProp
```

所有版本共有

```
defaultConfig {
    applicationId "com.phantomvk.app"
    minSdkVersion 16
    targetSdkVersion 25
    versionCode 1
    versionName "1.0"
    buildConfigField "String", "BUG_REPORT_URL", "\"https://phantomvk.com\api\report\""
}
```

### 3.5 分包名后安装问题

在Gradle分包名，安装失败，既不能覆盖，有不能新产生


### 3.6 问题解决

```xml
<provider
    android:name=".data.ContentProvider"
    android:authorities="com.phantomvk.app.provider"
    android:exported="false" />
```


```xml
<provider
    android:name=".data.ContentProvider"
    android:authorities="${applicationId}.provider"
    android:exported="false" />
```

## 五、编译检查

添加lintOptions

Analyze > Inspect Code

```
lintOptions {
    disable 'InvalidPackage'
    abortOnError false
}
```

```
packagingOptions {
    exclude 'META-INF/NOTICE'
    exclude 'META-INF/NOTICE.txt'
    exclude 'META-INF/LICENSE'
    exclude 'META-INF/LICENSE.txt'
}
```


## 六、构建命令

gradle环境,grablew grablew.bat

如果已经构建好`Gradle`的环境，可以直接使用`gradle`命令完成构建工作。如果没有搭建环境，也可以用项目目录下的`gradlew`或`gradlew.bat`


构建命令实例

```
> gradle build

> gradle assembleDebug
> gradle assembleRelease

> gradle assembleInternal
> gradle assembleExternal

> gradle assembleInternalDebug
> gradle assembleInternalRelease
```


