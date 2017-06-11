---
layout:     post
title:      "Android 22新系统授权失败导致闪退"
date:       2017-06-06
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

```
java.lang.RuntimeException: Error receiving broadcast Intent { act=android.net.conn.CONNECTIVITY_CHANGE flg=0x4000010 (has extras) }
      in com.quanshi.tang.network.NetworkUtils$ConnectReceiver@2a526a5
            at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:1235)
            at android.os.Handler.handleCallback(Handler.java:761)
            at android.os.Handler.dispatchMessage(Handler.java:98)
            at android.os.Looper.loop(Looper.java:156)
            at android.app.ActivityThread.main(ActivityThread.java:6585)
            at java.lang.reflect.Method.invoke(Native Method)
            at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:941)
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:831)
      Caused by: java.lang.SecurityException: getSubscriberId: Neither user 10190 nor current process has android.permission.READ_PHONE_STATE.
            at android.os.Parcel.readException(Parcel.java:1665)
            at android.os.Parcel.readException(Parcel.java:1618)
            at com.android.internal.telephony.IPhoneSubInfo$Stub$Proxy.getSubscriberIdForSubscriber(IPhoneSubInfo.java:590)
            at android.telephony.TelephonyManager.getSubscriberId(TelephonyManager.java:2208)
            at android.telephony.TelephonyManager.getSubscriberId(TelephonyManager.java:2189)
            at com.quanshi.tang.network.NetworkUtils.getProvidersName(Unknown Source)
            at com.quanshi.tang.network.NetworkUtils.getNetworkInfo(Unknown Source)
            at com.quanshi.tang.network.NetworkUtils.connectChanged(Unknown Source)
            at com.quanshi.tang.network.NetworkUtils.access$100(Unknown Source)
            at com.quanshi.tang.network.NetworkUtils$ConnectReceiver.onReceive(Unknown Source)
            at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:1222)
            at android.os.Handler.handleCallback(Handler.java:761) 
            at android.os.Handler.dispatchMessage(Handler.java:98) 
            at android.os.Looper.loop(Looper.java:156) 
            at android.app.ActivityThread.main(ActivityThread.java:6585) 
            at java.lang.reflect.Method.invoke(Native Method) 
            at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:941) 
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:831) 
```

从Android 6.0起，系统启用了更加严格的权限管理。如READ_PHONE_STATE这种权限，即使在AndroidManifest中已经声明，如果没有在运行时并向用户明确请求允许权限，又没有做相应处理，就会导致应用闪退。

但是apk可通过build.gradle的targetSDK表明最高支持的系统版本。已知新的权限系统只在Android 6.0(23)及以上的系统有效，那把targetSDK设置小于等于22即可避开新的权限管理。


