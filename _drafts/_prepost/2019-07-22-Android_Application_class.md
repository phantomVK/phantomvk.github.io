---
layout:     post
title:      "Application创建过程"
date:       2019-07-22
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

__Application__ 在Android进程占据重要地位，每个进程只有一个实例，且继承自 __Context__ 可直接当 __Context__ 使用。多数第三方依赖库的初始化、应用全局配置、建立缓存都在其初始化过程中执行。

当应用依赖库日渐增多，而 __Application__ 初始化就在主线程里进行，整个初始化过程越长，导致应用冷启动时间也会越长。可以把不依赖主线程的依赖库，放在子线程进行初始化。为了令整个初始化流程总体更快结束，还要注意控制创建子线程数量、每个子线程执行时长，和子线程不能和主线程抢夺时间片等问题。

因为创建子线程本身也是耗时操作，并且要考虑用完的子线程是否有必要复用。建议直接创建一个不复用的子线程，或把创建一个异步任务在 __AsyncTask__ 里并行执行。

没有自定义 __Application__ 的 __AndroidManifest__ 如下：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.phantomvk.messagekit">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme"
        tools:ignore="GoogleAppIndexingWarning">

        <activity
            android:name=".view.MainActivity"
            android:theme="@style/AppTheme.Launcher">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

自定义 __Application__ 内按需行第三方依赖库的初始化操作。


```java
package com.phantomvk.messagekit

import android.app.Application

class Application : Application() {

    override fun onCreate() {
        super.onCreate()
        // Init.
    }
}
```

然后在 __AndroidManifest__ 内声明自定义的 __Application__。由下面 __package__ 和 __name__ 组合可知自定义 __Application__ 全路径为 __com.phantomvk.messagekit.Application__。后面会用到这个全路径名。

如果没有自定义 __Application__，则全路径名默认为 __android.app.Application__，即使已经编写自定义类也不会调用。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.phantomvk.messagekit">

    <application
        android:name=".Application"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme"
        tools:ignore="GoogleAppIndexingWarning">
    </application>
</manifest>
```

主线程收到 __H.BIND_APPLICATION__ 消息，调用 __handleBindApplication(AppBindData data)__。省略其他不相关的操作，直接来到 __Application__ 的创建流程。

```java
private void handleBindApplication(AppBindData data) {
    .....

    Application app;
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        // LoadedApk通过反射创建Application实例
        app = data.info.makeApplication(data.restrictedBackupMode, null);

        // Propagate autofill compat state
        app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);

        mInitialApplication = app;

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                installContentProviders(app, data.providers);
                // For process that contains content providers, we want to
                // ensure that the JIT is enabled "at some point".
                mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
            }
        }

        // Do this after providers, since instrumentation tests generally start their
        // test thread at this point, and we don't want that racing.
        try {
            mInstrumentation.onCreate(data.instrumentationArgs);
        }
        catch (Exception e) {
            throw new RuntimeException(
                "Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
        }
        try {
            // 创建完成的Application，触发onCreate()的调用开始我们的自定义初始化
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                  "Unable to create application " + app.getClass().getName()
                  + ": " + e.toString(), e);
            }
        }
    } finally {
      .....
    }
    .....
}
```

上述源码中 __data.info__ 的类为 __LoadedApk__ ，直接看 __LoadedApk.makeApplication()__。方法内检查 __Application__ 已经创建就不会再次初始化。


```java
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    // 已经初始化则直接退出
    if (mApplication != null) {
        return mApplication;
    }

    Application app = null;

    // 从ApplicationInfo，即AndroidManifest中获取定义的Application名称
    // 从文初实例可知类型为: com.phantomvk.messagekit.Application
    String appClass = mApplicationInfo.className;
    // 若没有声明自定义类，从这里可知默认类名为: android.app.Application
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        // 类加载器用于Application类加载
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            initializeJavaContextClassLoader();
        }
        // 先用ActivityThread引用创建ContextImpl的实例
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 然后通过上述类名反射获得Application实例，appContext会保存到实例中
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + ": " + e.toString(), e);
        }
    }
    // 把Application记录到ActivityThread中
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!instrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    }

    // Rewrite the R 'constants' for all library apks.
    SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }

        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }

    return app;
}
```

展开上述源码中的 __Application__ 反射创建流程。

```java
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    // context设置到Applicaton实例父类，即Context的数据成员: Context mBase;
    app.attach(context);
    return app;
}
```

上文把名为 __context__ 的 __ContextImpl__ 实例设置给 __Applicaton__ 的父类成员 __mBase__。以后每当把 __Application__ 当 __Context__ 使用是，其实所有操作都委托给  __mBase__ 这个实例。

而实例或 __Application__ 则和普通的 Java 反射创建实例没有差别，即直接通过类名 __newInstance()__ 得到实例。

```java
public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
        @NonNull String className)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Application) cl.loadClass(className).newInstance();
}
```

