---
layout:     post
title:      "SharedPreferences与线程安全"
date:       2019-03-24
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---
## 一、前言

__SharedPreferences__ 通过读写磁盘xml文件的方式，为客户端提供便捷的键值对持久化服务。同时，支持同步和异步的数据提交方式，保证主线程尽少被影响。

虽然此工具类因方便使用的优点深得开发者的青睐，但其实现多线程操作、多进程操作是否安全的问题，却鲜有人探究。对 __SharedPreferences__ 存取操作感兴趣的读者，这里先为您呈上文章 [Android源码系列(12) -- SharedPreferences](/2018/09/14/SharedPreferences/)，请慢用。

接下来，将透过应用进程启动的流程，一步步得出主题结论。因为涉及 __ActivityThread__、__ApplicationThread__、__ActivityManagerService__、Android IPC等知识，请自行查阅，本文不再赘述。本文源码来自 __Android 23__。

## 二、ActivityThread

省略前面系统的 __Zygote__ 进程孵化出具体应用进程的 __ActivityThread__，直到 __ActivityThread__ 把 __ApplicationThread__ 注册到 __ActivityManagerService__。注册完成后 __ActivityManagerService__ 通过IPC调用 __IApplicationThread.bindApplication(...)__ 

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {

    .....

    try {
        .....

        // 经过上述处理后，通过IPC调用ApplicationThread.bindApplication()
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked());
        updateLruProcessLocked(app, false, null);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        // todo: Yikes!  What should we do?  For now we will try to
        // start another process, but that could easily get us in
        // an infinite loop of restarting processes...
        Slog.wtf(TAG, "Exception thrown during bind of " + app, e);

        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        startProcessLocked(app, "bind fail", processName);
        return false;
    }

    .....

    return true;
}
```

实现方法是 __ActivityThread.ApplicationThread.bindApplication()__

```java
public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
        Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
        Bundle coreSettings) {

    .....

    IPackageManager pm = getPackageManager();
    android.content.pm.PackageInfo pi = null;
    try {
        pi = pm.getPackageInfo(appInfo.packageName, 0, UserHandle.myUserId());
    } catch (RemoteException e) {
    }
    if (pi != null) {
        boolean sharedUserIdSet = (pi.sharedUserId != null);
        boolean processNameNotDefault =
        (pi.applicationInfo != null &&
         !appInfo.packageName.equals(pi.applicationInfo.processName));
        boolean sharable = (sharedUserIdSet || processNameNotDefault);

        // 除非应用是个共享进程，否则把应用信息告诉VMRuntime
        if (!sharable) {
            VMRuntime.registerAppInfo(appInfo.packageName, appInfo.dataDir,
                                    appInfo.processName);
        }
    }

    // 构建AppBindData
    AppBindData data = new AppBindData();
    data.processName = processName;
    data.appInfo = appInfo;
    data.providers = providers;
    data.instrumentationName = instrumentationName;
    data.instrumentationArgs = instrumentationArgs;
    data.instrumentationWatcher = instrumentationWatcher;
    data.instrumentationUiAutomationConnection = instrumentationUiConnection;
    data.debugMode = debugMode;
    data.enableOpenGlTrace = enableOpenGlTrace;
    data.restrictedBackupMode = isRestrictedBackupMode;
    data.persistent = persistent;
    data.config = config;
    data.compatInfo = compatInfo;
    data.initProfilerInfo = profilerInfo;
    // ApplicationThread内通过Handler发送消息到ActivityThread
    sendMessage(H.BIND_APPLICATION, data);
}
```

__ActivityThread__ 的 __Handler__ 实现类 __H__ 接收 __BIND_APPLICATION__ 指令并执行逻辑

```java
private class H extends Handler
```

回调 __H__ 的方法 __handleMessage(Message msg__) 

```java
public void handleMessage(Message msg) {
    switch (msg.what) {
        .....

        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            // 从Message取出传送的对象
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        
        .....
    }
}
```

上面调用 __ActivityThread__ 的 __handleBindApplication(data)__

```java
// ActivityThread的成员变量mInstrumentation
Instrumentation mInstrumentation;

private void handleBindApplication(AppBindData data) {
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);

    mProfiler = new Profiler();
    .....

    // 创建ContextImpl
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    if (!Process.isIsolated()) {
        final File cacheDir = appContext.getCacheDir();

        if (cacheDir != null) {
            // Provide a usable directory for temporary files
            System.setProperty("java.io.tmpdir", cacheDir.getAbsolutePath());
        } else {
            Log.v(TAG, "Unable to initialize \"java.io.tmpdir\" property due to missing cache directory");
        }

        // Use codeCacheDir to store generated/compiled graphics code
        final File codeCacheDir = appContext.getCodeCacheDir();
        if (codeCacheDir != null) {
            setupGraphicsSupport(data.info, codeCacheDir);
        } else {
            Log.e(TAG, "Unable to setupGraphicsSupport due to missing code-cache directory");
        }
    }

    .....
        
    if (data.instrumentationName != null) {
        InstrumentationInfo ii = null;
        try {
            ii = appContext.getPackageManager().
                getInstrumentationInfo(data.instrumentationName, 0);
        } catch (PackageManager.NameNotFoundException e) {
        }
        if (ii == null) {
            throw new RuntimeException(
                "Unable to find instrumentation info for: "
                + data.instrumentationName);
        }

        mInstrumentationPackageName = ii.packageName;
        mInstrumentationAppDir = ii.sourceDir;
        mInstrumentationSplitAppDirs = ii.splitSourceDirs;
        mInstrumentationLibDir = ii.nativeLibraryDir;
        mInstrumentedAppDir = data.info.getAppDir();
        mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
        mInstrumentedLibDir = data.info.getLibDir();

        ApplicationInfo instrApp = new ApplicationInfo();
        instrApp.packageName = ii.packageName;
        instrApp.sourceDir = ii.sourceDir;
        instrApp.publicSourceDir = ii.publicSourceDir;
        instrApp.splitSourceDirs = ii.splitSourceDirs;
        instrApp.splitPublicSourceDirs = ii.splitPublicSourceDirs;
        instrApp.dataDir = ii.dataDir;
        instrApp.nativeLibraryDir = ii.nativeLibraryDir;
        LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true, false);
        ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

        try {
            // 获取类加载器
            java.lang.ClassLoader cl = instrContext.getClassLoader();
            // 通过反射创建mInstrumentation实例
            // 类名由data.instrumentationName.getClassName()指定
            mInstrumentation = (Instrumentation)
                cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            throw new RuntimeException(
                "Unable to instantiate instrumentation "
                + data.instrumentationName + ": " + e.toString(), e);
        }

        // 上面创建的ContextImpl实例，引用保存在Instrumentation实例里面
        // 每个进程是一个ActivityThread，其中只有一个Instrumentation实例
        // 可知ContextImpl在一个进程里只有一个实例，所以进程能控制其线程安全
        mInstrumentation.init(this, instrContext, appContext,
               new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
               data.instrumentationUiAutomationConnection);

        if (mProfiler.profileFile != null && !ii.handleProfiling
                && mProfiler.profileFd == null) {
            mProfiler.handlingProfiling = true;
            File file = new File(mProfiler.profileFile);
            file.getParentFile().mkdirs();
            Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
        }

    } else {
        mInstrumentation = new Instrumentation();
    }
    
    .....
}
```

## 三、Instrumentation

```java
public class Instrumentation {
    // 用于线程同步的实例
    private final Object mSync = new Object();
    // 持有ActivityThread
    private ActivityThread mThread = null;
    // 从ActivityThread的线程Looper获得mMessageQueue
    private MessageQueue mMessageQueue = null;
    private Context mInstrContext;
    // ApplicationContext，就是ContextImpl的实例
    private Context mAppContext;
    private ComponentName mComponent;
    private Thread mRunner;
    private List<ActivityWaiter> mWaitingActivities;
    private List<ActivityMonitor> mActivityMonitors;
    private IInstrumentationWatcher mWatcher;
    private IUiAutomationConnection mUiAutomationConnection;
    private boolean mAutomaticPerformanceSnapshots = false;
    private PerformanceCollector mPerformanceCollector;
    private Bundle mPerfMetrics = new Bundle();
    private UiAutomation mUiAutomation;
    
    .....
        
    // 实例创建后通过给成员变量赋值
    final void init(ActivityThread thread,
            Context instrContext, Context appContext, ComponentName component, 
            IInstrumentationWatcher watcher, IUiAutomationConnection uiAutomationConnection) {
        mThread = thread;
        mMessageQueue = mThread.getLooper().myQueue();
        mInstrContext = instrContext;
        mAppContext = appContext;
        mComponent = component;
        mWatcher = watcher;
        mUiAutomationConnection = uiAutomationConnection;
    }
}
```

## 四、ContextImpl

而在 __ContextImpl__ 里面，持有一个静态哈希表，键为文件的包路径，值为该包路径对应的 __SharedPreferences__ 的实例。如果该包路径实例不存在就创建新实例，否则从哈希表中获取实例。

由于 __getSharedPreferences__ 内部把 __ContextImpl.class__ 类实例作为锁对象，所以每次获取指定包路径对应实例都是线程安全的。

根据类签名可知 __ContextImpl__ 是 __Context__ 的具体实现类，为 __Activity__ 和其他应用组件提供基础上下文对象。

```java
class ContextImpl extends Context {
    .....
    
    // 包名和对应已缓存的SharedPreferencesImpl实例
    private static ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>> sSharedPrefs;
    
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            if (sSharedPrefs == null) {
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }

            final String packageName = getPackageName();
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }

            // At least one application in the world actually passes in a null
            // name.  This happened to work because when we generated the file name
            // we would stringify it to "null.xml".  Nice.
            if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                    Build.VERSION_CODES.KITKAT) {
                if (name == null) {
                    name = "null";
                }
            }

            sp = packagePrefs.get(name);
            if (sp == null) {
                File prefsFile = getSharedPrefsFile(name);
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
        }
        // 只在HONEYCOMB或以下的版本会对不可预料的数据进行重载
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }

    .....
}
```

## 五、总结

上面的解析已经移除很多不相关的源码，流程已经足够简洁。

总结流程如下：

- __ApplicationThread__ 是 __ActivityThread__ 的内部类，也是其成员变量之一；

- __ActivityThread__ 创建后把自己的 __ApplicationThread__ 实例 __IPC__ 注册到 __ActivityManagerService__；
- __ActivityManagerService__ 注册 __ApplicationThread__ 之后 __IPC__ 调用后者 __bindApplication()__ 方法，表示注册工作已完成。拜托 __ApplicationThread__ 告知 __ActivityThread__ 继续进行 __Application__ 初始化；
- __ApplicationThread.bindApplication()__ 通过发送标志为 __BIND_APPLICATION__ 的 __Message__ 告知 __ActivityThread__ 进行 __Applicatoin__ 初始化；
- __ActivityThread__ 执行 __handleBindApplication()__，方法内部创建 __Instrumentation__ 保存在成员变量 __mInstrumentation__；
- 方法内部随后也创建 __ContextImpl__ 实例，把 __ContextImpl__ 实例引用保存于 __mInstrumentation__；
- 所有以上所有实例均只有一个，且 __SharedPreferences__ 的获取线程安全；

延伸问题，上面分析已知 __SharedPreferences__ 线程安全。而 __SharedPreferences__ 表面支持进程安全，即多个进程可同时写入文件，但实际 __Google__ 并不认可这种操作。因为多个文件写入操作没有在系统层调度协调，不能保证同时写入的安全。

## 六、参考链接

- [SharedPreferences and Thread Safety - StackOverflow](https://stackoverflow.com/questions/4693387/sharedpreferences-and-thread-safety)
- [SharedPreferences.Editor.apply()](https://developer.android.com/reference/android/content/SharedPreferences.Editor.html#apply())
- [SharedPreferences.Editor.commit()](https://developer.android.com/reference/android/content/SharedPreferences.Editor.html#commit())