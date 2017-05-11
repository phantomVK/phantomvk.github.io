---
layout:     post
title:      "Gradle构建安装包"
date:       2017-05-10
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android
---



## 一、默认配置

首先看看android域默认构建参数：`defaultConfig`包含应用默认包名、最低SDK版本、目标SDK版本、应用版本序号和应用版本代号，其他地方的相同配置会覆盖默认参数。

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

`defaultConfig`里定义的变量在所有子域中生效，前提是没有被新变量值覆盖。例如：

```groovy
defaultConfig {
        applicationId "com.phantomvk.app"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        buildConfigField "String", "BUG_REPORT_URL", "\"https://phantomvk.com\api\report\""
    }
```

## 二、构建类型

`buildType`构建类型默认是`debug`模式，提供并支持`release`模式。

由于`debug`模式很少甚至不需要使用自定义配置项，所以`debug`配置项是不显示出来的，倒是可以显式添加。

`release`里能开启代码混淆和资源压缩的支持，需要在`proguard-android.txt`里编写规则。

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

`productFlavors`里可自定义应用不同版本，如免费版和收费版，内部版和公开版。不同版本可以利用静态变量使用各自定义值。

`buildConfigField`定义值经过编译后可以在Java代码中访问`BuildConfig`类获得。

下面的示例为内部版和外部版`buildConfigField`分别设置相同`URL`字符串类型变量名，但对应不同变量值。

```groovy
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

如果在渠道中定义了`applicationId`，这个值会覆盖`defaultConfig`里面的`applicationId`，同时修改`androidManifest`中`package`的项目包名。大家可以自己尝试定义一下，然后解apk看`androidManifest`的内容。

一个名为`internal`的渠道版本

```groovy
internal {
    applicationId "com.phantomvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

同理`external`

```groovy
external {
    applicationId "com.phantomvk.app.external"
    buildConfigField "String", "TYPE", "\"External\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\ext\""
}
```

如果不想在`applicationId`重复长字符串，可以使用`applicationIdSuffix`实现。构建时会把拼接`defaultConfig.applicationId`和`applicationIdSuffix`的结果作为新包名，取代默认的`defaultConfig.applicationId`。

如下`applicationIdSuffix`的结果是`com.phantomvk.app.internal`

```groovy
internal {
    applicationIdSuffix ".internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

`applicationId`和`applicationIdSuffix`各有优缺点：

* `applicationIdSuffix`长度短，易于阅读，作为后缀添加；
* `applicationId`可指定其他和本工程完全不同的包名，如`com.pvk.app.internal`；但管理多个互相没有任何关联的包名容易引起混乱

下面使用`applicationId`定义一个新的渠道包名

```groovy
internal {
    applicationId "com.pvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

### 3.3 资源配置

`resValue`可以在`res/values`的子文件中访问。

如自定义不同渠道的App名：先删除`res/values/string.xml`中`app_name`，然后定义一个名为`app_name`的`resValue`变量。

```groovy
internal {
    applicationId "com.pvk.app.internal"
    resValue "string", "AppName", "PhantomvkInt"
    manifestPlaceholders = [app_key: "phantomvk"]
    manifestPlaceholders = [app_secret: "jnce3r93n"]
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

而`manifestPlaceholders`定义的键值对可以在`AndroidManifest.xml`里读取

```xml
<meta-data android:name="app_key" android:value="${app_key}" />
<meta-data android:name="app_secret" android:value="${app_secret}" />
```

### 3.4 安装包重命名

Android Studio默认App安装包名字如下，相比网上发布的应用，缺少应用名、应用版本号等字段。

```
app-external-debug.apk
app-external-release-unsigned.apk
app-internal-debug.apk
app-internal-release-unsigned.apk
```

打包过程中可以执行命名规则，对每个成功编译的安装包重命名为合适的名字

```groovy
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

上面的规则产生以下的安装包名。相比默认名字，易读且信息量更多。

```
Phantom_v1.0_external_debug.apk
Phantom_v1.0_external.apk
Phantom_v1.0_internal_debug.apk
Phantom_v1.0_internal.apk
```


## 四、全局配置

### 4.1 项目全局变量

为了统一项目所有的版本代号，先在`Project`的`build.gradle`中设置全局变量

```groovy
ext {
    versionCodeProp = 1
    versionNameProp = "1.0"
}
```



在各个`Module`下的`build.gradle`可以用以下的方式引用

```groovy
versionCode rootProject.ext.versionCodeProp
versionName rootProject.ext.versionNameProp
```

### 4.2 分包名安装报错

我曾经遇到在Gradle中渠道正确分包名，但在同一台手机里安装不同渠道应用会失败的问题。现象是既不能覆盖前一个渠道应用，又不能安装为一个新的应用，仅报错：`无法完成安装`

这个问题的原因是`AndroidManifest`里`ContentProvider`声明组件时使用了相同授权名。因此多个`ContentProvider`是不能同时安装的，不然手机会不知道应该提供哪个`ContentProvider`

```xml
<provider
    android:name=".data.ContentProvider"
    android:authorities="com.phantomvk.app.provider"
    android:exported="false" />
```

我定义的是`applicationId`绝对包名，所以用如下方法解决问题

```xml
<provider
    android:name=".data.ContentProvider"
    android:authorities="${applicationId}.provider"
    android:exported="false" />
```

## 五、编译检查

### 5.1 添加lintOptions

命令行执行gradle编译会执行lint检查，并在遇到`error`时强制退出。不幸的是，这个错误可能是第三方jar包或aar导致的，我们无法改正。

如果这些`error`不会导致应用运行时奔溃或抛异常，用`abortOnError false`忽略所有`error`继续编译。不放心的可以用 Analyze > Inspect Code 检查应用中存在的`warning`和`error`，修正的错误。

```groovy
lintOptions {
    disable 'InvalidPackage'
    abortOnError false
}
```


### 5.2 packagingOptions

多个包中存在相同的文件，在编译合并也可能会报错。因不关系到运行代码，直接排除在打包外即可。

```groovy
packagingOptions {
    exclude 'META-INF/NOTICE'
    exclude 'META-INF/NOTICE.txt'
    exclude 'META-INF/LICENSE'
    exclude 'META-INF/LICENSE.txt'
}
```


## 六、构建命令

已经构建好`Gradle`的环境可使用`gradle`命令完成构建工作。如果没有搭建环境，也可以用项目目录下的`gradlew`或`gradlew.bat`临时构建。

```shell
# 构建所有版本
> gradle build

# 按照debug或release构建
> gradle assembleDebug
> gradle assembleRelease

# 按照自定义渠道构建
> gradle assembleInternal
> gradle assembleExternal

# 构建指定渠道debug版本
> gradle assembleInternalDebug
> gradle assembleInternalRelease
```


