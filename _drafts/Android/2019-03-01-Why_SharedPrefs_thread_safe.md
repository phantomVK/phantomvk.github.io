---
layout:     post
title:      "SharedPreferences与线程安全"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---
为何 __SharedPreferences__ 线程安全，且为何 __SharedPreferences__ 进程不安全

__Android 23__

__ActivityThread__ 把 __ApplicationThread__ 注册到 __ActivityManagerService__，注册完成后 __ActivityManagerService__ 通过IPC调用 __IApplicationThread.bindApplication(...)__ 

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {

    // Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }

    if (app == null) {
        Slog.w(TAG, "No pending application record for pid " + pid
                + " (IApplicationThread " + thread + "); dropping process");
        EventLog.writeEvent(EventLogTags.AM_DROP_PROCESS, pid);
        if (pid > 0 && pid != MY_PID) {
            Process.killProcessQuiet(pid);
            //TODO: killProcessGroup(app.info.uid, pid);
        } else {
            try {
                thread.scheduleExit();
            } catch (Exception e) {
                // Ignore exceptions.
            }
        }
        return false;
    }

    // If this application record is still attached to a previous
    // process, clean it up now.
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }

    // Tell the process all about itself.

    if (DEBUG_ALL) Slog.v(
            TAG, "Binding process pid " + pid + " to record " + app);

    final String processName = app.processName;
    try {
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        startProcessLocked(app, "link fail", processName);
        return false;
    }

    EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.userId, app.pid, app.processName);

    app.makeActive(thread, mProcessStats);
    app.curAdj = app.setAdj = -100;
    app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
    app.forcingToForeground = null;
    updateProcessForegroundLocked(app, false, false);
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;
    app.killedByAm = false;

    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

    if (!normalMode) {
        Slog.i(TAG, "Launching preboot mode app: " + app);
    }

    if (DEBUG_ALL) Slog.v(
        TAG, "New app record " + app
        + " thread=" + thread.asBinder() + " pid=" + pid);
    try {
        int testMode = IApplicationThread.DEBUG_OFF;
        if (mDebugApp != null && mDebugApp.equals(processName)) {
            testMode = mWaitForDebugger
                ? IApplicationThread.DEBUG_WAIT
                : IApplicationThread.DEBUG_ON;
            app.debugging = true;
            if (mDebugTransient) {
                mDebugApp = mOrigDebugApp;
                mWaitForDebugger = mOrigWaitForDebugger;
            }
        }
        String profileFile = app.instrumentationProfileFile;
        ParcelFileDescriptor profileFd = null;
        int samplingInterval = 0;
        boolean profileAutoStop = false;
        if (mProfileApp != null && mProfileApp.equals(processName)) {
            mProfileProc = app;
            profileFile = mProfileFile;
            profileFd = mProfileFd;
            samplingInterval = mSamplingInterval;
            profileAutoStop = mAutoStopProfiler;
        }
        boolean enableOpenGlTrace = false;
        if (mOpenGlTraceApp != null && mOpenGlTraceApp.equals(processName)) {
            enableOpenGlTrace = true;
            mOpenGlTraceApp = null;
        }

        // If the app is being launched for restore or full backup, set it up specially
        boolean isRestrictedBackupMode = false;
        if (mBackupTarget != null && mBackupAppName.equals(processName)) {
            isRestrictedBackupMode = (mBackupTarget.backupMode == BackupRecord.RESTORE)
                    || (mBackupTarget.backupMode == BackupRecord.RESTORE_FULL)
                    || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL);
        }

        ensurePackageDexOpt(app.instrumentationInfo != null
                ? app.instrumentationInfo.packageName
                : app.info.packageName);
        if (app.instrumentationClass != null) {
            ensurePackageDexOpt(app.instrumentationClass.getPackageName());
        }
        if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Binding proc "
                + processName + " with config " + mConfiguration);
        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        app.compat = compatibilityInfoForPackageLocked(appInfo);
        if (profileFd != null) {
            profileFd = profileFd.dup();
        }
        ProfilerInfo profilerInfo = profileFile == null ? null
                : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
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

    // Remove this record from the list of starting applications.
    mPersistentStartingProcesses.remove(app);
    if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG_PROCESSES,
            "Attach application locked removing on hold: " + app);
    mProcessesOnHold.remove(app);

    boolean badApp = false;
    boolean didSomething = false;

    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }

    // Find any services that should be running in this process...
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
            badApp = true;
        }
    }

    // Check if a next-broadcast receiver is in this process...
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            // If the app died trying to launch the receiver we declare it 'bad'
            Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
            badApp = true;
        }
    }

    // Check whether the next backup agent is in this process...
    if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
        if (DEBUG_BACKUP) Slog.v(TAG_BACKUP,
                "New app is backup target, launching agent for " + app);
        ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
        try {
            thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                    compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                    mBackupTarget.backupMode);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
            badApp = true;
        }
    }

    if (badApp) {
        app.kill("error during init", true);
        handleAppDiedLocked(app, false, true);
        return false;
    }

    if (!didSomething) {
        updateOomAdjLocked();
    }

    return true;
}
```

实现方法是 __ActivityThread.ApplicationThread.bindApplication__

```java
public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
        Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
        Bundle coreSettings) {

    if (services != null) {
        // Setup the service cache in the ServiceManager
        ServiceManager.initServiceCache(services);
    }

    setCoreSettings(coreSettings);

    /*
     * Two possible indications that this package could be
     * sharing its runtime with other packages:
     *
     * 1.) the sharedUserId attribute is set in the manifest,
     *     indicating a request to share a VM with other
     *     packages with the same sharedUserId.
     *
     * 2.) the application element of the manifest has an
     *     attribute specifying a non-default process name,
     *     indicating the desire to run in another packages VM.
     *
     * If sharing is enabled we do not have a unique application
     * in a process and therefore cannot rely on the package
     * name inside the runtime.
     */
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

        // Tell the VMRuntime about the application, unless it is shared
        // inside a process.
        if (!sharable) {
            VMRuntime.registerAppInfo(appInfo.packageName, appInfo.dataDir,
                                    appInfo.processName);
        }
    }

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
    sendMessage(H.BIND_APPLICATION, data);
}
```

```java
private class H extends Handler
```

回调 __H__ 的方法 __handleMessage(Message msg__ 

```java
public void handleMessage(Message msg) {
    switch (msg.what) {
        .....

        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        
        .....
    }
}
```

ActivityThread

```java
private void handleBindApplication(AppBindData data) {
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);

    mProfiler = new Profiler();
    if (data.initProfilerInfo != null) {
        mProfiler.profileFile = data.initProfilerInfo.profileFile;
        mProfiler.profileFd = data.initProfilerInfo.profileFd;
        mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
        mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
    }

    // send up app name; do this *before* waiting for debugger
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
                                            UserHandle.myUserId());

    if (data.persistent) {
        // Persistent processes on low-memory devices do not get to
        // use hardware accelerated drawing, since this can add too much
        // overhead to the process.
        if (!ActivityManager.isHighEndGfx()) {
            HardwareRenderer.disable(false);
        }
    }

    if (mProfiler.profileFd != null) {
        mProfiler.startProfiling();
    }

    // If the app is Honeycomb MR1 or earlier, switch its AsyncTask
    // implementation to use the pool executor.  Normally, we use the
    // serialized executor as the default. This has to happen in the
    // main thread so the main looper is set right.
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }

    Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

    /*
     * Before spawning a new process, reset the time zone to be the system time zone.
     * This needs to be done because the system time zone could have changed after the
     * the spawning of this process. Without doing this this process would have the incorrect
     * system time zone.
     */
    TimeZone.setDefault(null);

    /*
     * Initialize the default locale in this process for the reasons we set the time zone.
     */
    Locale.setDefault(data.config.locale);

    /*
     * Update the system configuration since its preloaded and might not
     * reflect configuration changes. The configuration object passed
     * in AppBindData can be safely assumed to be up to date
     */
    mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
    mCurDefaultDisplayDpi = data.config.densityDpi;
    applyCompatConfiguration(mCurDefaultDisplayDpi);

    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

    /**
     * Switch this process to density compatibility mode if needed.
     */
    if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
            == 0) {
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    updateDefaultDensity();

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


    final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
    DateFormat.set24HourTimePref(is24Hr);

    View.mDebugViewAttributes =
            mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;

    /**
     * For system applications on userdebug/eng builds, log stack
     * traces of disk and network access to dropbox for analysis.
     */
    if ((data.appInfo.flags &
         (ApplicationInfo.FLAG_SYSTEM |
          ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
        StrictMode.conditionallyEnableDebugLogging();
    }

    /**
     * For apps targetting SDK Honeycomb or later, we don't allow
     * network usage on the main event loop / UI thread.
     *
     * Note to those grepping:  this is what ultimately throws
     * NetworkOnMainThreadException ...
     */
    if (data.appInfo.targetSdkVersion > 9) {
        StrictMode.enableDeathOnNetwork();
    }

    NetworkSecurityPolicy.getInstance().setCleartextTrafficPermitted(
            (data.appInfo.flags & ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC) != 0);

    if (data.debugMode != IApplicationThread.DEBUG_OFF) {
        // XXX should have option to change the port.
        Debug.changeDebugPort(8100);
        if (data.debugMode == IApplicationThread.DEBUG_WAIT) {
            Slog.w(TAG, "Application " + data.info.getPackageName()
                  + " is waiting for the debugger on port 8100...");

            IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.showWaitingForDebugger(mAppThread, true);
            } catch (RemoteException ex) {
            }

            Debug.waitForDebugger();

            try {
                mgr.showWaitingForDebugger(mAppThread, false);
            } catch (RemoteException ex) {
            }

        } else {
            Slog.w(TAG, "Application " + data.info.getPackageName()
                  + " can be debugged on port 8100...");
        }
    }

    // Enable OpenGL tracing if required
    if (data.enableOpenGlTrace) {
        GLUtils.setTracingLevel(1);
    }

    // Allow application-generated systrace messages if we're debuggable.
    boolean appTracingAllowed = (data.appInfo.flags&ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    Trace.setAppTracingAllowed(appTracingAllowed);

    /**
     * Initialize the default http proxy in this process for the reasons we set the time zone.
     */
    IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
    if (b != null) {
        // In pre-boot mode (doing initial launch to collect password), not
        // all system is up.  This includes the connectivity service, so don't
        // crash if we can't get it.
        IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
        try {
            final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
            Proxy.setHttpProxySystemProperty(proxyInfo);
        } catch (RemoteException e) {}
    }

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
            java.lang.ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)
                cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            throw new RuntimeException(
                "Unable to instantiate instrumentation "
                + data.instrumentationName + ": " + e.toString(), e);
        }

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

    if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
    } else {
        // Small heap, clamp to the current growth limit and let the heap release
        // pages after the growth limit to the non growth limit capacity. b/18387825
        dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
    }

    // Allow disk access during application and provider setup. This could
    // block processing ordered broadcasts, but later processing would
    // probably end up doing the same disk access.
    final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        if (!data.restrictedBackupMode) {
            List<ProviderInfo> providers = data.providers;
            if (providers != null) {
                installContentProviders(app, providers);
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
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```

```java
/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {
    .....
    
    /**
     * Map from package name, to preference name, to cached preferences.
     */
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

## 参考链接

- https://stackoverflow.com/questions/4693387/sharedpreferences-and-thread-safety