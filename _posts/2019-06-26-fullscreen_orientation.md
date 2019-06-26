---
layout:     post
title:      "Only fullscreen opaque activities can request orientation"
date:       2019-06-26
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

#### 问题

最近同事把 __targetSdkVersion__ 从 __26__ 升到 __28__ 后，原本以为没有兼容性问题，没真机检查就发测试版。结果内部试用 __Android 8.0__ 和 __Android 8.1__ 手机打开 __Activity__ 崩溃。

上报错误具体如下：

```
java.lang.RuntimeException:Unable to start activity ComponentInfo{com.xx.xx/com.xx.xxmessage.chat.ui.RoomActivity}: java.lang.IllegalStateException: Only fullscreen activities can request orientation
```

其他开发者也遇到相同问题：

- [Only fullscreen opaque activities can request orientation](https://stackoverflow.com/questions/48072438/java-lang-illegalstateexception-only-fullscreen-opaque-activities-can-request-o)
- [解决Android 8.0的Only fullscreen opaque activities can request orientation](https://www.jianshu.com/p/883c19af074b)
- [Android 8.0跳坑之'Only fullscreen opaque activities can request orientation'](https://blog.csdn.net/DadaCockWire/article/details/80250152)

#### 状况

代码一直都在 __Activity__ 基类的 __onCreate()__ 中调用方向锁定

```kotlin
requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_PORTRAIT
```

除了用代码指定会出现问题，在 __AndroidMenifest__ 指定该参数也不能幸免：

```xml
android:screenOrientation="portrait"
```

这个问题并不仅因单个条件引起，因为界面要做右滑退出，所以Window背景必须设置为透明

```xml
<item name="android:windowIsTranslucent">true</item>
```

结果就碰到官方系统做的检查，出现标题的异常

#### 排查

读AOSP的提交：[Prevent non-fullscreen activities from influencing orientation](https://github.com/aosp-mirror/platform_frameworks_base/commit/39791594560b2326625b663ed6796882900c220f#diff-a9aa0352703240c8ae70f1c83add4bc8R981)，抽出代码如下：

```java
if (ActivityInfo.isFixedOrientation(requestedOrientation)
        && !fullscreen
        && appInfo.targetSdkVersion >= O) {
    throw new IllegalStateException("Only fullscreen activities can request orientation");
}
```

__fullscreen__ 有多个条件控制

```java
Entry ent = AttributeCache.instance().get(packageName,
                realTheme, com.android.internal.R.styleable.Window, userId);

fullscreen = ent != null && !ActivityInfo.isTranslucentOrFloating(ent.array);
```

展开 __isTranslucentOrFloating()__

```java
// Determines whether the {@link Activity} is considered translucent or floating.
public static boolean isTranslucentOrFloating(TypedArray attributes) {
    final boolean isTranslucent =
            attributes.getBoolean(com.android.internal.R.styleable.Window_windowIsTranslucent,
                    false);
    final boolean isSwipeToDismiss = !attributes.hasValue(
            com.android.internal.R.styleable.Window_windowIsTranslucent)
            && attributes.getBoolean(
                    com.android.internal.R.styleable.Window_windowSwipeToDismiss, false);
    final boolean isFloating =
            attributes.getBoolean(com.android.internal.R.styleable.Window_windowIsFloating,
                    false);

    return isFloating || isTranslucent || isSwipeToDismiss;
}
```

只要满足 __isFloating__、__isTranslucent__、__!windowIsTranslucent && isSwipeToDismiss__ 条件其中之一都不算 __fullscreen__，目的是不让 __非全屏__ 或 __透明__ 的界面决定手机界面的朝向，因为透明的界面能透视背景界面。

#### 解决方案

解决这个问题就是打破联合条件之一：

- 可移除已经制定的透明或半透明属性；
- 或移除显式指定屏幕方向的代码；
- 或 __targetSdkVersion__ 不超过 __26__ ；

#### 看法

从个人角度来看，官方在这个问题上的处理手段极为粗暴。正常来说，检查全屏和屏幕方向条件后， 应该先警告开发者，且忽略已经指定的设置，保证应用运行时兼容性。

结果现在非要粗暴抛出异常，非常不厚道。如果测试没有覆盖 __Android 8.0 - Android 8.1__，只验证 __Android9.0__，或者依赖第三方SDK造成问题，发生产完蛋了。