---
layout:     post
title:      "dlopen is 32-bit instead of 64-bit"
date:       2017-06-05
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

有的手机默认支持64位，启动的时候会尝试加载64位的so。不过包却不一定对64位做出支持。当系统无法加载到理想的包，就会抛出以下异常。

```
06-01 14:40:51.903 25196-25196/com.abc.app.pkg E/art: 
    dlopen("/data/data/com.abc.app.pkg/files/libs/libBaiduMapSDK_base_v4_2_1.so", RTLD_LAZY) failed: 
    dlopen failed: "/data/data/com.abc.app.pkg/files/libs/libBaiduMapSDK_base_v4_2_1.so" is 32-bit instead of 64-bit

06-01 14:40:51.904 25196-25196/com.abc.app.pkg E/NativeLoader: loadException java.lang.UnsatisfiedLinkError: 
    dlopen failed: "/data/data/com.abc.app.pkg/files/libs/libBaiduMapSDK_base_v4_2_1.so" is 32-bit instead of 64-bit
            at java.lang.Runtime.load(Runtime.java:331)
            at java.lang.System.load(System.java:981)
            at com.baidu.platform.comapi.NativeLoader.f(Unknown Source)
            at com.baidu.platform.comapi.NativeLoader.a(Unknown Source)
            at com.baidu.platform.comapi.NativeLoader.c(Unknown Source)
            at com.baidu.platform.comapi.NativeLoader.loadCustomizeNativeLibrary(Unknown Source)
            at com.baidu.platform.comapi.NativeLoader.loadLibrary(Unknown Source)
            at com.baidu.platform.comapi.a.<clinit>(Unknown Source)
            at com.baidu.platform.comapi.a.a(Unknown Source)
            at com.baidu.platform.comapi.b.a(Unknown Source)
            at com.baidu.mapapi.SDKInitializer.initialize(Unknown Source)
            at com.baidu.mapapi.SDKInitializer.initialize(Unknown Source)
            at com.gnet.calendarsdk.common.MyApplication.init(MyApplication.java:92)
            at com.gnet.calendarsdk.UCCalendarClient.init(UCCalendarClient.java:34)
            at im.vector.Application.initQuanshiSDK(Application.java:226)
            at im.vector.Application.onCreate(Application.java:126)
            at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1024)
            at android.app.ActivityThread.handleBindApplication(ActivityThread.java:5076)
            at android.app.ActivityThread.access$1600(ActivityThread.java:187)
            at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1645)
            at android.os.Handler.dispatchMessage(Handler.java:111)
            at android.os.Looper.loop(Looper.java:194)
            at android.app.ActivityThread.main(ActivityThread.java:5869)
            at java.lang.reflect.Method.invoke(Native Method)
            at java.lang.reflect.Method.invoke(Method.java:372)
            at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1019)
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:814)
```  
  
不过，Android 64位是可以向下兼容32位程序的，apk只需要在budil.gradle中明确支持的指令集，以此适配手机即可。

```groovy
android {
    defaultConfig {
        ndk {
            abiFilters "armeabi", "armeabi-v7a", "x86", "mips"
        }
    }
}
```

