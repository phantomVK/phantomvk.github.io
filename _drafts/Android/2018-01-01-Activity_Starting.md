---
layout:     post
title:      "图解Activity启动流程"
date:       2018-07-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、概述

## 二、启动流程

#### 2.1 Activity.startActivity

```java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

/**
 * Launch a new activity.  You will not receive any information about when
 * the activity exits.  This implementation overrides the base version,
 * providing information about
 * the activity performing the launch.  Because of this additional
 * information, the {@link Intent#FLAG_ACTIVITY_NEW_TASK} launch flag is not
 * required; if not specified, the new activity will be added to the
 * task of the caller.
 *
 * <p>This method throws {@link android.content.ActivityNotFoundException}
 * if there was no Activity found to run the given Intent.
 *
 * @param intent The intent to start.
 * @param options Additional options for how the Activity should be started.
 * See {@link android.content.Context#startActivity(Intent, Bundle)
 * Context.startActivity(Intent, Bundle)} for more details.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see {@link #startActivity(Intent)}
 * @see #startActivityForResult
 */
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

#### 2.2 Activity.startActivityForResult

__startActivity()__ 方法通过 __requestCode = -1__ 的参数调用 __startActivityForResult()__ 方法

```java
/**
 * Same as calling {@link #startActivityForResult(Intent, int, Bundle)}
 * with no options.
 *
 * @param intent The intent to start.
 * @param requestCode If >= 0, this code will be returned in
 *                    onActivityResult() when the activity exits.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see #startActivity
 */
public void startActivityForResult(Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}

/**
 * Launch an activity for which you would like a result when it finished.
 * When this activity exits, your
 * onActivityResult() method will be called with the given requestCode.
 * Using a negative requestCode is the same as calling
 * {@link #startActivity} (the activity is not launched as a sub-activity).
 *
 * <p>Note that this method should only be used with Intent protocols
 * that are defined to return a result.  In other protocols (such as
 * {@link Intent#ACTION_MAIN} or {@link Intent#ACTION_VIEW}), you may
 * not get the result when you expect.  For example, if the activity you
 * are launching uses the singleTask launch mode, it will not run in your
 * task and thus you will immediately receive a cancel result.
 *
 * <p>As a special case, if you call startActivityForResult() with a requestCode
 * >= 0 during the initial onCreate(Bundle savedInstanceState)/onResume() of your
 * activity, then your window will not be displayed until a result is
 * returned back from the started activity.  This is to avoid visible
 * flickering when redirecting to another activity.
 *
 * <p>This method throws {@link android.content.ActivityNotFoundException}
 * if there was no Activity found to run the given Intent.
 *
 * @param intent The intent to start.
 * @param requestCode If >= 0, this code will be returned in
 *                    onActivityResult() when the activity exits.
 * @param options Additional options for how the Activity should be started.
 * See {@link android.content.Context#startActivity(Intent, Bundle)
 * Context.startActivity(Intent, Bundle)} for more details.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see #startActivity
 */
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {
        // 看小结3.1的解析，mToken是IBinder对象
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        
        // requestCode表示Activity使用完毕后有返回结果
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
    } else {
        // requestCode为0，默认
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

#### 2.3 Instrumentation.execStartActivities

类似方法 __execStartActivity(Context, IBinder, IBinder, Activity, Intent, int, Bundle)__，但接受一个将要被启动 __activities__ 的数组。

```java
/**
 * Like {@link #execStartActivity(Context, IBinder, IBinder, Activity, Intent, int, Bundle)},
 * but accepts an array of activities to be started.  Note that active
 * {@link ActivityMonitor} objects only match against the first activity in
 * the array.
 *
 * {@hide}
 */
public void execStartActivities(Context who, IBinder contextThread,
        IBinder token, Activity target, Intent[] intents, Bundle options) {
    execStartActivitiesAsUser(who, contextThread, token, target, intents, options,
            UserHandle.myUserId());
}

/**
 * Like {@link #execStartActivity(Context, IBinder, IBinder, Activity, Intent, int, Bundle)},
 * but accepts an array of activities to be started.  Note that active
 * {@link ActivityMonitor} objects only match against the first activity in
 * the array.
 *
 * {@hide}
 */
public void execStartActivitiesAsUser(Context who, IBinder contextThread,
        IBinder token, Activity target, Intent[] intents, Bundle options,
        int userId) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                if (am.match(who, null, intents[0])) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return;
                    }
                    break;
                }
            }
        }
    }
    try {
        String[] resolvedTypes = new String[intents.length];
        for (int i=0; i<intents.length; i++) {
            intents[i].migrateExtraStreamToClipData();
            intents[i].prepareToLeaveProcess();
            resolvedTypes[i] = intents[i].resolveTypeIfNeeded(who.getContentResolver());
        }
        // 看小节2.4
        int result = ActivityManagerNative.getDefault()
            .startActivities(whoThread, who.getBasePackageName(), intents, resolvedTypes,
                    token, options, userId);
        // 检查Activity启动结果
        checkStartActivityResult(result, intents[0]);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
}
```

#### 2.4 ActivityManagerNative.getDefault

__ActivityManagerNative.getDefault()__ 返回 __ActivityManagerProxy__ 实例

```java
/**
 * Retrieve the system's default/global activity manager.
 */
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

#### 2.5 ActivityManagerProxy

__ActivityManagerProxy __ 是 __ActivityManagerNative__ 的内部类 

```java
class ActivityManagerProxy implements IActivityManager {
    .....

    // caller: 当前应用ApplicationThread的对象mAppThread
    // callingPackage: ContextImpl.getBasePackageName()获取当前Activity所在包名
    // intent: 启动Activity是设置的Intent实例
    // resolvedType: 从Intent.resolveTypeIfNeeded()获得
    // resultTo: 当前Activity.token
    // resultWho: 当前Activity.mEmbeddedID
    // requestCode: 启动Activity时设置的requestCode
    // startFlags: 0
    // profilerInfo: null
    // options: 可选Bundle参数，一般为空
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        // 进入小节2.6
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    
    ....
}
```

#### 2.6 ActivityManagerNative

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    case START_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        String callingPackage = data.readString();
        Intent intent = Intent.CREATOR.createFromParcel(data);
        String resolvedType = data.readString();
        IBinder resultTo = data.readStrongBinder();
        String resultWho = data.readString();
        int requestCode = data.readInt();
        int startFlags = data.readInt();
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        Bundle options = data.readInt() != 0
                ? Bundle.CREATOR.createFromParcel(data) : null;
        // 到小节2.7
        int result = startActivity(app, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
        reply.writeNoException();
        reply.writeInt(result);
        return true;
    }

    .....
}
```

#### 2.7 ActivityManagerService.startActivity

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) {
    // 进入下面的重载方法中
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, options,
        UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
            false, ALLOW_FULL_ONLY, "startActivity", null);
    // 看小节2.8
    return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, options, false, userId, null, null);
}
```

#### 2.8 ActivityStackSupervisor.startActivityMayWait

```java
// caller: 
// callingUid: 
// intent: 
// resolvedType: 
// voiceSession: 
// voiceInteractor: 
// resultTo: 
// resultWho: 
// requestCode: 
// startFlags: 
// profilerInfo: 
// outResult: 
// config: 
// options: 
// ignoreTargetSecurity: 
// userId: 
// iContainer: 
// inTask: 
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
        Bundle options, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }
    boolean componentSpecified = intent.getComponent() != null;

    // 创建一个参数与客户端Intent相同的新Intent，避免修改客户端对象的配置
    intent = new Intent(intent);

    // Collect information about the target of the Intent.
    ActivityInfo aInfo =
            resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);

    ActivityContainer container = (ActivityContainer)iContainer;
    synchronized (mService) {
        if (container != null && container.mParentActivity != null &&
                container.mParentActivity.state != RESUMED) {
            // Cannot start a child activity if the parent is not resumed.
            return ActivityManager.START_CANCELED;
        }
        final int realCallingPid = Binder.getCallingPid();
        final int realCallingUid = Binder.getCallingUid();
        int callingPid;
        if (callingUid >= 0) {
            callingPid = -1;
        } else if (caller == null) {
            callingPid = realCallingPid;
            callingUid = realCallingUid;
        } else {
            callingPid = callingUid = -1;
        }

        final ActivityStack stack;
        if (container == null || container.mStack.isOnHomeDisplay()) {
            stack = mFocusedStack;
        } else {
            stack = container.mStack;
        }
        stack.mConfigWillChange = config != null && mService.mConfiguration.diff(config) != 0;
        if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                "Starting activity when config will change = " + stack.mConfigWillChange);

        final long origId = Binder.clearCallingIdentity();

        if (aInfo != null &&
                (aInfo.applicationInfo.privateFlags
                        &ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            // This may be a heavy-weight process!  Check to see if we already
            // have another, different heavy-weight process running.
            if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                if (mService.mHeavyWeightProcess != null &&
                        (mService.mHeavyWeightProcess.info.uid != aInfo.applicationInfo.uid ||
                        !mService.mHeavyWeightProcess.processName.equals(aInfo.processName))) {
                    int appCallingUid = callingUid;
                    if (caller != null) {
                        ProcessRecord callerApp = mService.getRecordForAppLocked(caller);
                        if (callerApp != null) {
                            appCallingUid = callerApp.info.uid;
                        } else {
                            Slog.w(TAG, "Unable to find app for caller " + caller
                                  + " (pid=" + callingPid + ") when starting: "
                                  + intent.toString());
                            ActivityOptions.abort(options);
                            return ActivityManager.START_PERMISSION_DENIED;
                        }
                    }

                    IIntentSender target = mService.getIntentSenderLocked(
                            ActivityManager.INTENT_SENDER_ACTIVITY, "android",
                            appCallingUid, userId, null, null, 0, new Intent[] { intent },
                            new String[] { resolvedType }, PendingIntent.FLAG_CANCEL_CURRENT
                            | PendingIntent.FLAG_ONE_SHOT, null);

                    Intent newIntent = new Intent();
                    if (requestCode >= 0) {
                        // Caller is requesting a result.
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_HAS_RESULT, true);
                    }
                    newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_INTENT,
                            new IntentSender(target));
                    if (mService.mHeavyWeightProcess.activities.size() > 0) {
                        ActivityRecord hist = mService.mHeavyWeightProcess.activities.get(0);
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_APP,
                                hist.packageName);
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_TASK,
                                hist.task.taskId);
                    }
                    newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_NEW_APP,
                            aInfo.packageName);
                    newIntent.setFlags(intent.getFlags());
                    newIntent.setClassName("android",
                            HeavyWeightSwitcherActivity.class.getName());
                    intent = newIntent;
                    resolvedType = null;
                    caller = null;
                    callingUid = Binder.getCallingUid();
                    callingPid = Binder.getCallingPid();
                    componentSpecified = true;
                    try {
                        ResolveInfo rInfo =
                            AppGlobals.getPackageManager().resolveIntent(
                                    intent, null,
                                    PackageManager.MATCH_DEFAULT_ONLY
                                    | ActivityManagerService.STOCK_PM_FLAGS, userId);
                        aInfo = rInfo != null ? rInfo.activityInfo : null;
                        aInfo = mService.getActivityInfoForUser(aInfo, userId);
                    } catch (RemoteException e) {
                        aInfo = null;
                    }
                }
            }
        }

        int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                voiceSession, voiceInteractor, resultTo, resultWho,
                requestCode, callingPid, callingUid, callingPackage,
                realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                componentSpecified, null, container, inTask);

        Binder.restoreCallingIdentity(origId);

        if (stack.mConfigWillChange) {
            // If the caller also wants to switch to a new configuration,
            // do so now.  This allows a clean switch, as we are waiting
            // for the current activity to pause (so we will not destroy
            // it), and have not yet started the next activity.
            mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                    "updateConfiguration()");
            stack.mConfigWillChange = false;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Updating to new configuration after starting activity.");
            mService.updateConfigurationLocked(config, null, false, false);
        }

        if (outResult != null) {
            outResult.result = res;
            if (res == ActivityManager.START_SUCCESS) {
                mWaitingActivityLaunched.add(outResult);
                do {
                    try {
                        mService.wait();
                    } catch (InterruptedException e) {
                    }
                } while (!outResult.timeout && outResult.who == null);
            } else if (res == ActivityManager.START_TASK_TO_FRONT) {
                ActivityRecord r = stack.topRunningActivityLocked(null);
                if (r.nowVisible && r.state == RESUMED) {
                    outResult.timeout = false;
                    outResult.who = new ComponentName(r.info.packageName, r.info.name);
                    outResult.totalTime = 0;
                    outResult.thisTime = 0;
                } else {
                    outResult.thisTime = SystemClock.uptimeMillis();
                    mWaitingActivityVisible.add(outResult);
                    do {
                        try {
                            mService.wait();
                        } catch (InterruptedException e) {
                        }
                    } while (!outResult.timeout && outResult.who == null);
                }
            }
        }

        return res;
    }
}
```

#### 2.9 ActivityStackSupervisor.resolveActivity

```java
ActivityInfo resolveActivity(Intent intent, String resolvedType, int startFlags,
        ProfilerInfo profilerInfo, int userId) {
    // Collect information about the target of the Intent.
    ActivityInfo aInfo;
    try {
        ResolveInfo rInfo =
            AppGlobals.getPackageManager().resolveIntent(
                    intent, resolvedType,
                    PackageManager.MATCH_DEFAULT_ONLY
                                | ActivityManagerService.STOCK_PM_FLAGS, userId);
        aInfo = rInfo != null ? rInfo.activityInfo : null;
    } catch (RemoteException e) {
        aInfo = null;
    }

    if (aInfo != null) {
        // Store the found target back into the intent, because now that
        // we have it we never want to do this again.  For example, if the
        // user navigates back to this point in the history, we should
        // always restart the exact same activity.
        intent.setComponent(new ComponentName(
                aInfo.applicationInfo.packageName, aInfo.name));

        // Don't debug things in the system process
        if ((startFlags&ActivityManager.START_FLAG_DEBUG) != 0) {
            if (!aInfo.processName.equals("system")) {
                mService.setDebugApp(aInfo.processName, true, false);
            }
        }

        if ((startFlags&ActivityManager.START_FLAG_OPENGL_TRACES) != 0) {
            if (!aInfo.processName.equals("system")) {
                mService.setOpenGlTraceApp(aInfo.applicationInfo, aInfo.processName);
            }
        }

        if (profilerInfo != null) {
            if (!aInfo.processName.equals("system")) {
                mService.setProfileApp(aInfo.applicationInfo, aInfo.processName, profilerInfo);
            }
        }
    }
    return aInfo;
}
```

####  2.10 AppGlobals.getPackageManager

```java
/**
 * Return the raw interface to the package manager.
 * @return The package manager.
 */
public static IPackageManager getPackageManager() {
    return ActivityThread.getPackageManager();
}
```

```java
public static IPackageManager getPackageManager() {
    if (sPackageManager != null) {
        //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
        return sPackageManager;
    }
    IBinder b = ServiceManager.getService("package");
    //Slog.v("PackageManager", "default service binder = " + b);
    sPackageManager = IPackageManager.Stub.asInterface(b);
    //Slog.v("PackageManager", "default service = " + sPackageManager);
    return sPackageManager;
}
```

获取 ApplicationPackageManager

#### 2.11 PackageManagerService.resolveIntent

```java
@Override
public ResolveInfo resolveIntent(Intent intent, String resolvedType,
        int flags, int userId) {
    if (!sUserManager.exists(userId)) return null;
    enforceCrossUserPermission(Binder.getCallingUid(), userId, false, false, "resolve intent");
    List<ResolveInfo> query = queryIntentActivities(intent, resolvedType, flags, userId);
    return chooseBestActivity(intent, resolvedType, flags, query, userId);
}
```

#### 2.12 PackageManagerService.queryIntentActivities

```java
@Override
public List<ResolveInfo> queryIntentActivities(Intent intent,
        String resolvedType, int flags, int userId) {
    if (!sUserManager.exists(userId)) return Collections.emptyList();
    enforceCrossUserPermission(Binder.getCallingUid(), userId, false, false, "query intent activities");
    ComponentName comp = intent.getComponent();
    if (comp == null) {
        if (intent.getSelector() != null) {
            intent = intent.getSelector();
            comp = intent.getComponent();
        }
    }

    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
        // 获取ActivityInfo
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            final ResolveInfo ri = new ResolveInfo();
            ri.activityInfo = ai;
            list.add(ri);
        }
        return list;
    }

    // reader
    synchronized (mPackages) {
        final String pkgName = intent.getPackage();
        if (pkgName == null) {
            List<CrossProfileIntentFilter> matchingFilters =
                    getMatchingCrossProfileIntentFilters(intent, resolvedType, userId);
            // Check for results that need to skip the current profile.
            ResolveInfo xpResolveInfo  = querySkipCurrentProfileIntents(matchingFilters, intent,
                    resolvedType, flags, userId);
            if (xpResolveInfo != null && isUserEnabled(xpResolveInfo.targetUserId)) {
                List<ResolveInfo> result = new ArrayList<ResolveInfo>(1);
                result.add(xpResolveInfo);
                return filterIfNotPrimaryUser(result, userId);
            }

            // Check for results in the current profile.
            List<ResolveInfo> result = mActivities.queryIntent(
                    intent, resolvedType, flags, userId);

            // Check for cross profile results.
            xpResolveInfo = queryCrossProfileIntents(
                    matchingFilters, intent, resolvedType, flags, userId);
            if (xpResolveInfo != null && isUserEnabled(xpResolveInfo.targetUserId)) {
                result.add(xpResolveInfo);
                Collections.sort(result, mResolvePrioritySorter);
            }
            result = filterIfNotPrimaryUser(result, userId);
            if (hasWebURI(intent)) {
                CrossProfileDomainInfo xpDomainInfo = null;
                final UserInfo parent = getProfileParent(userId);
                if (parent != null) {
                    xpDomainInfo = getCrossProfileDomainPreferredLpr(intent, resolvedType,
                            flags, userId, parent.id);
                }
                if (xpDomainInfo != null) {
                    if (xpResolveInfo != null) {
                        // If we didn't remove it, the cross-profile ResolveInfo would be twice
                        // in the result.
                        result.remove(xpResolveInfo);
                    }
                    if (result.size() == 0) {
                        result.add(xpDomainInfo.resolveInfo);
                        return result;
                    }
                } else if (result.size() <= 1) {
                    return result;
                }
                result = filterCandidatesWithDomainPreferredActivitiesLPr(intent, flags, result,
                        xpDomainInfo, userId);
                Collections.sort(result, mResolvePrioritySorter);
            }
            return result;
        }
        final PackageParser.Package pkg = mPackages.get(pkgName);
        if (pkg != null) {
            return filterIfNotPrimaryUser(
                    mActivities.queryIntentForPackage(
                            intent, resolvedType, flags, pkg.activities, userId),
                    userId);
        }
        return new ArrayList<ResolveInfo>();
    }
}
```

#### 2.13 ActivityStackSupervisor.startActivityLocked

```java
final int startActivityLocked(IApplicationThread caller,
        Intent intent, String resolvedType, ActivityInfo aInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode,
        int callingPid, int callingUid, String callingPackage,
        int realCallingPid, int realCallingUid, int startFlags, Bundle options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        ActivityContainer container, TaskRecord inTask) {
    int err = ActivityManager.START_SUCCESS;

    // 获取调用者的进程记录对象
    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }

    final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        // 获取调用者所在的Activity
        sourceRecord = isInAnyStackLocked(resultTo);
        if (sourceRecord != null) {
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }

    // 根据launchFlags启动Activity
    final int launchFlags = intent.getFlags();

    // 
    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
        // Transfer the result target from the source activity to the new
        // one being started, including any failures.
        // request != -1
        if (requestCode >= 0) {
            ActivityOptions.abort(options);
            return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
        }
        resultRecord = sourceRecord.resultTo;
        if (resultRecord != null && !resultRecord.isInStackLocked()) {
            resultRecord = null;
        }
        resultWho = sourceRecord.resultWho;
        requestCode = sourceRecord.requestCode;
        sourceRecord.resultTo = null;
        if (resultRecord != null) {
            resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
        }
        if (sourceRecord.launchedFromUid == callingUid) {
            // The new activity is being launched from the same uid as the previous
            // activity in the flow, and asking to forward its result back to the
            // previous.  In this case the activity is serving as a trampoline between
            // the two, so we also want to update its launchedFromPackage to be the
            // same as the previous activity.  Note that this is safe, since we know
            // these two packages come from the same uid; the caller could just as
            // well have supplied that same package name itself.  This specifially
            // deals with the case of an intent picker/chooser being launched in the app
            // flow to redirect to an activity picked by the user, where we want the final
            // activity to consider it to have been launched by the previous app activity.
            callingPackage = sourceRecord.launchedFromPackage;
        }
    }

    // 从Intent中无法找到相应的Component
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        // We couldn't find a class that can handle the given Intent.
        // That's the end of that!
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }

    // 找不到Intent指定的Activity类
    if (err == ActivityManager.START_SUCCESS && aInfo == null) {
        // We couldn't find the specific class specified in the Intent.
        // Also the end of the line.
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }

    // 尝试启动后台Activity, 但该Activity对用户不可见
    if (err == ActivityManager.START_SUCCESS
            && !isCurrentProfileLocked(userId)
            && (aInfo.flags & FLAG_SHOW_FOR_ALL_USERS) == 0) {
        // Trying to launch a background activity that doesn't show for all users.
        err = ActivityManager.START_NOT_CURRENT_USER_ACTIVITY;
    }

    if (err == ActivityManager.START_SUCCESS && sourceRecord != null
            && sourceRecord.task.voiceSession != null) {
        // If this activity is being launched as part of a voice session, we need
        // to ensure that it is safe to do so.  If the upcoming activity will also
        // be part of the voice session, we can only launch it if it has explicitly
        // said it supports the VOICE category, or it is a part of the calling app.
        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && sourceRecord.info.applicationInfo.uid != aInfo.applicationInfo.uid) {
            try {
                intent.addCategory(Intent.CATEGORY_VOICE);
                if (!AppGlobals.getPackageManager().activitySupportsIntent(
                        intent.getComponent(), intent, resolvedType)) {
                    Slog.w(TAG,
                            "Activity being started in current voice task does not support voice: "
                            + intent);
                    err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Failure checking voice capabilities", e);
                err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
            }
        }
    }

    if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
        // If the caller is starting a new voice session, just make sure the target
        // is actually allowing it to run this way.
        try {
            if (!AppGlobals.getPackageManager().activitySupportsIntent(intent.getComponent(),
                    intent, resolvedType)) {
                Slog.w(TAG,
                        "Activity being started in new voice task does not support: "
                        + intent);
                err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Failure checking voice capabilities", e);
            err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
        }
    }

    // resultStack为空
    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;

    if (err != ActivityManager.START_SUCCESS) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(-1,
                resultRecord, resultWho, requestCode,
                Activity.RESULT_CANCELED, null);
        }
        ActivityOptions.abort(options);
        return err;
    }

    boolean abort = false;

    final int startAnyPerm = mService.checkPermission(
            START_ANY_ACTIVITY, callingPid, callingUid);

    // 权限检查
    if (startAnyPerm != PERMISSION_GRANTED) {
        .....
    }

    abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
            callingPid, resolvedType, aInfo.applicationInfo);

    if (mService.mController != null) {
        try {
            // The Intent we give to the watcher has the extra data
            // stripped off, since it can contain private information.
            Intent watchIntent = intent.cloneFilter();
            abort |= !mService.mController.activityStarting(watchIntent,
                    aInfo.applicationInfo.packageName);
        } catch (RemoteException e) {
            mService.mController = null;
        }
    }

    if (abort) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                    Activity.RESULT_CANCELED, null);
        }
        // We pretend to the caller that it was really started, but
        // they will just get a cancel result.
        ActivityOptions.abort(options);
        return ActivityManager.START_SUCCESS;
    }

    ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
            intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
            requestCode, componentSpecified, voiceSession != null, this, container, options);
    if (outActivity != null) {
        outActivity[0] = r;
    }

    if (r.appTimeTracker == null && sourceRecord != null) {
        // If the caller didn't specify an explicit time tracker, we want to continue
        // tracking under any it has.
        r.appTimeTracker = sourceRecord.appTimeTracker;
    }

    final ActivityStack stack = mFocusedStack;
    if (voiceSession == null && (stack.mResumedActivity == null
            || stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
        if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                realCallingPid, realCallingUid, "Activity start")) {
            PendingActivityLaunch pal =
                    new PendingActivityLaunch(r, sourceRecord, startFlags, stack);
            mPendingActivityLaunches.add(pal);
            ActivityOptions.abort(options);
            return ActivityManager.START_SWITCHES_CANCELED;
        }
    }

    if (mService.mDidAppSwitch) {
        // This is the second allowed switch since we stopped switches,
        // so now just generally allow switches.  Use case: user presses
        // home (switches disabled, switch to home, mDidAppSwitch now true);
        // user taps a home icon (coming from home so allowed, we hit here
        // and now allow anyone to switch again).
        mService.mAppSwitchesAllowedTime = 0;
    } else {
        mService.mDidAppSwitch = true;
    }

    // 启动PendingActivity
    doPendingActivityLaunchesLocked(false);

    err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
            startFlags, true, options, inTask);

    if (err < 0) {
        // If someone asked to have the keyguard dismissed on the next
        // activity start, but we are not actually doing an activity
        // switch...  just dismiss the keyguard now, because we
        // probably want to see whatever is behind it.
        notifyActivityDrawnForKeyguard();
    }
    return err;
}
```

#### 2.14 ActivityManagerService.checkAppSwitchAllowedLocked

```java
boolean checkAppSwitchAllowedLocked(int sourcePid, int sourceUid,
        int callingPid, int callingUid, String name) {
    if (mAppSwitchesAllowedTime < SystemClock.uptimeMillis()) {
        return true;
    }

    int perm = checkComponentPermission(
            android.Manifest.permission.STOP_APP_SWITCHES, sourcePid,
            sourceUid, -1, true);
    if (perm == PackageManager.PERMISSION_GRANTED) {
        return true;
    }

    // If the actual IPC caller is different from the logical source, then
    // also see if they are allowed to control app switches.
    if (callingUid != -1 && callingUid != sourceUid) {
        perm = checkComponentPermission(
                android.Manifest.permission.STOP_APP_SWITCHES, callingPid,
                callingUid, -1, true);
        if (perm == PackageManager.PERMISSION_GRANTED) {
            return true;
        }
    }

    Slog.w(TAG, name + " request from " + sourceUid + " stopped");
    return false;
}
```

#### 2.15 ActivityStackSupervisor.doPendingActivityLaunchesLocked

```java
final void doPendingActivityLaunchesLocked(boolean doResume) {
    while (!mPendingActivityLaunches.isEmpty()) {
        PendingActivityLaunch pal = mPendingActivityLaunches.remove(0);
        startActivityUncheckedLocked(pal.r, pal.sourceRecord, null, null, pal.startFlags,
                doResume && mPendingActivityLaunches.isEmpty(), null, null);
    }
}
```

#### 2.16 ActivityStackSupervisor.startActivityUncheckedLocked

```java
final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
        boolean doResume, Bundle options, TaskRecord inTask) {
    final Intent intent = r.intent;
    final int callingUid = r.launchedFromUid;

    // In some flows in to this function, we retrieve the task record and hold on to it
    // without a lock before calling back in to here...  so the task at this point may
    // not actually be in recents.  Check for that, and if it isn't in recents just
    // consider it invalid.
    if (inTask != null && !inTask.inRecents) {
        Slog.w(TAG, "Starting activity in task not in recents: " + inTask);
        inTask = null;
    }

    // 获取Activity启动模式
    final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
    final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
    final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;

    int launchFlags = intent.getFlags();
    if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 &&
            (launchSingleInstance || launchSingleTask)) {
        // We have a conflict between the Intent and the Activity manifest, manifest wins.
        launchFlags &=
                ~(Intent.FLAG_ACTIVITY_NEW_DOCUMENT | Intent.FLAG_ACTIVITY_MULTIPLE_TASK);
    } else {
        switch (r.info.documentLaunchMode) {
            case ActivityInfo.DOCUMENT_LAUNCH_NONE:
                break;
            case ActivityInfo.DOCUMENT_LAUNCH_INTO_EXISTING:
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                break;
            case ActivityInfo.DOCUMENT_LAUNCH_ALWAYS:
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                break;
            case ActivityInfo.DOCUMENT_LAUNCH_NEVER:
                launchFlags &= ~Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
                break;
        }
    }

    final boolean launchTaskBehind = r.mLaunchTaskBehind
            && !launchSingleTask && !launchSingleInstance
            && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;

    if (r.resultTo != null && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0
            && r.resultTo.task.stack != null) {
        // For whatever reason this activity is being launched into a new
        // task...  yet the caller has requested a result back.  Well, that
        // is pretty messed up, so instead immediately send back a cancel
        // and let the new task continue launched as normal without a
        // dependency on its originator.
        Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
        r.resultTo.task.stack.sendActivityResultLocked(-1,
                r.resultTo, r.resultWho, r.requestCode,
                Activity.RESULT_CANCELED, null);
        r.resultTo = null;
    }

    if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 && r.resultTo == null) {
        launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
    }

    // If we are actually going to launch in to a new task, there are some cases where
    // we further want to do multiple task.
    if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
        if (launchTaskBehind
                || r.info.documentLaunchMode == ActivityInfo.DOCUMENT_LAUNCH_ALWAYS) {
            launchFlags |= Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
        }
    }

    // We'll invoke onUserLeaving before onPause only if the launching
    // activity did not explicitly state that this is an automated launch.
    mUserLeaving = (launchFlags & Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;
    if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
            "startActivity() => mUserLeaving=" + mUserLeaving);

    // If the caller has asked not to resume at this point, we make note
    // of this in the record so that we can skip it when trying to find
    // the top running activity.
    // 不需要马上resume，标记为delayedResume
    if (!doResume) {
        r.delayedResume = true;
    }

    ActivityRecord notTop =
            (launchFlags & Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP) != 0 ? r : null;

    // If the onlyIfNeeded flag is set, then we can do this if the activity
    // being launched is the same as the one making the call...  or, as
    // a special case, if we do not know the caller then we count the
    // current top activity as the caller.
    if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
        ActivityRecord checkedCaller = sourceRecord;
        if (checkedCaller == null) {
            checkedCaller = mFocusedStack.topRunningNonDelayedActivityLocked(notTop);
        }
        if (!checkedCaller.realActivity.equals(r.realActivity)) {
            // Caller is not the same as launcher, so always needed.
            startFlags &= ~ActivityManager.START_FLAG_ONLY_IF_NEEDED;
        }
    }

    boolean addingToTask = false;
    TaskRecord reuseTask = null;

    // If the caller is not coming from another activity, but has given us an
    // explicit task into which they would like us to launch the new activity,
    // then let's see about doing that.
    if (sourceRecord == null && inTask != null && inTask.stack != null) {
        final Intent baseIntent = inTask.getBaseIntent();
        final ActivityRecord root = inTask.getRootActivity();
        if (baseIntent == null) {
            ActivityOptions.abort(options);
            throw new IllegalArgumentException("Launching into task without base intent: "
                    + inTask);
        }

        // If this task is empty, then we are adding the first activity -- it
        // determines the root, and must be launching as a NEW_TASK.
        if (launchSingleInstance || launchSingleTask) {
            if (!baseIntent.getComponent().equals(r.intent.getComponent())) {
                ActivityOptions.abort(options);
                throw new IllegalArgumentException("Trying to launch singleInstance/Task "
                        + r + " into different task " + inTask);
            }
            if (root != null) {
                ActivityOptions.abort(options);
                throw new IllegalArgumentException("Caller with inTask " + inTask
                        + " has root " + root + " but target is singleInstance/Task");
            }
        }

        // If task is empty, then adopt the interesting intent launch flags in to the
        // activity being started.
        if (root == null) {
            final int flagsOfInterest = Intent.FLAG_ACTIVITY_NEW_TASK
                    | Intent.FLAG_ACTIVITY_MULTIPLE_TASK | Intent.FLAG_ACTIVITY_NEW_DOCUMENT
                    | Intent.FLAG_ACTIVITY_RETAIN_IN_RECENTS;
            launchFlags = (launchFlags&~flagsOfInterest)
                    | (baseIntent.getFlags()&flagsOfInterest);
            intent.setFlags(launchFlags);
            inTask.setIntent(r);
            addingToTask = true;

        // If the task is not empty and the caller is asking to start it as the root
        // of a new task, then we don't actually want to start this on the task.  We
        // will bring the task to the front, and possibly give it a new intent.
        } else if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            addingToTask = false;

        } else {
            addingToTask = true;
        }

        reuseTask = inTask;
    } else {
        inTask = null;
    }

    if (inTask == null) {
        if (sourceRecord == null) {
            // This activity is not being started from another...  in this
            // case we -always- start a new task.
            // 新Activity不是由其他Activity启动的，因此创建新任务栈
            if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0 && inTask == null) {
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
        } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
            // The original activity who is starting us is running as a single
            // instance...  this new activity it is starting must go on its
            // own task.
            // 源Activity的任务栈类型是SINGLE_INSTANCE，则需要给新Activity创建新任务栈
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        } else if (launchSingleInstance || launchSingleTask) {
            // The activity being started is a single instance...  it always
            // gets launched into its own task.
            // 新Activity被启动为SingleInstance或SingleTask，需要给新Activity创建新任务栈
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }
    }

    ActivityInfo newTaskInfo = null;
    Intent newTaskIntent = null;
    ActivityStack sourceStack;
    if (sourceRecord != null) {
        // 源调用者处于finishing状态
        if (sourceRecord.finishing) {
            // If the source is finishing, we can't further count it as our source.  This
            // is because the task it is associated with may now be empty and on its way out,
            // so we don't want to blindly throw it in to that task.  Instead we will take
            // the NEW_TASK flow and try to find a task for it. But save the task information
            // so it can be used when creating the new task.
            if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                newTaskInfo = sourceRecord.info;
                newTaskIntent = sourceRecord.task.intent;
            }
            sourceRecord = null;
            sourceStack = null;
        } else {
            // 源调用者没有finishing，这把源调用者所处任务栈赋值给sourceStack
            sourceStack = sourceRecord.task.stack;
        }
    } else {
        sourceStack = null;
    }

    boolean movedHome = false;
    ActivityStack targetStack;

    intent.setFlags(launchFlags);
    final boolean noAnimation = (launchFlags & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0;

    // We may want to try to place the new activity in to an existing task.  We always
    // do this if the target activity is singleTask or singleInstance; we will also do
    // this if NEW_TASK has been requested, and there is not an additional qualifier telling
    // us to still place it in a new task: multi task, always doc mode, or being asked to
    // launch this as a new task behind the current one.
    if (((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
            (launchFlags & Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
            || launchSingleInstance || launchSingleTask) {
        // If bring to front is requested, and no result is requested and we have not
        // been given an explicit task to launch in to, and
        // we can find a task that was started with this same
        // component, then instead of launching bring that one to the front.
        if (inTask == null && r.resultTo == null) {
            // See if there is a task to bring to the front.  If this is
            // a SINGLE_INSTANCE activity, there can be one and only one
            // instance of it in the history, and it is always in its own
            // unique task, so we do a special search.
            ActivityRecord intentActivity = !launchSingleInstance ?
                    findTaskLocked(r) : findActivityLocked(intent, r.info);
            if (intentActivity != null) {
                // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused
                // but still needs to be a lock task mode violation since the task gets
                // cleared out and the device would otherwise leave the locked task.
                if (isLockTaskModeViolation(intentActivity.task,
                        (launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
                    showLockTaskToast();
                    Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
                    return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
                }
                if (r.task == null) {
                    r.task = intentActivity.task;
                }
                if (intentActivity.task.intent == null) {
                    // This task was started because of movement of
                    // the activity based on affinity...  now that we
                    // are actually launching it, we can assign the
                    // base intent.
                    intentActivity.task.setIntent(r);
                }
                targetStack = intentActivity.task.stack;
                targetStack.mLastPausedActivity = null;
                // If the target task is not in the front, then we need
                // to bring it to the front...  except...  well, with
                // SINGLE_TASK_LAUNCH it's not entirely clear.  We'd like
                // to have the same behavior as if a new instance was
                // being started, which means not bringing it to the front
                // if the caller is not itself in the front.
                final ActivityStack focusStack = getFocusedStack();
                ActivityRecord curTop = (focusStack == null)
                        ? null : focusStack.topRunningNonDelayedActivityLocked(notTop);
                boolean movedToFront = false;
                if (curTop != null && (curTop.task != intentActivity.task ||
                        curTop.task != focusStack.topTask())) {
                    r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                    if (sourceRecord == null || (sourceStack.topActivity() != null &&
                            sourceStack.topActivity().task == sourceRecord.task)) {
                        // We really do want to push this one into the user's face, right now.
                        if (launchTaskBehind && sourceRecord != null) {
                            intentActivity.setTaskToAffiliateWith(sourceRecord.task);
                        }
                        movedHome = true;
                        targetStack.moveTaskToFrontLocked(intentActivity.task, noAnimation,
                                options, r.appTimeTracker, "bringingFoundTaskToFront");
                        movedToFront = true;
                        if ((launchFlags &
                                (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                                == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                            // Caller wants to appear on home activity.
                            intentActivity.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                        }
                        options = null;
                    }
                }
                if (!movedToFront) {
                    if (DEBUG_TASKS) Slog.d(TAG_TASKS, "Bring to front target: " + targetStack
                            + " from " + intentActivity);
                    targetStack.moveToFront("intentActivityFound");
                }

                // If the caller has requested that the target task be
                // reset, then do so.
                if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                }
                if ((startFlags & ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                    // We don't need to start a new activity, and
                    // the client said not to do anything if that
                    // is the case, so this is it!  And for paranoia, make
                    // sure we have correctly resumed the top activity.
                    if (doResume) {
                        resumeTopActivitiesLocked(targetStack, null, options);

                        // Make sure to notify Keyguard as well if we are not running an app
                        // transition later.
                        if (!movedToFront) {
                            notifyActivityDrawnForKeyguard();
                        }
                    } else {
                        ActivityOptions.abort(options);
                    }
                    return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                }
                if ((launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK)) {
                    // The caller has requested to completely replace any
                    // existing task with its new activity.  Well that should
                    // not be too hard...
                    reuseTask = intentActivity.task;
                    reuseTask.performClearTaskLocked();
                    reuseTask.setIntent(r);
                } else if ((launchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                        || launchSingleInstance || launchSingleTask) {
                    // In this situation we want to remove all activities
                    // from the task up to the one being started.  In most
                    // cases this means we are resetting the task to its
                    // initial state.
                    ActivityRecord top =
                            intentActivity.task.performClearTaskLocked(r, launchFlags);
                    if (top != null) {
                        if (top.frontOfTask) {
                            // Activity aliases may mean we use different
                            // intents for the top activity, so make sure
                            // the task now has the identity of the new
                            // intent.
                            top.task.setIntent(r);
                        }
                        ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT,
                                r, top.task);
                        top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    } else {
                        // A special case: we need to start the activity because it is not
                        // currently running, and the caller has asked to clear the current
                        // task to have this activity at the top.
                        addingToTask = true;
                        // Now pretend like this activity is being started by the top of its
                        // task, so it is put in the right place.
                        sourceRecord = intentActivity;
                        TaskRecord task = sourceRecord.task;
                        if (task != null && task.stack == null) {
                            // Target stack got cleared when we all activities were removed
                            // above. Go ahead and reset it.
                            targetStack = computeStackFocus(sourceRecord, false /* newTask */);
                            targetStack.addTask(
                                    task, !launchTaskBehind /* toTop */, false /* moving */);
                        }

                    }
                } else if (r.realActivity.equals(intentActivity.task.realActivity)) {
                    // In this case the top activity on the task is the
                    // same as the one being launched, so we take that
                    // as a request to bring the task to the foreground.
                    // If the top activity in the task is the root
                    // activity, deliver this new intent to it if it
                    // desires.
                    if (((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0 || launchSingleTop)
                            && intentActivity.realActivity.equals(r.realActivity)) {
                        ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r,
                                intentActivity.task);
                        if (intentActivity.frontOfTask) {
                            intentActivity.task.setIntent(r);
                        }
                        intentActivity.deliverNewIntentLocked(callingUid, r.intent,
                                r.launchedFromPackage);
                    } else if (!r.intent.filterEquals(intentActivity.task.intent)) {
                        // In this case we are launching the root activity
                        // of the task, but with a different intent.  We
                        // should start a new instance on top.
                        addingToTask = true;
                        sourceRecord = intentActivity;
                    }
                } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                    // In this case an activity is being launched in to an
                    // existing task, without resetting that task.  This
                    // is typically the situation of launching an activity
                    // from a notification or shortcut.  We want to place
                    // the new activity on top of the current task.
                    addingToTask = true;
                    sourceRecord = intentActivity;
                } else if (!intentActivity.task.rootWasReset) {
                    // In this case we are launching in to an existing task
                    // that has not yet been started from its front door.
                    // The current task has been brought to the front.
                    // Ideally, we'd probably like to place this new task
                    // at the bottom of its stack, but that's a little hard
                    // to do with the current organization of the code so
                    // for now we'll just drop it.
                    intentActivity.task.setIntent(r);
                }
                if (!addingToTask && reuseTask == null) {
                    // We didn't do anything...  but it was needed (a.k.a., client
                    // don't use that intent!)  And for paranoia, make
                    // sure we have correctly resumed the top activity.
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null, options);
                        if (!movedToFront) {
                            // Make sure to notify Keyguard as well if we are not running an app
                            // transition later.
                            notifyActivityDrawnForKeyguard();
                        }
                    } else {
                        ActivityOptions.abort(options);
                    }
                    return ActivityManager.START_TASK_TO_FRONT;
                }
            }
        }
    }

    //String uri = r.intent.toURI();
    //Intent intent2 = new Intent(uri);
    //Slog.i(TAG, "Given intent: " + r.intent);
    //Slog.i(TAG, "URI is: " + uri);
    //Slog.i(TAG, "To intent: " + intent2);

    if (r.packageName != null) {
        // If the activity being launched is the same as the one currently
        // at the top, then we need to check if it should only be launched
        // once.
        ActivityStack topStack = mFocusedStack;
        ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);
        if (top != null && r.resultTo == null) {
            if (top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                if (top.app != null && top.app.thread != null) {
                    if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                        || launchSingleTop || launchSingleTask) {
                        ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top,
                                top.task);
                        // For paranoia, make sure we have correctly
                        // resumed the top activity.
                        topStack.mLastPausedActivity = null;
                        if (doResume) {
                            resumeTopActivitiesLocked();
                        }
                        ActivityOptions.abort(options);
                        if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                            // We don't need to start a new activity, and
                            // the client said not to do anything if that
                            // is the case, so this is it!
                            return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                        }
                        top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                        return ActivityManager.START_DELIVERED_TO_TOP;
                    }
                }
            }
        }

    } else {
        if (r.resultTo != null && r.resultTo.task.stack != null) {
            r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho,
                    r.requestCode, Activity.RESULT_CANCELED, null);
        }
        ActivityOptions.abort(options);
        return ActivityManager.START_CLASS_NOT_FOUND;
    }

    boolean newTask = false;
    boolean keepCurTransition = false;

    TaskRecord taskToAffiliate = launchTaskBehind && sourceRecord != null ?
            sourceRecord.task : null;

    // Should this be considered a new task?
    if (r.resultTo == null && inTask == null && !addingToTask
            && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        targetStack = computeStackFocus(r, newTask);
        targetStack.moveToFront("startingNewTask");

        if (reuseTask == null) {
            r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                    newTaskInfo != null ? newTaskInfo : r.info,
                    newTaskIntent != null ? newTaskIntent : intent,
                    voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                    taskToAffiliate);
            if (DEBUG_TASKS) Slog.v(TAG_TASKS,
                    "Starting new activity " + r + " in new task " + r.task);
        } else {
            r.setTask(reuseTask, taskToAffiliate);
        }
        if (isLockTaskModeViolation(r.task)) {
            Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
            return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
        if (!movedHome) {
            if ((launchFlags &
                    (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                    == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                // Caller wants to appear on home activity, so before starting
                // their own activity we will bring home to the front.
                r.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
            }
        }
    } else if (sourceRecord != null) {
        final TaskRecord sourceTask = sourceRecord.task;
        if (isLockTaskModeViolation(sourceTask)) {
            Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
            return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
        targetStack = sourceTask.stack;
        targetStack.moveToFront("sourceStackToFront");
        final TaskRecord topTask = targetStack.topTask();
        if (topTask != sourceTask) {
            targetStack.moveTaskToFrontLocked(sourceTask, noAnimation, options,
                    r.appTimeTracker, "sourceTaskToFront");
        }
        if (!addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
            // In this case, we are adding the activity to an existing
            // task, but the caller has asked to clear that task if the
            // activity is already running.
            ActivityRecord top = sourceTask.performClearTaskLocked(r, launchFlags);
            keepCurTransition = true;
            if (top != null) {
                ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                // For paranoia, make sure we have correctly
                // resumed the top activity.
                targetStack.mLastPausedActivity = null;
                if (doResume) {
                    targetStack.resumeTopActivityLocked(null);
                }
                ActivityOptions.abort(options);
                return ActivityManager.START_DELIVERED_TO_TOP;
            }
        } else if (!addingToTask &&
                (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
            // In this case, we are launching an activity in our own task
            // that may already be running somewhere in the history, and
            // we want to shuffle it to the front of the stack if so.
            final ActivityRecord top = sourceTask.findActivityInHistoryLocked(r);
            if (top != null) {
                final TaskRecord task = top.task;
                task.moveActivityToFrontLocked(top);
                ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, task);
                top.updateOptionsLocked(options);
                top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                targetStack.mLastPausedActivity = null;
                if (doResume) {
                    targetStack.resumeTopActivityLocked(null);
                }
                return ActivityManager.START_DELIVERED_TO_TOP;
            }
        }
        // An existing activity is starting this new activity, so we want
        // to keep the new one in the same task as the one that is starting
        // it.
        r.setTask(sourceTask, null);
        if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                + " in existing task " + r.task + " from source " + sourceRecord);

    } else if (inTask != null) {
        // The caller is asking that the new activity be started in an explicit
        // task it has provided to us.
        if (isLockTaskModeViolation(inTask)) {
            Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
            return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
        targetStack = inTask.stack;
        targetStack.moveTaskToFrontLocked(inTask, noAnimation, options, r.appTimeTracker,
                "inTaskToFront");

        // Check whether we should actually launch the new activity in to the task,
        // or just reuse the current activity on top.
        ActivityRecord top = inTask.getTopActivity();
        if (top != null && top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
            if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                    || launchSingleTop || launchSingleTask) {
                ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top, top.task);
                if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                    // We don't need to start a new activity, and
                    // the client said not to do anything if that
                    // is the case, so this is it!
                    return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                }
                top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                return ActivityManager.START_DELIVERED_TO_TOP;
            }
        }

        if (!addingToTask) {
            // We don't actually want to have this activity added to the task, so just
            // stop here but still tell the caller that we consumed the intent.
            ActivityOptions.abort(options);
            return ActivityManager.START_TASK_TO_FRONT;
        }

        r.setTask(inTask, null);
        if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                + " in explicit task " + r.task);

    } else {
        // This not being started from an existing activity, and not part
        // of a new task...  just put it in the top task, though these days
        // this case should never happen.
        targetStack = computeStackFocus(r, newTask);
        targetStack.moveToFront("addingToTopTask");
        ActivityRecord prev = targetStack.topActivity();
        r.setTask(prev != null ? prev.task : targetStack.createTaskRecord(getNextTaskId(),
                        r.info, intent, null, null, true), null);
        mWindowManager.moveTaskToTop(r.task.taskId);
        if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                + " in new guessed " + r.task);
    }

    mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
            intent, r.getUriPermissionsLocked(), r.userId);

    if (sourceRecord != null && sourceRecord.isRecentsActivity()) {
        r.task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE);
    }
    if (newTask) {
        EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.userId, r.task.taskId);
    }
    ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
    targetStack.mLastPausedActivity = null;
    targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
    if (!launchTaskBehind) {
        // Don't set focus on an activity that's going to the back.
        mService.setFocusedActivityLocked(r, "startedActivity");
    }
    return ActivityManager.START_SUCCESS;
}
```

#### 2.17 ActivityStack.startActivityLocked

```java
final void startActivityLocked(ActivityRecord r, boolean newTask,
        boolean doResume, boolean keepCurTransition, Bundle options) {
    TaskRecord rTask = r.task;
    final int taskId = rTask.taskId;
    // mLaunchTaskBehind tasks get placed at the back of the task stack.
    if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
        // Last activity in task had been removed or ActivityManagerService is reusing task.
        // Insert or replace.
        // Might not even be in.
        insertTaskAtTop(rTask, r);
        mWindowManager.moveTaskToTop(taskId);
    }
    TaskRecord task = null;
    if (!newTask) {
        // If starting in an existing task, find where that is...
        boolean startIt = true;
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            task = mTaskHistory.get(taskNdx);
            if (task.getTopActivity() == null) {
                // All activities in task are finishing.
                continue;
            }
            if (task == r.task) {
                // Here it is!  Now, if this is not yet visible to the
                // user, then just add it without starting; it will
                // get started when the user navigates back to it.
                if (!startIt) {
                    if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                            + task, new RuntimeException("here").fillInStackTrace());
                    task.addActivityToTop(r);
                    r.putInHistory();
                    mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                            r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                            (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0,
                            r.userId, r.info.configChanges, task.voiceSession != null,
                            r.mLaunchTaskBehind);
                    if (VALIDATE_TOKENS) {
                        validateAppTokensLocked();
                    }
                    ActivityOptions.abort(options);
                    return;
                }
                break;
            } else if (task.numFullscreen > 0) {
                startIt = false;
            }
        }
    }

    // Place a new activity at top of stack, so it is next to interact
    // with the user.

    // If we are not placing the new activity frontmost, we do not want
    // to deliver the onUserLeaving callback to the actual frontmost
    // activity
    if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
        mStackSupervisor.mUserLeaving = false;
        if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                "startActivity() behind front, mUserLeaving=false");
    }

    task = r.task;

    // Slot the activity into the history stack and proceed
    if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
            new RuntimeException("here").fillInStackTrace());
    task.addActivityToTop(r);
    task.setFrontOfTask();

    r.putInHistory();
    if (!isHomeStack() || numActivities() > 0) {
        // We want to show the starting preview window if we are
        // switching to a new task, or the next activity's process is
        // not currently running.
        boolean showStartingIcon = newTask;
        ProcessRecord proc = r.app;
        if (proc == null) {
            proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
        }
        if (proc == null || proc.thread == null) {
            showStartingIcon = true;
        }
        if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                "Prepare open transition: starting " + r);
        if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
            mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
            mNoAnimActivities.add(r);
        } else {
            mWindowManager.prepareAppTransition(newTask
                    ? r.mLaunchTaskBehind
                            ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                            : AppTransition.TRANSIT_TASK_OPEN
                    : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
            mNoAnimActivities.remove(r);
        }
        mWindowManager.addAppToken(task.mActivities.indexOf(r),
                r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
        boolean doShow = true;
        if (newTask) {
            // Even though this activity is starting fresh, we still need
            // to reset it to make sure we apply affinities to move any
            // existing activities from other tasks in to it.
            // If the caller has requested that the target task be
            // reset, then do so.
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                resetTaskIfNeededLocked(r, r);
                doShow = topRunningNonDelayedActivityLocked(null) == r;
            }
        } else if (options != null && new ActivityOptions(options).getAnimationType()
                == ActivityOptions.ANIM_SCENE_TRANSITION) {
            doShow = false;
        }
        if (r.mLaunchTaskBehind) {
            // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
            // tell WindowManager that r is visible even though it is at the back of the stack.
            mWindowManager.setAppVisibility(r.appToken, true);
            ensureActivitiesVisibleLocked(null, 0);
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            // Figure out if we are transitioning from another activity that is
            // "has the same starting icon" as the next one.  This allows the
            // window manager to keep the previous window it had previously
            // created, if it still had one.
            ActivityRecord prev = mResumedActivity;
            if (prev != null) {
                // We don't want to reuse the previous starting preview if:
                // (1) The current activity is in a different task.
                if (prev.task != r.task) {
                    prev = null;
                }
                // (2) The current activity is already displayed.
                else if (prev.nowVisible) {
                    prev = null;
                }
            }
            mWindowManager.setAppStartingWindow(
                    r.appToken, r.packageName, r.theme,
                    mService.compatibilityInfoForPackageLocked(
                            r.info.applicationInfo), r.nonLocalizedLabel,
                    r.labelRes, r.icon, r.logo, r.windowFlags,
                    prev != null ? prev.appToken : null, showStartingIcon);
            r.mStartingWindowShown = true;
        }
    } else {
        // If this is the first activity, don't do any fancy animations,
        // because there is nothing for it to animate on top of.
        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
        ActivityOptions.abort(options);
        options = null;
    }
    if (VALIDATE_TOKENS) {
        validateAppTokensLocked();
    }

    if (doResume) {
        mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
    }
}
```

#### 2.18 ActivityStackSupervisor.resumeTopActivitiesLocked

```java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
        Bundle targetOptions) {
    if (targetStack == null) {
        targetStack = mFocusedStack;
    }
    // Do targetStack first.
    boolean result = false;
    if (isFrontStack(targetStack)) {
        result = targetStack.resumeTopActivityLocked(target, targetOptions);
    }

    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (stack == targetStack) {
                // Already started above.
                continue;
            }
            if (isFrontStack(stack)) {
                stack.resumeTopActivityLocked(null);
            }
        }
    }
    return result;
}
```

#### 2.19 ActivityStack.resumeTopActivityLocked

```java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
            mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
            mService.updateSleepIfNeededLocked();
        }
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    return result;
}
```

#### 2.20 ActivityStack.resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
    if (DEBUG_LOCKSCREEN) mService.logLockScreen("");

    if (!mService.mBooting && !mService.mBooted) {
        // Not ready yet!
        return false;
    }

    ActivityRecord parent = mActivityContainer.mParentActivity;
    if ((parent != null && parent.state != ActivityState.RESUMED) ||
            !mActivityContainer.isAttachedLocked()) {
        // Do not resume this stack if its parent is not resumed.
        // TODO: If in a loop, make sure that parent stack resumeTopActivity is called 1st.
        return false;
    }

    cancelInitializingActivities();

    // Find the first activity that is not finishing.
    final ActivityRecord next = topRunningActivityLocked(null);

    // Remember how we'll process this pause/resume situation, and ensure
    // that the state is reset however we wind up proceeding.
    final boolean userLeaving = mStackSupervisor.mUserLeaving;
    mStackSupervisor.mUserLeaving = false;

    final TaskRecord prevTask = prev != null ? prev.task : null;
    if (next == null) {
        // There are no more activities!
        final String reason = "noMoreActivities";
        if (!mFullscreen) {
            // Try to move focus to the next visible stack with a running activity if this
            // stack is not covering the entire screen.
            final ActivityStack stack = getNextVisibleStackLocked();
            if (adjustFocusToNextVisibleStackLocked(stack, reason)) {
                return mStackSupervisor.resumeTopActivitiesLocked(stack, prev, null);
            }
        }
        // Let's just start up the Launcher...
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: No more activities go home");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        // Only resume home if on home display
        final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
        return isOnHomeDisplay() &&
                mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, reason);
    }

    next.delayedResume = false;

    // If the top activity is the resumed one, nothing to do.
    if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                mStackSupervisor.allResumedActivitiesComplete()) {
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        mWindowManager.executeAppTransition();
        mNoAnimActivities.clear();
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Top activity resumed " + next);
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    final TaskRecord nextTask = next.task;
    if (prevTask != null && prevTask.stack == this &&
            prevTask.isOverHomeStack() && prev.finishing && prev.frontOfTask) {
        if (DEBUG_STACK)  mStackSupervisor.validateTopActivitiesLocked();
        if (prevTask == nextTask) {
            prevTask.setFrontOfTask();
        } else if (prevTask != topTask()) {
            // This task is going away but it was supposed to return to the home stack.
            // Now the task above it has to return to the home task instead.
            final int taskNdx = mTaskHistory.indexOf(prevTask) + 1;
            mTaskHistory.get(taskNdx).setTaskToReturnTo(HOME_ACTIVITY_TYPE);
        } else if (!isOnHomeDisplay()) {
            return false;
        } else if (!isHomeStack()){
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Launching home next");
            final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                    HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, "prevFinished");
        }
    }

    // If we are sleeping, and there is no resumed activity, and the top
    // activity is paused, well that is the state we want.
    if (mService.isSleepingOrShuttingDown()
            && mLastPausedActivity == next
            && mStackSupervisor.allPausedActivitiesComplete()) {
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        mWindowManager.executeAppTransition();
        mNoAnimActivities.clear();
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Going to sleep and all paused");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    // Make sure that the user who owns this activity is started.  If not,
    // we will just leave it as is because someone should be bringing
    // another user's activities to the top of the stack.
    if (mService.mStartedUsers.get(next.userId) == null) {
        Slog.w(TAG, "Skipping resume of top activity " + next
                + ": user " + next.userId + " is stopped");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    // The activity may be waiting for stop, but that is no longer
    // appropriate for it.
    mStackSupervisor.mStoppingActivities.remove(next);
    mStackSupervisor.mGoingToSleepActivities.remove(next);
    next.sleeping = false;
    mStackSupervisor.mWaitingVisibleActivities.remove(next);

    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resuming " + next);

    // If we are currently pausing an activity, then don't do anything
    // until that is done.
    if (!mStackSupervisor.allPausedActivitiesComplete()) {
        if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                "resumeTopActivityLocked: Skip resume: some activity pausing.");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    // Okay we are now going to start a switch, to 'next'.  We may first
    // have to pause the current activity, but this is an important point
    // where we have decided to go to 'next' so keep track of that.
    // XXX "App Redirected" dialog is getting too many false positives
    // at this point, so turn off for now.
    if (false) {
        if (mLastStartedActivity != null && !mLastStartedActivity.finishing) {
            long now = SystemClock.uptimeMillis();
            final boolean inTime = mLastStartedActivity.startTime != 0
                    && (mLastStartedActivity.startTime + START_WARN_TIME) >= now;
            final int lastUid = mLastStartedActivity.info.applicationInfo.uid;
            final int nextUid = next.info.applicationInfo.uid;
            if (inTime && lastUid != nextUid
                    && lastUid != next.launchedFromUid
                    && mService.checkPermission(
                            android.Manifest.permission.STOP_APP_SWITCHES,
                            -1, next.launchedFromUid)
                    != PackageManager.PERMISSION_GRANTED) {
                mService.showLaunchWarningLocked(mLastStartedActivity, next);
            } else {
                next.startTime = now;
                mLastStartedActivity = next;
            }
        } else {
            next.startTime = SystemClock.uptimeMillis();
            mLastStartedActivity = next;
        }
    }

    mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

    // We need to start pausing the current activity so the top one
    // can be resumed...
    boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
    if (mResumedActivity != null) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Pausing " + mResumedActivity);
        pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
    }
    if (pausing) {
        if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                "resumeTopActivityLocked: Skip resume: need to start pausing");
        // At this point we want to put the upcoming activity's process
        // at the top of the LRU list, since we know we will be needing it
        // very soon and it would be a waste to let it get killed if it
        // happens to be sitting towards the end.
        if (next.app != null && next.app.thread != null) {
            mService.updateLruProcessLocked(next.app, true, null);
        }
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
    }

    // If the most recent activity was noHistory but was only stopped rather
    // than stopped+finished because the device went to sleep, we need to make
    // sure to finish it as we're making a new activity topmost.
    if (mService.isSleeping() && mLastNoHistoryActivity != null &&
            !mLastNoHistoryActivity.finishing) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "no-history finish of " + mLastNoHistoryActivity + " on new resume");
        requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                null, "resume-no-history", false);
        mLastNoHistoryActivity = null;
    }

    if (prev != null && prev != next) {
        if (!mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                && next != null && !next.nowVisible) {
            mStackSupervisor.mWaitingVisibleActivities.add(prev);
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                    "Resuming top, waiting visible to hide: " + prev);
        } else {
            // The next activity is already visible, so hide the previous
            // activity's windows right now so we can show the new one ASAP.
            // We only do this if the previous is finishing, which should mean
            // it is on top of the one being resumed so hiding it quickly
            // is good.  Otherwise, we want to do the normal route of allowing
            // the resumed activity to be shown so we can decide if the
            // previous should actually be hidden depending on whether the
            // new one is found to be full-screen or not.
            if (prev.finishing) {
                mWindowManager.setAppVisibility(prev.appToken, false);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Not waiting for visible to hide: " + prev + ", waitingVisible="
                        + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                        + ", nowVisible=" + next.nowVisible);
            } else {
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Previous already visible but still waiting to hide: " + prev
                        + ", waitingVisible="
                        + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                        + ", nowVisible=" + next.nowVisible);
            }
        }
    }

    // Launching this app's activity, make sure the app is no longer
    // considered stopped.
    try {
        AppGlobals.getPackageManager().setPackageStoppedState(
                next.packageName, false, next.userId); /* TODO: Verify if correct userid */
    } catch (RemoteException e1) {
    } catch (IllegalArgumentException e) {
        Slog.w(TAG, "Failed trying to unstop package "
                + next.packageName + ": " + e);
    }

    // We are starting up the next activity, so tell the window manager
    // that the previous one will be hidden soon.  This way it can know
    // to ignore it when computing the desired screen orientation.
    boolean anim = true;
    if (prev != null) {
        if (prev.finishing) {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare close transition: prev=" + prev);
            if (mNoAnimActivities.contains(prev)) {
                anim = false;
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(prev.task == next.task
                        ? AppTransition.TRANSIT_ACTIVITY_CLOSE
                        : AppTransition.TRANSIT_TASK_CLOSE, false);
            }
            mWindowManager.setAppWillBeHidden(prev.appToken);
            mWindowManager.setAppVisibility(prev.appToken, false);
        } else {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: prev=" + prev);
            if (mNoAnimActivities.contains(next)) {
                anim = false;
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(prev.task == next.task
                        ? AppTransition.TRANSIT_ACTIVITY_OPEN
                        : next.mLaunchTaskBehind
                                ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                : AppTransition.TRANSIT_TASK_OPEN, false);
            }
        }
        if (false) {
            mWindowManager.setAppWillBeHidden(prev.appToken);
            mWindowManager.setAppVisibility(prev.appToken, false);
        }
    } else {
        if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, "Prepare open transition: no previous");
        if (mNoAnimActivities.contains(next)) {
            anim = false;
            mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
        } else {
            mWindowManager.prepareAppTransition(AppTransition.TRANSIT_ACTIVITY_OPEN, false);
        }
    }

    Bundle resumeAnimOptions = null;
    if (anim) {
        ActivityOptions opts = next.getOptionsForTargetActivityLocked();
        if (opts != null) {
            resumeAnimOptions = opts.toBundle();
        }
        next.applyOptionsLocked();
    } else {
        next.clearOptionsLocked();
    }

    ActivityStack lastStack = mStackSupervisor.getLastStack();
    if (next.app != null && next.app.thread != null) {
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next);

        // This activity is now becoming visible.
        mWindowManager.setAppVisibility(next.appToken, true);

        // schedule launch ticks to collect information about slow apps.
        next.startLaunchTickingLocked();

        ActivityRecord lastResumedActivity =
                lastStack == null ? null :lastStack.mResumedActivity;
        ActivityState lastState = next.state;

        mService.updateCpuStats();

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next + " (in existing)");
        next.state = ActivityState.RESUMED;
        mResumedActivity = next;
        next.task.touchActiveTime();
        mRecentTasks.addLocked(next.task);
        mService.updateLruProcessLocked(next.app, true, null);
        updateLRUListLocked(next);
        mService.updateOomAdjLocked();

        // Have the window manager re-evaluate the orientation of
        // the screen based on the new activity order.
        boolean notUpdated = true;
        if (mStackSupervisor.isFrontStack(this)) {
            Configuration config = mWindowManager.updateOrientationFromAppTokens(
                    mService.mConfiguration,
                    next.mayFreezeScreenLocked(next.app) ? next.appToken : null);
            if (config != null) {
                next.frozenBeforeDestroy = true;
            }
            notUpdated = !mService.updateConfigurationLocked(config, next, false, false);
        }

        if (notUpdated) {
            // The configuration update wasn't able to keep the existing
            // instance of the activity, and instead started a new one.
            // We should be all done, but let's just make sure our activity
            // is still at the top and schedule another run if something
            // weird happened.
            ActivityRecord nextNext = topRunningActivityLocked(null);
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                    "Activity config changed during resume: " + next
                    + ", new next: " + nextNext);
            if (nextNext != next) {
                // Do over!
                mStackSupervisor.scheduleResumeTopActivities();
            }
            if (mStackSupervisor.reportResumedActivityLocked(next)) {
                mNoAnimActivities.clear();
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        try {
            // Deliver all pending results.
            ArrayList<ResultInfo> a = next.results;
            if (a != null) {
                final int N = a.size();
                if (!next.finishing && N > 0) {
                    if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                            "Delivering results to " + next + ": " + a);
                    next.app.thread.scheduleSendResult(next.appToken, a);
                }
            }

            if (next.newIntents != null) {
                next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
            }

            EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                    System.identityHashCode(next), next.task.taskId, next.shortComponentName);

            next.sleeping = false;
            mService.showAskCompatModeDialogLocked(next);
            next.app.pendingUiClean = true;
            next.app.forceProcessStateUpTo(mService.mTopProcessState);
            next.clearOptionsLocked();
            next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                    mService.isNextTransitionForward(), resumeAnimOptions);

            mStackSupervisor.checkReadyForSleepLocked();

            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed " + next);
        } catch (Exception e) {
            // Whoops, need to restart this activity!
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Resume failed; resetting state to "
                    + lastState + ": " + next);
            next.state = lastState;
            if (lastStack != null) {
                lastStack.mResumedActivity = lastResumedActivity;
            }
            Slog.i(TAG, "Restarting because process died: " + next);
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else  if (SHOW_APP_STARTING_PREVIEW && lastStack != null &&
                    mStackSupervisor.isFrontStack(lastStack)) {
                mWindowManager.setAppStartingWindow(
                        next.appToken, next.packageName, next.theme,
                        mService.compatibilityInfoForPackageLocked(next.info.applicationInfo),
                        next.nonLocalizedLabel, next.labelRes, next.icon, next.logo,
                        next.windowFlags, null, true);
            }
            mStackSupervisor.startSpecificActivityLocked(next, true, false);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        // From this point on, if something goes wrong there is no way
        // to recover the activity.
        try {
            next.visible = true;
            completeResumeLocked(next);
        } catch (Exception e) {
            // If any exception gets thrown, toss away this
            // activity and try the next one.
            Slog.w(TAG, "Exception thrown during resume of " + next, e);
            requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                    "resume-exception", true);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }
        next.stopped = false;

    } else {
        // Whoops, need to restart this activity!
        if (!next.hasBeenLaunched) {
            next.hasBeenLaunched = true;
        } else {
            if (SHOW_APP_STARTING_PREVIEW) {
                mWindowManager.setAppStartingWindow(
                        next.appToken, next.packageName, next.theme,
                        mService.compatibilityInfoForPackageLocked(
                                next.info.applicationInfo),
                        next.nonLocalizedLabel,
                        next.labelRes, next.icon, next.logo, next.windowFlags,
                        null, true);
            }
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
        }
        if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }

    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```

#### ActivityStackSupervisor.pauseBackStacks


```java
/**
 * Pause all activities in either all of the stacks or just the back stacks.
 * @param userLeaving Passed to pauseActivity() to indicate whether to call onUserLeaving().
 * @return true if any activity was paused as a result of this call.
 */
boolean pauseBackStacks(boolean userLeaving, boolean resuming, boolean dontWait) {
    boolean someActivityPaused = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFrontStack(stack) && stack.mResumedActivity != null) {
                if (DEBUG_STATES) Slog.d(TAG_STATES, "pauseBackStacks: stack=" + stack +
                        " mResumedActivity=" + stack.mResumedActivity);
                someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                        dontWait);
            }
        }
    }
    return someActivityPaused;
}
```

#### ActivityStack.startPausingLocked

```java
/**
 * Start pausing the currently resumed activity.  It is an error to call this if there
 * is already an activity being paused or there is no resumed activity.
 *
 * @param userLeaving True if this should result in an onUserLeaving to the current activity.
 * @param uiSleeping True if this is happening with the user interface going to sleep (the
 * screen turning off).
 * @param resuming True if this is being called as part of resuming the top activity, so
 * we shouldn't try to instigate a resume here.
 * @param dontWait True if the caller does not want to wait for the pause to complete.  If
 * set to true, we will immediately complete the pause here before returning.
 * @return Returns true if an activity now is in the PAUSING state, and we are waiting for
 * it to tell us when it is done.
 */
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
        boolean dontWait) {
    if (mPausingActivity != null) {
        Slog.wtf(TAG, "Going to pause when pause is already pending for " + mPausingActivity
                + " state=" + mPausingActivity.state);
        if (!mService.isSleeping()) {
            // Avoid recursion among check for sleep and complete pause during sleeping.
            // Because activity will be paused immediately after resume, just let pause
            // be completed by the order of activity paused from clients.
            completePauseLocked(false);
        }
    }
    ActivityRecord prev = mResumedActivity;
    if (prev == null) {
        if (!resuming) {
            Slog.wtf(TAG, "Trying to pause when nothing is resumed");
            mStackSupervisor.resumeTopActivitiesLocked();
        }
        return false;
    }

    if (mActivityContainer.mParentActivity == null) {
        // Top level stack, not a child. Look for child stacks.
        mStackSupervisor.pauseChildStacks(prev, userLeaving, uiSleeping, resuming, dontWait);
    }

    if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
    else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);
    mResumedActivity = null;
    mPausingActivity = prev;
    mLastPausedActivity = prev;
    mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
            || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
    prev.state = ActivityState.PAUSING;
    prev.task.touchActiveTime();
    clearLaunchTime(prev);
    final ActivityRecord next = mStackSupervisor.topRunningActivityLocked();
    if (mService.mHasRecents && (next == null || next.noDisplay || next.task != prev.task || uiSleeping)) {
        prev.updateThumbnailLocked(screenshotActivities(prev), null);
    }
    stopFullyDrawnTraceIfNeeded();

    mService.updateCpuStats();

    if (prev.app != null && prev.app.thread != null) {
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
        try {
            EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                    prev.userId, System.identityHashCode(prev),
                    prev.shortComponentName);
            mService.updateUsageStats(prev, false);
            prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                    userLeaving, prev.configChangeFlags, dontWait);
        } catch (Exception e) {
            // Ignore exception, if process died other code will cleanup.
            Slog.w(TAG, "Exception thrown during pause", e);
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
    } else {
        mPausingActivity = null;
        mLastPausedActivity = null;
        mLastNoHistoryActivity = null;
    }

    // If we are not going to sleep, we want to ensure the device is
    // awake until the next activity is started.
    if (!uiSleeping && !mService.isSleepingOrShuttingDown()) {
        mStackSupervisor.acquireLaunchWakelock();
    }

    if (mPausingActivity != null) {
        // Have the window manager pause its key dispatching until the new
        // activity has started.  If we're pausing the activity just because
        // the screen is being turned off and the UI is sleeping, don't interrupt
        // key dispatch; the same activity will pick it up again on wakeup.
        if (!uiSleeping) {
            prev.pauseKeyDispatchingLocked();
        } else if (DEBUG_PAUSE) {
             Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
        }

        if (dontWait) {
            // If the caller said they don't want to wait for the pause, then complete
            // the pause now.
            completePauseLocked(false);
            return false;

        } else {
            // Schedule a pause timeout in case the app doesn't respond.
            // We don't give it much time because this directly impacts the
            // responsiveness seen by the user.
            Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
            msg.obj = prev;
            prev.pauseTime = SystemClock.uptimeMillis();
            mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Waiting for pause to complete...");
            return true;
        }

    } else {
        // This activity failed to schedule the
        // pause, so just treat it as being paused now.
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
        if (!resuming) {
            mStackSupervisor.getFocusedStack().resumeTopActivityLocked(null);
        }
        return false;
    }
}
```

#### ActivityStack.completePauseLocked

```Java
private void completePauseLocked(boolean resumeNext) {
    ActivityRecord prev = mPausingActivity;
    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Complete pause: " + prev);

    if (prev != null) {
        prev.state = ActivityState.PAUSED;
        if (prev.finishing) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Executing finish of activity: " + prev);
            prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false);
        } else if (prev.app != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending stop: " + prev);
            if (mStackSupervisor.mWaitingVisibleActivities.remove(prev)) {
                if (DEBUG_SWITCH || DEBUG_PAUSE) Slog.v(TAG_PAUSE,
                        "Complete pause, no longer waiting: " + prev);
            }
            if (prev.configDestroy) {
                // The previous is being paused because the configuration
                // is changing, which means it is actually stopping...
                // To juggle the fact that we are also starting a new
                // instance right now, we need to first completely stop
                // the current instance before starting the new one.
                if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Destroying after pause: " + prev);
                destroyActivityLocked(prev, true, "pause-config");
            } else if (!hasVisibleBehindActivity() || mService.isSleepingOrShuttingDown()) {
                // If we were visible then resumeTopActivities will release resources before
                // stopping.
                mStackSupervisor.mStoppingActivities.add(prev);
                if (mStackSupervisor.mStoppingActivities.size() > 3 ||
                        prev.frontOfTask && mTaskHistory.size() <= 1) {
                    // If we already have a few activities waiting to stop,
                    // then give up on things going idle and start clearing
                    // them out. Or if r is the last of activity of the last task the stack
                    // will be empty and must be cleared immediately.
                    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "To many pending stops, forcing idle");
                    mStackSupervisor.scheduleIdleLocked();
                } else {
                    mStackSupervisor.checkReadyForSleepLocked();
                }
            }
        } else {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "App died during pause, not stopping: " + prev);
            prev = null;
        }
        // It is possible the activity was freezing the screen before it was paused.
        // In that case go ahead and remove the freeze this activity has on the screen
        // since it is no longer visible.
        prev.stopFreezingScreenLocked(true /*force*/);
        mPausingActivity = null;
    }

    if (resumeNext) {
        final ActivityStack topStack = mStackSupervisor.getFocusedStack();
        if (!mService.isSleepingOrShuttingDown()) {
            mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
        } else {
            mStackSupervisor.checkReadyForSleepLocked();
            ActivityRecord top = topStack.topRunningActivityLocked(null);
            if (top == null || (prev != null && top != prev)) {
                // If there are no more activities available to run,
                // do resume anyway to start something.  Also if the top
                // activity on the stack is not the just paused activity,
                // we need to go ahead and resume it to ensure we complete
                // an in-flight app switch.
                mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
            }
        }
    }

    if (prev != null) {
        prev.resumeKeyDispatchingLocked();

        if (prev.app != null && prev.cpuTimeAtResume > 0
                && mService.mBatteryStatsService.isOnBattery()) {
            long diff = mService.mProcessCpuTracker.getCpuTimeForPid(prev.app.pid)
                    - prev.cpuTimeAtResume;
            if (diff > 0) {
                BatteryStatsImpl bsi = mService.mBatteryStatsService.getActiveStatistics();
                synchronized (bsi) {
                    BatteryStatsImpl.Uid.Proc ps =
                            bsi.getProcessStatsLocked(prev.info.applicationInfo.uid,
                                    prev.info.packageName);
                    if (ps != null) {
                        ps.addForegroundTimeLocked(diff);
                    }
                }
            }
        }
        prev.cpuTimeAtResume = 0; // reset it
    }

    // Notfiy when the task stack has changed
    mService.notifyTaskStackChangedLocked();
}
```

#### ActivityStackSupervisor.startSpecificActivityLocked

```java
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // activity的application是否在运行？
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    r.task.stack.setLaunchTime(r);

    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                // Don't add this if it is a platform component that is marked
                // to run in multiple processes, because this is actually
                // part of the framework so doesn't make sense to track as a
                // separate apk in the process.
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                        mService.mProcessStats);
            }
            // 目标Activity已经启动
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    // 需要启动的Activity所属进程不存在，通过Zygote创建应用进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```

#### ActivityManagerService.startProcessLocked


```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
        boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    long startTime = SystemClock.elapsedRealtime();
    ProcessRecord app;
    if (!isolated) {
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
        checkTime(startTime, "startProcess: after getProcessRecord");

        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            // If we are in the background, then check to see if this process
            // is bad.  If so, we will just silently fail.
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                if (DEBUG_PROCESSES) Slog.v(TAG, "Bad process: " + info.uid
                        + "/" + info.processName);
                return null;
            }
        } else {
            // When the user is explicitly starting a process, then clear its
            // crash count so that we won't make it bad until they see at
            // least one crash dialog again, and make the process good again
            // if it had been bad.
            if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                    + "/" + info.processName);
            mProcessCrashTimes.remove(info.processName, info.uid);
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                        UserHandle.getUserId(info.uid), info.uid,
                        info.processName);
                mBadProcesses.remove(info.processName, info.uid);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        // If this is an isolated process, it can't re-use an existing process.
        app = null;
    }

    // We don't have to do anything more if:
    // (1) There is an existing application record; and
    // (2) The caller doesn't think it is dead, OR there is no thread
    //     object attached to it so we know it couldn't have crashed; and
    // (3) There is a pid assigned to it, so it is either starting or
    //     already running.
    if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "startProcess: name=" + processName
            + " app=" + app + " knownToBeDead=" + knownToBeDead
            + " thread=" + (app != null ? app.thread : null)
            + " pid=" + (app != null ? app.pid : -1));
    if (app != null && app.pid > 0) {
        if (!knownToBeDead || app.thread == null) {
            // We already have the app running, or are waiting for it to
            // come up (we have a pid but not yet its thread), so keep it.
            if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "App already running: " + app);
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            checkTime(startTime, "startProcess: done, added package to proc");
            return app;
        }

        // An application record is attached to a previous process,
        // clean it up now.
        if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG_PROCESSES, "App died: " + app);
        checkTime(startTime, "startProcess: bad proc running, killing");
        killProcessGroup(app.info.uid, app.pid);
        handleAppDiedLocked(app, true, true);
        checkTime(startTime, "startProcess: done killing old proc");
    }

    String hostingNameStr = hostingName != null
            ? hostingName.flattenToShortString() : null;

    if (app == null) {
        checkTime(startTime, "startProcess: creating new process record");
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
        if (app == null) {
            Slog.w(TAG, "Failed making new process record for "
                    + processName + "/" + info.uid + " isolated=" + isolated);
            return null;
        }
        app.crashHandler = crashHandler;
        checkTime(startTime, "startProcess: done creating new process record");
    } else {
        // If this is a new package in the process, add the package to the list
        app.addPackage(info.packageName, info.versionCode, mProcessStats);
        checkTime(startTime, "startProcess: added package to existing proc");
    }

    // If the system is not ready yet, then hold off on starting this
    // process until it is.
    if (!mProcessesReady
            && !isAllowedWhileBooting(info)
            && !allowWhileBooting) {
        if (!mProcessesOnHold.contains(app)) {
            mProcessesOnHold.add(app);
        }
        if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES,
                "System not ready, putting on hold: " + app);
        checkTime(startTime, "startProcess: returning with proc on hold");
        return app;
    }

    checkTime(startTime, "startProcess: stepping in to startProcess");
    startProcessLocked(
            app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
    checkTime(startTime, "startProcess: done starting proc!");
    return (app.pid != 0) ? app : null;
}
```

```java
private final void startProcessLocked(ProcessRecord app,
        String hostingType, String hostingNameStr) {
    startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
            null /* entryPoint */, null /* entryPointArgs */);
}

private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    long startTime = SystemClock.elapsedRealtime();
    if (app.pid > 0 && app.pid != MY_PID) {
        checkTime(startTime, "startProcess: removing from pids map");
        synchronized (mPidsSelfLocked) {
            mPidsSelfLocked.remove(app.pid);
            mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        }
        checkTime(startTime, "startProcess: done removing from pids map");
        app.setPid(0);
    }

    if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG_PROCESSES,
            "startProcessLocked removing on hold: " + app);
    mProcessesOnHold.remove(app);

    checkTime(startTime, "startProcess: starting to update cpu stats");
    updateCpuStats();
    checkTime(startTime, "startProcess: done updating cpu stats");

    try {
        try {
            if (AppGlobals.getPackageManager().isPackageFrozen(app.info.packageName)) {
                // This is caught below as if we had failed to fork zygote
                throw new RuntimeException("Package " + app.info.packageName + " is frozen!");
            }
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }

        int uid = app.uid;
        int[] gids = null;
        int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
        if (!app.isolated) {
            int[] permGids = null;
            try {
                checkTime(startTime, "startProcess: getting gids from package manager");
                final IPackageManager pm = AppGlobals.getPackageManager();
                permGids = pm.getPackageGids(app.info.packageName, app.userId);
                MountServiceInternal mountServiceInternal = LocalServices.getService(
                        MountServiceInternal.class);
                mountExternal = mountServiceInternal.getExternalStorageMountMode(uid,
                        app.info.packageName);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            /*
             * Add shared application and profile GIDs so applications can share some
             * resources like shared libraries and access user-wide resources
             */
            if (ArrayUtils.isEmpty(permGids)) {
                gids = new int[2];
            } else {
                gids = new int[permGids.length + 2];
                System.arraycopy(permGids, 0, gids, 2, permGids.length);
            }
            gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
            gids[1] = UserHandle.getUserGid(UserHandle.getUserId(uid));
        }
        checkTime(startTime, "startProcess: building args");
        if (mFactoryTest != FactoryTest.FACTORY_TEST_OFF) {
            if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                    && mTopComponent != null
                    && app.processName.equals(mTopComponent.getPackageName())) {
                uid = 0;
            }
            if (mFactoryTest == FactoryTest.FACTORY_TEST_HIGH_LEVEL
                    && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                uid = 0;
            }
        }
        int debugFlags = 0;
        if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
            debugFlags |= Zygote.DEBUG_ENABLE_DEBUGGER;
            // Also turn on CheckJNI for debuggable apps. It's quite
            // awkward to turn on otherwise.
            debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;
        }
        // Run the app in safe mode if its manifest requests so or the
        // system is booted in safe mode.
        if ((app.info.flags & ApplicationInfo.FLAG_VM_SAFE_MODE) != 0 ||
            mSafeMode == true) {
            debugFlags |= Zygote.DEBUG_ENABLE_SAFEMODE;
        }
        if ("1".equals(SystemProperties.get("debug.checkjni"))) {
            debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;
        }
        String jitDebugProperty = SystemProperties.get("debug.usejit");
        if ("true".equals(jitDebugProperty)) {
            debugFlags |= Zygote.DEBUG_ENABLE_JIT;
        } else if (!"false".equals(jitDebugProperty)) {
            // If we didn't force disable by setting false, defer to the dalvik vm options.
            if ("true".equals(SystemProperties.get("dalvik.vm.usejit"))) {
                debugFlags |= Zygote.DEBUG_ENABLE_JIT;
            }
        }
        String genDebugInfoProperty = SystemProperties.get("debug.generate-debug-info");
        if ("true".equals(genDebugInfoProperty)) {
            debugFlags |= Zygote.DEBUG_GENERATE_DEBUG_INFO;
        }
        if ("1".equals(SystemProperties.get("debug.jni.logging"))) {
            debugFlags |= Zygote.DEBUG_ENABLE_JNI_LOGGING;
        }
        if ("1".equals(SystemProperties.get("debug.assert"))) {
            debugFlags |= Zygote.DEBUG_ENABLE_ASSERT;
        }

        String requiredAbi = (abiOverride != null) ? abiOverride : app.info.primaryCpuAbi;
        if (requiredAbi == null) {
            requiredAbi = Build.SUPPORTED_ABIS[0];
        }

        String instructionSet = null;
        if (app.info.primaryCpuAbi != null) {
            instructionSet = VMRuntime.getInstructionSet(app.info.primaryCpuAbi);
        }

        app.gids = gids;
        app.requiredAbi = requiredAbi;
        app.instructionSet = instructionSet;

        // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                app.processName);
        checkTime(startTime, "startProcess: asking zygote to start proc");
        // 调用Process.start()，通过socket发送消息给zygote
        // Zygote派生出一个子进程，子进程将通过反射调用ActivityThread的main函数
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);
        checkTime(startTime, "startProcess: returned from zygote!");
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        if (app.isolated) {
            mBatteryStatsService.addIsolatedUid(app.uid, app.info.uid);
        }
        mBatteryStatsService.noteProcessStart(app.processName, app.info.uid);
        checkTime(startTime, "startProcess: done updating battery stats");

        EventLog.writeEvent(EventLogTags.AM_PROC_START,
                UserHandle.getUserId(uid), startResult.pid, uid,
                app.processName, hostingType,
                hostingNameStr != null ? hostingNameStr : "");

        if (app.persistent) {
            Watchdog.getInstance().processStarted(app.processName, startResult.pid);
        }

        checkTime(startTime, "startProcess: building log message");
        StringBuilder buf = mStringBuilder;
        buf.setLength(0);
        buf.append("Start proc ");
        buf.append(startResult.pid);
        buf.append(':');
        buf.append(app.processName);
        buf.append('/');
        UserHandle.formatUid(buf, uid);
        if (!isActivityProcess) {
            buf.append(" [");
            buf.append(entryPoint);
            buf.append("]");
        }
        buf.append(" for ");
        buf.append(hostingType);
        if (hostingNameStr != null) {
            buf.append(" ");
            buf.append(hostingNameStr);
        }
        Slog.i(TAG, buf.toString());
        app.setPid(startResult.pid);
        app.usingWrapper = startResult.usingWrapper;
        app.removed = false;
        app.killed = false;
        app.killedByAm = false;
        checkTime(startTime, "startProcess: starting to update pids map");
        synchronized (mPidsSelfLocked) {
            this.mPidsSelfLocked.put(startResult.pid, app);
            if (isActivityProcess) {
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        }
        checkTime(startTime, "startProcess: done updating pids map");
    } catch (RuntimeException e) {
        // XXX do better error recovery.
        app.setPid(0);
        mBatteryStatsService.noteProcessFinish(app.processName, app.info.uid);
        if (app.isolated) {
            mBatteryStatsService.removeIsolatedUid(app.uid, app.info.uid);
        }
        Slog.e(TAG, "Failure starting process " + app.processName, e);
    }
}
```

#### ActivityManagerService.attachApplicationLocked

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

#### ActivityStackSupervisor.attachApplicationLocked

```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFrontStack(stack)) {
                continue;
            }
            ActivityRecord hr = stack.topRunningActivityLocked(null);
            if (hr != null) {
                if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                        && processName.equals(hr.processName)) {
                    try {
                        if (realStartActivityLocked(hr, app, true, true)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Exception in new application when starting activity "
                              + hr.intent.getComponent().flattenToShortString(), e);
                        throw e;
                    }
                }
            }
        }
    }
    if (!didSomething) {
        ensureActivitiesVisibleLocked(null, 0);
    }
    return didSomething;
}
```

#### ApplicationThreadProxy.ApplicationThreadProxy

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo info, Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state, PersistableBundle persistentState, List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
     Parcel data = Parcel.obtain();
     data.writeInterfaceToken(IApplicationThread.descriptor);
     intent.writeToParcel(data, 0);
     data.writeStrongBinder(token);
     data.writeInt(ident);
     info.writeToParcel(data, 0);
     curConfig.writeToParcel(data, 0);
     if (overrideConfig != null) {
         data.writeInt(1);
         overrideConfig.writeToParcel(data, 0);
     } else {
         data.writeInt(0);
     }
     compatInfo.writeToParcel(data, 0);
     data.writeString(referrer);
     data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
     data.writeInt(procState);
     data.writeBundle(state);
     data.writePersistableBundle(persistentState);
     data.writeTypedList(pendingResults);
     data.writeTypedList(pendingNewIntents);
     data.writeInt(notResumed ? 1 : 0);
     data.writeInt(isForward ? 1 : 0);
     if (profilerInfo != null) {
         data.writeInt(1);
         profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
     } else {
         data.writeInt(0);
     }
     mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
             IBinder.FLAG_ONEWAY);
     data.recycle();
 }
```

#### ActivityStackSupervisor.realStartActivityLocked

```java
final boolean realStartActivityLocked(ActivityRecord r,
        ProcessRecord app, boolean andResume, boolean checkConfig)
        throws RemoteException {

    if (andResume) {
        r.startFreezingScreenLocked(app, 0);
        mWindowManager.setAppVisibility(r.appToken, true);

        // schedule launch ticks to collect information about slow apps.
        r.startLaunchTickingLocked();
    }

    // Have the window manager re-evaluate the orientation of
    // the screen based on the new activity order.  Note that
    // as a result of this, it can call back into the activity
    // manager with a new orientation.  We don't care about that,
    // because the activity is not currently running so we are
    // just restarting it anyway.
    if (checkConfig) {
        Configuration config = mWindowManager.updateOrientationFromAppTokens(
                mService.mConfiguration,
                r.mayFreezeScreenLocked(app) ? r.appToken : null);
        mService.updateConfigurationLocked(config, r, false, false);
    }

    r.app = app;
    app.waitingToKill = null;
    r.launchCount++;
    r.lastLaunchTime = SystemClock.uptimeMillis();

    if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);

    int idx = app.activities.indexOf(r);
    if (idx < 0) {
        app.activities.add(r);
    }
    mService.updateLruProcessLocked(app, true, null);
    mService.updateOomAdjLocked();

    final TaskRecord task = r.task;
    if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE ||
            task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV) {
        setLockTaskModeLocked(task, LOCK_TASK_MODE_LOCKED, "mLockTaskAuth==LAUNCHABLE", false);
    }

    final ActivityStack stack = task.stack;
    try {
        if (app.thread == null) {
            throw new RemoteException();
        }
        List<ResultInfo> results = null;
        List<ReferrerIntent> newIntents = null;
        if (andResume) {
            results = r.results;
            newIntents = r.newIntents;
        }
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                + " newIntents=" + newIntents + " andResume=" + andResume);
        if (andResume) {
            EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY,
                    r.userId, System.identityHashCode(r),
                    task.taskId, r.shortComponentName);
        }
        if (r.isHomeActivity() && r.isNotResolverActivity()) {
            // Home process is the root process of the task.
            mService.mHomeProcess = task.mActivities.get(0).app;
        }
        mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
        r.sleeping = false;
        r.forceNewConfig = false;
        mService.showAskCompatModeDialogLocked(r);
        r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
        ProfilerInfo profilerInfo = null;
        if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
            if (mService.mProfileProc == null || mService.mProfileProc == app) {
                mService.mProfileProc = app;
                final String profileFile = mService.mProfileFile;
                if (profileFile != null) {
                    ParcelFileDescriptor profileFd = mService.mProfileFd;
                    if (profileFd != null) {
                        try {
                            profileFd = profileFd.dup();
                        } catch (IOException e) {
                            if (profileFd != null) {
                                try {
                                    profileFd.close();
                                } catch (IOException o) {
                                }
                                profileFd = null;
                            }
                        }
                    }

                    profilerInfo = new ProfilerInfo(profileFile, profileFd,
                            mService.mSamplingInterval, mService.mAutoStopProfiler);
                }
            }
        }

        if (andResume) {
            app.hasShownUi = true;
            app.pendingUiClean = true;
        }
        app.forceProcessStateUpTo(mService.mTopProcessState);
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

        if ((app.info.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            // This may be a heavy-weight process!  Note that the package
            // manager will ensure that only activity can run in the main
            // process of the .apk, which is the only thing that will be
            // considered heavy-weight.
            if (app.processName.equals(app.info.packageName)) {
                if (mService.mHeavyWeightProcess != null
                        && mService.mHeavyWeightProcess != app) {
                    Slog.w(TAG, "Starting new heavy weight process " + app
                            + " when already running "
                            + mService.mHeavyWeightProcess);
                }
                mService.mHeavyWeightProcess = app;
                Message msg = mService.mHandler.obtainMessage(
                        ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                msg.obj = r;
                mService.mHandler.sendMessage(msg);
            }
        }

    } catch (RemoteException e) {
        if (r.launchFailed) {
            // This is the second time we failed -- finish activity
            // and give up.
            Slog.e(TAG, "Second failure launching "
                  + r.intent.getComponent().flattenToShortString()
                  + ", giving up", e);
            mService.appDiedLocked(app);
            stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                    "2nd-crash", false);
            return false;
        }

        // This is the first time we failed -- restart process and
        // retry.
        app.activities.remove(r);
        throw e;
    }

    r.launchFailed = false;
    if (stack.updateLRUListLocked(r)) {
        Slog.w(TAG, "Activity " + r
              + " being launched, but already in LRU list");
    }

    if (andResume) {
        // As part of the process of launching, ActivityThread also performs
        // a resume.
        stack.minimalResumeActivityLocked(r);
    } else {
        // This activity is not starting in the resumed state... which
        // should look like we asked it to pause+stop (but remain visible),
        // and it has done so and reported back the current icicle and
        // other state.
        if (DEBUG_STATES) Slog.v(TAG_STATES,
                "Moving to STOPPED: " + r + " (starting in stopped state)");
        r.state = STOPPED;
        r.stopped = true;
    }

    // Launch the new version setup screen if needed.  We do this -after-
    // launching the initial activity (that is, home), so that it can have
    // a chance to initialize itself while in the background, making the
    // switch back to it faster and look better.
    if (isFrontStack(stack)) {
        mService.startSetupActivityLocked();
    }

    // Update any services we are bound to that might care about whether
    // their client may have activities.
    mService.mServices.updateServiceConnectionActivitiesLocked(r.app);

    return true;
}
```

#### ApplicationThreadNative.onTransact

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean finished = data.readInt() != 0;
        boolean userLeaving = data.readInt() != 0;
        int configChanges = data.readInt();
        boolean dontReport = data.readInt() != 0;
        schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
        return true;
    }

    case SCHEDULE_STOP_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean show = data.readInt() != 0;
        int configChanges = data.readInt();
        scheduleStopActivity(b, show, configChanges);
        return true;
    }

    case SCHEDULE_WINDOW_VISIBILITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean show = data.readInt() != 0;
        scheduleWindowVisibility(b, show);
        return true;
    }

    case SCHEDULE_SLEEPING_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        boolean sleeping = data.readInt() != 0;
        scheduleSleeping(b, sleeping);
        return true;
    }

    case SCHEDULE_RESUME_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        int procState = data.readInt();
        boolean isForward = data.readInt() != 0;
        Bundle resumeArgs = data.readBundle();
        scheduleResumeActivity(b, procState, isForward, resumeArgs);
        return true;
    }

    case SCHEDULE_SEND_RESULT_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder b = data.readStrongBinder();
        List<ResultInfo> ri = data.createTypedArrayList(ResultInfo.CREATOR);
        scheduleSendResult(b, ri);
        return true;
    }

    case SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        Intent intent = Intent.CREATOR.createFromParcel(data);
        IBinder b = data.readStrongBinder();
        int ident = data.readInt();
        ActivityInfo info = ActivityInfo.CREATOR.createFromParcel(data);
        Configuration curConfig = Configuration.CREATOR.createFromParcel(data);
        Configuration overrideConfig = null;
        if (data.readInt() != 0) {
            overrideConfig = Configuration.CREATOR.createFromParcel(data);
        }
        CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
        String referrer = data.readString();
        IVoiceInteractor voiceInteractor = IVoiceInteractor.Stub.asInterface(
                data.readStrongBinder());
        int procState = data.readInt();
        Bundle state = data.readBundle();
        PersistableBundle persistentState = data.readPersistableBundle();
        List<ResultInfo> ri = data.createTypedArrayList(ResultInfo.CREATOR);
        List<ReferrerIntent> pi = data.createTypedArrayList(ReferrerIntent.CREATOR);
        boolean notResumed = data.readInt() != 0;
        boolean isForward = data.readInt() != 0;
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        scheduleLaunchActivity(intent, b, ident, info, curConfig, overrideConfig, compatInfo,
                referrer, voiceInteractor, procState, state, persistentState, ri, pi,
                notResumed, isForward, profilerInfo);
        return true;
    }
    .....
}
```

#### ApplicationThread.scheduleLaunchActivity

__ActivityThread__ 的内部类

```java
private class ApplicationThread extends ApplicationThreadNative {
    private static final String DB_INFO_FORMAT = "  %8s %8s %14s %14s  %s";

    private int mLastProcessState = -1;

    private void updatePendingConfiguration(Configuration config) {
        synchronized (mResourcesManager) {
            if (mPendingConfiguration == null ||
                    mPendingConfiguration.isOtherSeqNewer(config)) {
                mPendingConfiguration = config;
            }
        }
    }

    .....

            // we use token to identify this activity without having to send the
    // activity itself back to the activity manager. (matters more with ipc)
    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        updateProcessState(procState, false);

        ActivityClientRecord r = new ActivityClientRecord();

        r.token = token;
        r.ident = ident;
        r.intent = intent;
        r.referrer = referrer;
        r.voiceInteractor = voiceInteractor;
        r.activityInfo = info;
        r.compatInfo = compatInfo;
        r.state = state;
        r.persistentState = persistentState;

        r.pendingResults = pendingResults;
        r.pendingIntents = pendingNewIntents;

        r.startsNotResumed = notResumed;
        r.isForward = isForward;

        r.profilerInfo = profilerInfo;

        r.overrideConfig = overrideConfig;
        updatePendingConfiguration(curConfig);

        sendMessage(H.LAUNCH_ACTIVITY, r);
    }

    .....
}
```

#### H.handleMessage

ActivivtyThread内部类 H

```java
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    public static final int PAUSE_ACTIVITY          = 101;
    public static final int PAUSE_ACTIVITY_FINISHING= 102;
    public static final int STOP_ACTIVITY_SHOW      = 103;
    public static final int STOP_ACTIVITY_HIDE      = 104;
    public static final int SHOW_WINDOW             = 105;
    public static final int HIDE_WINDOW             = 106;
    public static final int RESUME_ACTIVITY         = 107;
    public static final int SEND_RESULT             = 108;
    public static final int DESTROY_ACTIVITY        = 109;
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int NEW_INTENT              = 112;
    public static final int RECEIVER                = 113;
    public static final int CREATE_SERVICE          = 114;
    public static final int SERVICE_ARGS            = 115;
    public static final int STOP_SERVICE            = 116;

    public static final int CONFIGURATION_CHANGED   = 118;
    public static final int CLEAN_UP_CONTEXT        = 119;
    public static final int GC_WHEN_IDLE            = 120;
    public static final int BIND_SERVICE            = 121;
    public static final int UNBIND_SERVICE          = 122;
    public static final int DUMP_SERVICE            = 123;
    public static final int LOW_MEMORY              = 124;
    public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
    public static final int RELAUNCH_ACTIVITY       = 126;
    public static final int PROFILER_CONTROL        = 127;
    public static final int CREATE_BACKUP_AGENT     = 128;
    public static final int DESTROY_BACKUP_AGENT    = 129;
    public static final int SUICIDE                 = 130;
    public static final int REMOVE_PROVIDER         = 131;
    public static final int ENABLE_JIT              = 132;
    public static final int DISPATCH_PACKAGE_BROADCAST = 133;
    public static final int SCHEDULE_CRASH          = 134;
    public static final int DUMP_HEAP               = 135;
    public static final int DUMP_ACTIVITY           = 136;
    public static final int SLEEPING                = 137;
    public static final int SET_CORE_SETTINGS       = 138;
    public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
    public static final int TRIM_MEMORY             = 140;
    public static final int DUMP_PROVIDER           = 141;
    public static final int UNSTABLE_PROVIDER_DIED  = 142;
    public static final int REQUEST_ASSIST_CONTEXT_EXTRAS = 143;
    public static final int TRANSLUCENT_CONVERSION_COMPLETE = 144;
    public static final int INSTALL_PROVIDER        = 145;
    public static final int ON_NEW_ACTIVITY_OPTIONS = 146;
    public static final int CANCEL_VISIBLE_BEHIND = 147;
    public static final int BACKGROUND_VISIBLE_BEHIND_CHANGED = 148;
    public static final int ENTER_ANIMATION_COMPLETE = 149;

    String codeToString(int code) {
        if (DEBUG_MESSAGES) {
            switch (code) {
                case LAUNCH_ACTIVITY: return "LAUNCH_ACTIVITY";
                case PAUSE_ACTIVITY: return "PAUSE_ACTIVITY";
                case PAUSE_ACTIVITY_FINISHING: return "PAUSE_ACTIVITY_FINISHING";
                case STOP_ACTIVITY_SHOW: return "STOP_ACTIVITY_SHOW";
                case STOP_ACTIVITY_HIDE: return "STOP_ACTIVITY_HIDE";
                case SHOW_WINDOW: return "SHOW_WINDOW";
                case HIDE_WINDOW: return "HIDE_WINDOW";
                case RESUME_ACTIVITY: return "RESUME_ACTIVITY";
                case SEND_RESULT: return "SEND_RESULT";
                case DESTROY_ACTIVITY: return "DESTROY_ACTIVITY";
                case BIND_APPLICATION: return "BIND_APPLICATION";
                case EXIT_APPLICATION: return "EXIT_APPLICATION";
                case NEW_INTENT: return "NEW_INTENT";
                case RECEIVER: return "RECEIVER";
                case CREATE_SERVICE: return "CREATE_SERVICE";
                case SERVICE_ARGS: return "SERVICE_ARGS";
                case STOP_SERVICE: return "STOP_SERVICE";
                case CONFIGURATION_CHANGED: return "CONFIGURATION_CHANGED";
                case CLEAN_UP_CONTEXT: return "CLEAN_UP_CONTEXT";
                case GC_WHEN_IDLE: return "GC_WHEN_IDLE";
                case BIND_SERVICE: return "BIND_SERVICE";
                case UNBIND_SERVICE: return "UNBIND_SERVICE";
                case DUMP_SERVICE: return "DUMP_SERVICE";
                case LOW_MEMORY: return "LOW_MEMORY";
                case ACTIVITY_CONFIGURATION_CHANGED: return "ACTIVITY_CONFIGURATION_CHANGED";
                case RELAUNCH_ACTIVITY: return "RELAUNCH_ACTIVITY";
                case PROFILER_CONTROL: return "PROFILER_CONTROL";
                case CREATE_BACKUP_AGENT: return "CREATE_BACKUP_AGENT";
                case DESTROY_BACKUP_AGENT: return "DESTROY_BACKUP_AGENT";
                case SUICIDE: return "SUICIDE";
                case REMOVE_PROVIDER: return "REMOVE_PROVIDER";
                case ENABLE_JIT: return "ENABLE_JIT";
                case DISPATCH_PACKAGE_BROADCAST: return "DISPATCH_PACKAGE_BROADCAST";
                case SCHEDULE_CRASH: return "SCHEDULE_CRASH";
                case DUMP_HEAP: return "DUMP_HEAP";
                case DUMP_ACTIVITY: return "DUMP_ACTIVITY";
                case SLEEPING: return "SLEEPING";
                case SET_CORE_SETTINGS: return "SET_CORE_SETTINGS";
                case UPDATE_PACKAGE_COMPATIBILITY_INFO: return "UPDATE_PACKAGE_COMPATIBILITY_INFO";
                case TRIM_MEMORY: return "TRIM_MEMORY";
                case DUMP_PROVIDER: return "DUMP_PROVIDER";
                case UNSTABLE_PROVIDER_DIED: return "UNSTABLE_PROVIDER_DIED";
                case REQUEST_ASSIST_CONTEXT_EXTRAS: return "REQUEST_ASSIST_CONTEXT_EXTRAS";
                case TRANSLUCENT_CONVERSION_COMPLETE: return "TRANSLUCENT_CONVERSION_COMPLETE";
                case INSTALL_PROVIDER: return "INSTALL_PROVIDER";
                case ON_NEW_ACTIVITY_OPTIONS: return "ON_NEW_ACTIVITY_OPTIONS";
                case CANCEL_VISIBLE_BEHIND: return "CANCEL_VISIBLE_BEHIND";
                case BACKGROUND_VISIBLE_BEHIND_CHANGED: return "BACKGROUND_VISIBLE_BEHIND_CHANGED";
                case ENTER_ANIMATION_COMPLETE: return "ENTER_ANIMATION_COMPLETE";
            }
        }
        return Integer.toString(code);
    }
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            .....
            }
        }
    .....
    }
}
```

#### ActivityThread.handleLaunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    // 看下节
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // 调用Activity.onResume()
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);

        if (!r.activity.mFinished && r.startsNotResumed) {
            // The activity manager actually wants this one to start out
            // paused, because it needs to be visible but isn't in the
            // foreground.  We accomplish this by going through the
            // normal startup (because activities expect to go through
            // onResume() the first time they run, before their window
            // is displayed), and then pausing it.  However, in this case
            // we do -not- need to do the full pause cycle (of freezing
            // and such) because the activity manager assumes it can just
            // retain the current state it has.
            try {
                r.activity.mCalled = false;
                mInstrumentation.callActivityOnPause(r.activity);
                // We need to keep around the original state, in case
                // we need to be created again.  But we only do this
                // for pre-Honeycomb apps, which always save their state
                // when pausing, so we can not have them save their state
                // when restarting from a paused state.  For HC and later,
                // we want to (and can) let the state be saved as the normal
                // part of stopping the activity.
                if (r.isPreHoneycomb()) {
                    r.state = oldState;
                }
                if (!r.activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPause()");
                }

            } catch (SuperNotCalledException e) {
                throw e;

            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                            "Unable to pause activity "
                            + r.intent.getComponent().toShortString()
                            + ": " + e.toString(), e);
                }
            }
            r.paused = true;
        }
    } else {
        // If there was an error, for any reason, tell the activity
        // manager to stop us.
        try {
            ActivityManagerNative.getDefault()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
        } catch (RemoteException ex) {
            // Ignore
        }
    }
}
```

#### ActivityThread.performLaunchActivity

```java
private Activity handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    Activity activity = null;
    try {
        // 获取类加载器
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        // 通过Instrumentation创建Activity对象
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        // 获取Application对象
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            
            // 调用Activity.attach()，在里面创建DecorView
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                // 里面调用onCreate()
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                // 里面调用onStart
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                            r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}
```

## 三、总结



## 四、参考链接

- [Gityuan - startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)

