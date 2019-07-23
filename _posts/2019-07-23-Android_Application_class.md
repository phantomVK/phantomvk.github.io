---
layout:     post
title:      "Application创建过程"
date:       2019-07-23
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 一、简介

#### 1.1 简介

__Application__ 在Android应用进程占据重要地位，每个进程只有一个实例，继承自 __Context__ 父类可直接当 __Context__ 使用，肩负起众多功能。

![application_uml](/img/android/Application/application_uml.png)

#### 1.2 启动优化

应用第三方依赖库的 __初始化__、__定义全局配置__、__缓存建立__ 操作都在 __onCreate()__ 执行。当依赖库日渐增多而 __Application__ 初始化又在主线程进行，初始化任务越多，应用冷启动时间越长。因此，可把不依赖主线程的依赖库，放在子线程进行初始化。

为了令整个初始化流程更早结束，还要注意子线程数量、子线程最长执行时长，和子线程抢夺主线程时间片等问题。线程创建操作也是耗时操作，且用完的子线程要考虑是否有必要复用。

个人建议直接创建最多两个不复用的子线程用于初始化操作，或在 __AsyncTask__ 内并行执行。

```java
AsyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, Param)
```

#### 1.3 类声明

没有自定义 __Application__ 的 __AndroidManifest__ 如下：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.phantomvk.messagekit">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:theme="@style/AppTheme">

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

自定义 __Application__ 按需执行第三方依赖库初始化操作。


```java
package com.phantomvk.messagekit

import android.app.Application

class Application : Application() {
    override fun onCreate() {
        super.onCreate()
        // Init here.
    }
}
```

然后在 __AndroidManifest__ 内声明自定义的 __Application__。由下面 __package__ 和 __name__ 组合可知自定义 __Application__ 全路径为 __com.phantomvk.messagekit.Application__，后面会用到这个全路径名。

没有自定义 __Application__ 则全路径名默认为 __android.app.Application__，即使编写自定义类也不会被调用。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.phantomvk.messagekit">

    <application
        android:name=".Application"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:theme="@style/AppTheme">
    </application>
</manifest>
```

## 二、启动流程

#### 2.1 IPC注册

App启动时有注册 __ApplicationThread__ 实例到AMS的步骤，注册成功后AMS通过IPC调用 __ApplicationThread.bindApplication()__ 发出 __H.BIND_APPLICATION__ 消息，消息体内附 __AppBindData__ 作为后续初始化所需数据。

```java
private class ApplicationThread extends IApplicationThread.Stub {
    public final void bindApplication(String processName, ApplicationInfo appInfo,
            List<ProviderInfo> providers, ComponentName instrumentationName,
            ProfilerInfo profilerInfo, Bundle instrumentationArgs,
            IInstrumentationWatcher instrumentationWatcher,
            IUiAutomationConnection instrumentationUiConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation,
            boolean isRestrictedBackupMode, boolean persistent, Configuration config,
            CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
            String buildSerial, boolean autofillCompatibilityEnabled) {
        .....

        AppBindData data = new AppBindData();
        data.processName = processName;
        data.appInfo = appInfo;
        data.providers = providers;
        data.instrumentationName = instrumentationName;
        data.instrumentationArgs = instrumentationArgs;
        data.instrumentationWatcher = instrumentationWatcher;
        data.instrumentationUiAutomationConnection = instrumentationUiConnection;
        data.debugMode = debugMode;
        data.enableBinderTracking = enableBinderTracking;
        data.trackAllocation = trackAllocation;
        data.restrictedBackupMode = isRestrictedBackupMode;
        data.persistent = persistent;
        data.config = config;
        data.compatInfo = compatInfo;
        data.initProfilerInfo = profilerInfo;
        data.buildSerial = buildSerial;
        data.autofillCompatibilityEnabled = autofillCompatibilityEnabled;
        // 发出H.BIND_APPLICATION消息
        sendMessage(H.BIND_APPLICATION, data);
    }
}
```

#### 2.2 执行消息

主线程收到 __H.BIND_APPLICATION__ 消息，由 __Handler__ 实现类 __H__ 处理：

```java
class H extends Handler {
    public static final int BIND_APPLICATION        = 110;

    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BIND_APPLICATION:
                AppBindData data = (AppBindData)msg.obj;
                // 调用handleBindApplication()
                handleBindApplication(data);
                break;

            .....
        }
  }
```

#### 2.3 初始化

开始调用 __handleBindApplication(AppBindData data)__：

```java
private void handleBindApplication(AppBindData data) {
    .....

    Application app;
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        // data.info类型为LoadedApk，LoadedApk通过反射创建Application实例
        app = data.info.makeApplication(data.restrictedBackupMode, null);

        // Propagate autofill compat state
        app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);

        // 把Application引用保存在ActivityThread.mInitialApplication
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
        } catch (Exception e) {
            throw new RuntimeException(
                "Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
        }
        try {
            // 创建完成的Application实例，触发onCreate()调用开始内部初始化
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

上述源码中 __data.info__ 的类为 __LoadedApk__ ，直接看 __LoadedApk.makeApplication()__。方法内会检查 __Application__ 是否重复创建。


```java
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    // 如果已经初始化则LoadedApk.mApplication非空
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
        // 先用ActivityThread的引用创建ContextImpl实例
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 然后通过Application类名反射获得实例，并把appContext保存到实例中
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        // ContextImpl和Application互相保存Context
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
    // 赋值给LoadedApk.mApplication
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

#### 2.4 实例创建

展开上述源码中的 __Application__ 反射创建流程。

```java
// android.app.Instrumentation

public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    // context设置到Applicaton实例父类，即Context的数据成员mBase;
    app.attach(context);
    return app;
}

private AppComponentFactory getFactory(String pkg) {
    if (pkg == null) {
        Log.e(TAG, "No pkg specified, disabling AppComponentFactory");
        return AppComponentFactory.DEFAULT;
    }
    if (mThread == null) {
        Log.e(TAG, "Uninitialized ActivityThread, likely app-created Instrumentation,"
                + " disabling AppComponentFactory", new Throwable());
        return AppComponentFactory.DEFAULT;
    }
    LoadedApk apk = mThread.peekPackageInfo(pkg, true);
    // This is in the case of starting up "android".
    if (apk == null) apk = mThread.getSystemContext().mPackageInfo;
    return apk.getAppFactory();
}
```

上文把名为 __context__ 的 __ContextImpl__ 实例设置给 __Application__ 的父类成员 __mBase__。每次把 __Application__ 当 __Context__ 使用时，操作通过 __ContextWrapper__ 委托给 __mBase__ 实例，即上文的 __ContextImpl__ 实例。

而实例化 __Application__ 和一般Java反射创建实例没有差别，直接通过类名 __newInstance()__ 得到。

```java
public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
        @NonNull String className)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    // 从ClassLoader用默认构造方法反射建立实例
    return (Application) cl.loadClass(className).newInstance();
}
```

### 三、使用实例

最后看看如何从 __Application__ 获取 __applicationContext__，前者继承自 __ContextWrapper__，功能也在父类实现。

```java
public class Application extends ContextWrapper implements ComponentCallbacks2
```

可知 __application.getApplicationContext__ 取自 __ContextWrapper.mBase__。

```java
public class ContextWrapper extends Context {
    Context mBase;
  
    @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
}
```

__mBase__ 为实现抽象类 __Context__ 的 __ContextImpl__ 实例。

```java
class ContextImpl extends Context {
    final @NonNull LoadedApk mPackageInfo;
  
    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
}
```

从 __mPackageInfo.getApplication()__ 进入 __LoadedApk__ 类。这个 __mApplication__ 就是初始化时构建的 __Application__ 实例。

```java
public final class LoadedApk {
    private Application mApplication;

    Application getApplication() {
        return mApplication;
    }  
}
```

## 四、总结

简单流程总结：

- __Application__ 由 __LoadedApk__ 构建，保存到 __mApplication__，引用返回给 __ActivityThread__；
- __ActivityThread__ 把 __LoadedApk__ 返回的对象保存到 __mInitialApplication__；
- 因此 __LoadedApk.mApplication__ 和 __ActivityThread.mInitialApplication__ 指向相同对象；
- 正常情况这两个对象也不可能为空。

__Application__ 基本创建流程如上，如果对其他细节感兴趣，可以自行查看 __LoadedApk__、__ContextImpl__、__Instrumentation__ 等类源码。

