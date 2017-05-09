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

首先看看android域首次构建的变量参数。

`defaultConfig`包含应用默认包名、最低SDK版本、目标SDK版本、应用版本序号和应用版本代号。其他地方的相同配置参数会覆盖这里的默认参数。

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

`defaultConfig`里定义的变量在所有子域中生效，只要没有被新变量值覆盖。例如：

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

`buildType`默认是`debug`模式，提供并支持`release`模式。由于`debug`模式很少甚至不需要使用自定义配置项，所以`debug`配置项是不显示出来的，倒是可以手动添加。

`release`里能开启代码混淆和资源压缩的支持。代码混淆需要在`proguard-android.txt`里编写规则。

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

`productFlavors`里可自定义应用不同版本，如免费版和收费版，内部版和公开版。不同版本可以利用静态变量使用各自独立值。

`buildConfigField`定义的值经过编译后可以在Java代码中访问`BuildConfig`类中获得。

下面的示例为内部版和外部版`buildConfigField`分别设置了相同的`URL`字符串变量名，但对应变量值却不同。

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

如果在渠道中定义了`applicationId`，这个值会覆盖`defaultConfig`里面的`applicationId`，同时修改`androidManifest`中`package`的项目包名。这个可以自己尝试定义一下，然后解apk看`androidManifest`的内容。

```groovy
internal {
    applicationId "com.phantomvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

同理

```groovy
external {
    applicationId "com.phantomvk.app.external"
    buildConfigField "String", "TYPE", "\"External\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\ext\""
}
```

如果不想在`applicationId`重复长字符串，可以使用`applicationIdSuffix`，构建时`defaultConfig.applicationId`和`applicationIdSuffix`两个字符串，而不是完全覆盖`defaultConfig.applicationId`。

如下`applicationIdSuffix`的结果是`com.phantomvk.app.internal`

```
internal {
    applicationIdSuffix ".internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

`applicationId`和`applicationIdSuffix`各有优缺点：

* `applicationIdSuffix`长度短，易于阅读，只能作为后缀添加；
* `applicationId`指定其他和本工程完全不同的包名，如`com.pvk.app.internal`；但管理多个互相没有任何关联的包名容易引起混乱

下面使用`applicationId`定义一个新的渠道包名

```groovy
internal {
    applicationId "com.pvk.app.internal"
    buildConfigField "String", "TYPE", "\"Internal\""
    buildConfigField "String", "URL", "\"https://phantomvk.com\api\int\""
}
```

### 3.3 资源配置

`resValue`可以在`res/values`中访问。如在自定义不同渠道的App名：先删除`res/values/string.xml`中`app_name`，然后定义一个名为`app_name`的`resValue`变量。

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


`manifestPlaceholders`定义键值对可以在`AndroidManifest.xml`里读取

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

打包过程中可以执行命名规则，对每个成功编译的安装包重命名为合适的名字。

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

上面的规则产生以下的安装包名。相比默认名字，易读且信息量更多。

```
Phantom_v1.0_external_debug.apk
Phantom_v1.0_external.apk
Phantom_v1.0_internal_debug.apk
Phantom_v1.0_internal.apk
```


## 四、全局配置

为了统一项目所有的版本代号，在`Project`的`build.gradle`中设置全局变量

```
ext {
    versionCodeProp = 1
    versionNameProp = "1.0"
}
```



在各个`Module`下的`build.gradle`可以用下面的方式引用

```
versionCode 1
versionName "1.0"
        
versionCode rootProject.ext.versionCodeProp
versionName rootProject.ext.versionNameProp
```

### 4.1 分包名安装报错

我曾经遇到在Gradle中渠道正确分包名，但在同一台手机里安装不同渠道应用会失败。现象既不能覆盖前一个渠道应用，又不能安装为一个新的应用，显示`无法完成安装`。


### 4.2 问题解决

这个问题是`ContentProvider`申明是使用了相同的授权包名，多个`ContentProvider`是不能安装的，不然手机会不知道应该提供哪个`ContentProvider`，毕竟都同名。

这是我自己遇到的错误实例：

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

命令行执行gradle编译会在lint检查的时候，遇到`error`时强制退出。不幸的是，这个错误可能是jar包或者是aar里面的，我们自己无法改正。

如果这些`error`不会导致应用运行时奔溃或抛异常，用`abortOnError false`忽略所有`error`继续编译。不放心可以用 Analyze > Inspect Code检查应用中存在的`warning`和`error`。

```
lintOptions {
    disable 'InvalidPackage'
    abortOnError false
}
```


### 5.2 packagingOptions

还有多个包中存在相同的文件，在编译合并是会报错。因为不是关系到运行代码，直接排除在打包外。

```
packagingOptions {
    exclude 'META-INF/NOTICE'
    exclude 'META-INF/NOTICE.txt'
    exclude 'META-INF/LICENSE'
    exclude 'META-INF/LICENSE.txt'
}
```


## 六、构建命令

已经构建好`Gradle`的环境可使用`gradle`命令完成构建工作。如果没有搭建环境，也可以用项目目录下的`gradlew`或`gradlew.bat`临时构建。

```
> gradle build

> gradle assembleDebug
> gradle assembleRelease

> gradle assembleInternal
> gradle assembleExternal

> gradle assembleInternalDebug
> gradle assembleInternalRelease
```


