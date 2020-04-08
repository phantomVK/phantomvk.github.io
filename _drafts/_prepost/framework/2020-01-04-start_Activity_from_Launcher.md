---
layout:     post
title:      "从Launcher启动Activity"
date:       2020-04-07
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## Launcher到AMS

__Launcher__ 启动后加载应用信息，最终所有应用快捷图标展示到桌面上。而 __Launcher__ 本身也是 __Activity__，所以点击图标启动新 __Activity__ 页面，和应用内启动界面的差别不会非常大。

Anroid 8.0.0_r44

> packages/apps/Launcher3/src/com/android/launcher3/Launcher.java

```java
/**
 * Launches the intent referred by the clicked shortcut.
 *
 * @param v The view representing the clicked shortcut.
 */
public void onClick(View v) {
    // Make sure that rogue clicks don't get through while allapps is launching, or after the
    // view has detached (it's possible for this to happen if the view is removed mid touch).
    if (v.getWindowToken() == null) {
        return;
    }

    if (!mWorkspace.isFinishedSwitchingState()) {
        return;
    }

    if (v instanceof Workspace) {
        if (mWorkspace.isInOverviewMode()) {
            getUserEventDispatcher().logActionOnContainer(LauncherLogProto.Action.Type.TOUCH,
                    LauncherLogProto.Action.Direction.NONE,
                    LauncherLogProto.ContainerType.OVERVIEW, mWorkspace.getCurrentPage());
            showWorkspace(true);
        }
        return;
    }

    if (v instanceof CellLayout) {
        if (mWorkspace.isInOverviewMode()) {
            int page = mWorkspace.indexOfChild(v);
            getUserEventDispatcher().logActionOnContainer(LauncherLogProto.Action.Type.TOUCH,
                    LauncherLogProto.Action.Direction.NONE,
                    LauncherLogProto.ContainerType.OVERVIEW, page);
            mWorkspace.snapToPageFromOverView(page);
            showWorkspace(true);
        }
        return;
    }

    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        onClickAppShortcut(v);
    } else if (tag instanceof FolderInfo) {
        if (v instanceof FolderIcon) {
            onClickFolderIcon(v);
        }
    } else if ((FeatureFlags.LAUNCHER3_ALL_APPS_PULL_UP && v instanceof PageIndicator) ||
            (v == mAllAppsButton && mAllAppsButton != null)) {
        onClickAllAppsButton(v);
    } else if (tag instanceof AppInfo) {
        startAppShortcutOrInfoActivity(v);
    } else if (tag instanceof LauncherAppWidgetInfo) {
        if (v instanceof PendingAppWidgetHostView) {
            onClickPendingWidget((PendingAppWidgetHostView) v);
        }
    }
}
```

```java
/**
 * Event handler for an app shortcut click.
 *
 * @param v The view that was clicked. Must be a tagged with a {@link ShortcutInfo}.
 */
protected void onClickAppShortcut(final View v) {
    if (LOGD) Log.d(TAG, "onClickAppShortcut");
    Object tag = v.getTag();
    if (!(tag instanceof ShortcutInfo)) {
        throw new IllegalArgumentException("Input must be a Shortcut");
    }

    // Open shortcut
    final ShortcutInfo shortcut = (ShortcutInfo) tag;

    if (shortcut.isDisabled != 0) {
        if ((shortcut.isDisabled &
                ~ShortcutInfo.FLAG_DISABLED_SUSPENDED &
                ~ShortcutInfo.FLAG_DISABLED_QUIET_USER) == 0) {
            // If the app is only disabled because of the above flags, launch activity anyway.
            // Framework will tell the user why the app is suspended.
        } else {
            if (!TextUtils.isEmpty(shortcut.disabledMessage)) {
                // Use a message specific to this shortcut, if it has one.
                Toast.makeText(this, shortcut.disabledMessage, Toast.LENGTH_SHORT).show();
                return;
            }
            // Otherwise just use a generic error message.
            int error = R.string.activity_not_available;
            if ((shortcut.isDisabled & ShortcutInfo.FLAG_DISABLED_SAFEMODE) != 0) {
                error = R.string.safemode_shortcut_error;
            } else if ((shortcut.isDisabled & ShortcutInfo.FLAG_DISABLED_BY_PUBLISHER) != 0 ||
                    (shortcut.isDisabled & ShortcutInfo.FLAG_DISABLED_LOCKED_USER) != 0) {
                error = R.string.shortcut_not_available;
            }
            Toast.makeText(this, error, Toast.LENGTH_SHORT).show();
            return;
        }
    }

    // Check for abandoned promise
    if ((v instanceof BubbleTextView) && shortcut.isPromise()) {
        String packageName = shortcut.intent.getComponent() != null ?
                shortcut.intent.getComponent().getPackageName() : shortcut.intent.getPackage();
        if (!TextUtils.isEmpty(packageName)) {
            onClickPendingAppItem(v, packageName,
                    shortcut.hasStatusFlag(ShortcutInfo.FLAG_INSTALL_SESSION_ACTIVE));
            return;
        }
    }

    // Start activities
    startAppShortcutOrInfoActivity(v);
}
```

```java
private void startAppShortcutOrInfoActivity(View v) {
    ItemInfo item = (ItemInfo) v.getTag();
    Intent intent = item.getIntent();
    if (intent == null) {
        throw new IllegalArgumentException("Input must have a valid intent");
    }
    boolean success = startActivitySafely(v, intent, item);
    getUserEventDispatcher().logAppLaunch(v, intent); // TODO for discovered apps b/35802115

    if (success && v instanceof BubbleTextView) {
        mWaitingForResume = (BubbleTextView) v;
        mWaitingForResume.setStayPressed(true);
    }
}
```

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    if (mIsSafeModeEnabled && !Utilities.isSystemApp(this, intent)) {
        Toast.makeText(this, R.string.safemode_shortcut_error, Toast.LENGTH_SHORT).show();
        return false;
    }
    // Only launch using the new animation if the shortcut has not opted out (this is a
    // private contract between launcher and may be ignored in the future).
    boolean useLaunchAnimation = (v != null) &&
            !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
    Bundle optsBundle = useLaunchAnimation ? getActivityLaunchOptions(v) : null;

    UserHandle user = item == null ? null : item.user;

    // Prepare intent
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    if (v != null) {
        intent.setSourceBounds(getViewBounds(v));
    }
    try {
        if (Utilities.ATLEAST_MARSHMALLOW
                && (item instanceof ShortcutInfo)
                && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                 || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                && !((ShortcutInfo) item).isPromise()) {
            // Shortcuts need some special checks due to legacy reasons.
            startShortcutIntentSafely(intent, optsBundle, item);
        } else if (user == null || user.equals(Process.myUserHandle())) {
            // Could be launching some bookkeeping activity
            startActivity(intent, optsBundle);
        } else {
            LauncherAppsCompat.getInstance(this).startActivityForProfile(
                    intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
        }
        return true;
    } catch (ActivityNotFoundException|SecurityException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Unable to launch. tag=" + item + " intent=" + intent, e);
    }
    return false;
}
```

> frameworks/base/core/java/android/app/Activity.java

```java
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
 * See {@link android.content.Context#startActivity(Intent, Bundle)}
 * Context.startActivity(Intent, Bundle)} for more details.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see #startActivity(Intent)
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

```java
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
 * are launching uses {@link Intent#FLAG_ACTIVITY_NEW_TASK}, it will not
 * run in your task and thus you will immediately receive a cancel result.
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
 * See {@link android.content.Context#startActivity(Intent, Bundle)}
 * Context.startActivity(Intent, Bundle)} for more details.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see #startActivity
 */
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
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
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
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

> frameworks/base/core/java/android/app/Instrumentation.java

```java
/**
 * Execute a startActivity call made by the application.  The default 
 * implementation takes care of updating any active {@link ActivityMonitor}
 * objects and dispatches this call to the system activity manager; you can
 * override this to watch for the application to start an activity, and 
 * modify what happens when it does. 
 *
 * <p>This method returns an {@link ActivityResult} object, which you can 
 * use when intercepting application calls to avoid performing the start 
 * activity action but still return the result the application is 
 * expecting.  To do this, override this method to catch the call to start 
 * activity so that it returns a new ActivityResult containing the results 
 * you would like the application to see, and don't call up to the super 
 * class.  Note that an application is only expecting a result if 
 * <var>requestCode</var> is &gt;= 0.
 *
 * <p>This method throws {@link android.content.ActivityNotFoundException}
 * if there was no Activity found to run the given Intent.
 *
 * @param who The Context from which the activity is being started.
 * @param contextThread The main thread of the Context from which the activity
 *                      is being started.
 * @param token Internal token identifying to the system who is starting 
 *              the activity; may be null.
 * @param target Which activity is performing the start (and thus receiving 
 *               any result); may be null if this call is not being made
 *               from an activity.
 * @param intent The actual Intent to start.
 * @param requestCode Identifier for this request's result; less than zero 
 *                    if the caller is not expecting a result.
 * @param options Addition options.
 *
 * @return To force the return of a particular result, return an 
 *         ActivityResult object containing the desired data; otherwise
 *         return null.  The default implementation always returns null.
 *
 * @throws android.content.ActivityNotFoundException
 *
 * @see Activity#startActivity(Intent)
 * @see Activity#startActivityForResult(Intent, int)
 * @see Activity#startActivityFromChild
 *
 * {@hide}
 */
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                ActivityResult result = null;
                if (am.ignoreMatchingSpecificIntents()) {
                    result = am.onStartActivity(intent);
                }
                if (result != null) {
                    am.mHits++;
                    return result;
                } else if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

> frameworks/base/core/java/android/app/ActivityManager.java

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, null,
            "startActivityAsUser");
}
```

> frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask, String reason) {
    return startActivityMayWait(caller, callingUid, PID_NULL, UserHandle.USER_NULL,
         callingPackage, intent, resolvedType, voiceSession, voiceInteractor, resultTo,
         resultWho, requestCode, startFlags, profilerInfo, outResult, globalConfig, bOptions,
         ignoreTargetSecurity, userId, iContainer, inTask, reason);
}

final int startActivityMayWait(IApplicationThread caller, int callingUid,
        int requestRealCallingPid, int requestRealCallingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask, String reason) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }
    mSupervisor.mActivityMetricsLogger.notifyActivityLaunching();
    boolean componentSpecified = intent.getComponent() != null;

    // Save a copy in case ephemeral needs it
    final Intent ephemeralIntent = new Intent(intent);
    // Don't modify the client's object!
    intent = new Intent(intent);
    if (componentSpecified
            && intent.getData() != null
            && Intent.ACTION_VIEW.equals(intent.getAction())
            && mService.getPackageManagerInternalLocked()
                    .isInstantAppInstallerComponent(intent.getComponent())) {
        // intercept intents targeted directly to the ephemeral installer the
        // ephemeral installer should never be started with a raw URL; instead
        // adjust the intent so it looks like a "normal" instant app launch
        intent.setComponent(null /*component*/);
        componentSpecified = false;
    }

    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
    if (rInfo == null) {
        UserInfo userInfo = mSupervisor.getUserInfo(userId);
        if (userInfo != null && userInfo.isManagedProfile()) {
            // Special case for managed profiles, if attempting to launch non-cryto aware
            // app in a locked managed profile from an unlocked parent allow it to resolve
            // as user will be sent via confirm credentials to unlock the profile.
            UserManager userManager = UserManager.get(mService.mContext);
            boolean profileLockedAndParentUnlockingOrUnlocked = false;
            long token = Binder.clearCallingIdentity();
            try {
                UserInfo parent = userManager.getProfileParent(userId);
                profileLockedAndParentUnlockingOrUnlocked = (parent != null)
                        && userManager.isUserUnlockingOrUnlocked(parent.id)
                        && !userManager.isUserUnlockingOrUnlocked(userId);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
            if (profileLockedAndParentUnlockingOrUnlocked) {
                rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                        PackageManager.MATCH_DIRECT_BOOT_AWARE
                                | PackageManager.MATCH_DIRECT_BOOT_UNAWARE);
            }
        }
    }
    // Collect information about the target of the Intent.
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

    ActivityOptions options = ActivityOptions.fromBundle(bOptions);
    ActivityStackSupervisor.ActivityContainer container =
            (ActivityStackSupervisor.ActivityContainer)iContainer;
    synchronized (mService) {
        if (container != null && container.mParentActivity != null &&
                container.mParentActivity.state != RESUMED) {
            // Cannot start a child activity if the parent is not resumed.
            return ActivityManager.START_CANCELED;
        }

        final int realCallingPid = requestRealCallingPid != PID_NULL
            ? requestRealCallingPid
            : Binder.getCallingPid();
        final int realCallingUid = requestRealCallingUid != UserHandle.USER_NULL
            ? requestRealCallingUid
            : Binder.getCallingUid();

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
            stack = mSupervisor.mFocusedStack;
        } else {
            stack = container.mStack;
        }
        stack.mConfigWillChange = globalConfig != null
                && mService.getGlobalConfiguration().diff(globalConfig) != 0;
        if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                "Starting activity when config will change = " + stack.mConfigWillChange);

        final long origId = Binder.clearCallingIdentity();

        if (aInfo != null &&
                (aInfo.applicationInfo.privateFlags
                        & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            // This may be a heavy-weight process!  Check to see if we already
            // have another, different heavy-weight process running.
            if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                final ProcessRecord heavy = mService.mHeavyWeightProcess;
                if (heavy != null && (heavy.info.uid != aInfo.applicationInfo.uid
                        || !heavy.processName.equals(aInfo.processName))) {
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
                    if (heavy.activities.size() > 0) {
                        ActivityRecord hist = heavy.activities.get(0);
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_APP,
                                hist.packageName);
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_TASK,
                                hist.getTask().taskId);
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
                    rInfo = mSupervisor.resolveIntent(intent, null /*resolvedType*/, userId);
                    aInfo = rInfo != null ? rInfo.activityInfo : null;
                    if (aInfo != null) {
                        aInfo = mService.getActivityInfoForUser(aInfo, userId);
                    }
                }
            }
        }

        final ActivityRecord[] outRecord = new ActivityRecord[1];
        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                inTask, reason);

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
            mService.updateConfigurationLocked(globalConfig, null, false);
        }

        if (outResult != null) {
            outResult.result = res;
            if (res == ActivityManager.START_SUCCESS) {
                mSupervisor.mWaitingActivityLaunched.add(outResult);
                do {
                    try {
                        mService.wait();
                    } catch (InterruptedException e) {
                    }
                } while (outResult.result != START_TASK_TO_FRONT
                        && !outResult.timeout && outResult.who == null);
                if (outResult.result == START_TASK_TO_FRONT) {
                    res = START_TASK_TO_FRONT;
                }
            }
            if (res == START_TASK_TO_FRONT) {
                final ActivityRecord r = outRecord[0];

                // ActivityRecord may represent a different activity, but it should not be in
                // the resumed state.
                if (r.nowVisible && r.state == RESUMED) {
                    outResult.timeout = false;
                    outResult.who = r.realActivity;
                    outResult.totalTime = 0;
                    outResult.thisTime = 0;
                } else {
                    outResult.thisTime = SystemClock.uptimeMillis();
                    mSupervisor.waitActivityVisible(r.realActivity, outResult);
                    // Note: the timeout variable is not currently not ever set.
                    do {
                        try {
                            mService.wait();
                        } catch (InterruptedException e) {
                        }
                    } while (!outResult.timeout && outResult.who == null);
                }
            }
        }

        mSupervisor.mActivityMetricsLogger.notifyActivityLaunched(res, outRecord[0]);
        return res;
    }
}

int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
        TaskRecord inTask, String reason) {

    if (TextUtils.isEmpty(reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            container, inTask);

    if (outActivity != null) {
        // mLastStartActivityRecord[0] is set in the call to startActivity above.
        outActivity[0] = mLastStartActivityRecord[0];
    }
    return mLastStartActivityResult;
}

/** DO NOT call this method directly. Use {@link #startActivityLocked} instead. */
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
        TaskRecord inTask) {
    int err = ActivityManager.START_SUCCESS;
    // Pull the optional Ephemeral Installer-only bundle out of the options early.
    final Bundle verificationBundle
            = options != null ? options.popAppVerificationBundle() : null;

    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            Slog.w(TAG, "Unable to find app for caller " + caller
                    + " (pid=" + callingPid + ") when starting: "
                    + intent.toString());
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }

    final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

    if (err == ActivityManager.START_SUCCESS) {
        Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                + "} from uid " + callingUid);
    }

    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
        if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                "Will send result to " + resultTo + " " + sourceRecord);
        if (sourceRecord != null) {
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }

    final int launchFlags = intent.getFlags();

    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
        // Transfer the result target from the source activity to the new
        // one being started, including any failures.
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

    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        // We couldn't find a class that can handle the given Intent.
        // That's the end of that!
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }

    if (err == ActivityManager.START_SUCCESS && aInfo == null) {
        // We couldn't find the specific class specified in the Intent.
        // Also the end of the line.
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }

    if (err == ActivityManager.START_SUCCESS && sourceRecord != null
            && sourceRecord.getTask().voiceSession != null) {
        // If this activity is being launched as part of a voice session, we need
        // to ensure that it is safe to do so.  If the upcoming activity will also
        // be part of the voice session, we can only launch it if it has explicitly
        // said it supports the VOICE category, or it is a part of the calling app.
        if ((launchFlags & FLAG_ACTIVITY_NEW_TASK) == 0
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

    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.getStack();

    if (err != START_SUCCESS) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(
                    -1, resultRecord, resultWho, requestCode, RESULT_CANCELED, null);
        }
        ActivityOptions.abort(options);
        return err;
    }

    boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
            requestCode, callingPid, callingUid, callingPackage, ignoreTargetSecurity, callerApp,
            resultRecord, resultStack, options);
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

    mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage);
    mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid, callingUid,
            options);
    intent = mInterceptor.mIntent;
    rInfo = mInterceptor.mRInfo;
    aInfo = mInterceptor.mAInfo;
    resolvedType = mInterceptor.mResolvedType;
    inTask = mInterceptor.mInTask;
    callingPid = mInterceptor.mCallingPid;
    callingUid = mInterceptor.mCallingUid;
    options = mInterceptor.mActivityOptions;
    if (abort) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                    RESULT_CANCELED, null);
        }
        // We pretend to the caller that it was really started, but
        // they will just get a cancel result.
        ActivityOptions.abort(options);
        return START_SUCCESS;
    }

    // If permissions need a review before any of the app components can run, we
    // launch the review activity and pass a pending intent to start the activity
    // we are to launching now after the review is completed.
    if (mService.mPermissionReviewRequired && aInfo != null) {
        if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                aInfo.packageName, userId)) {
            IIntentSender target = mService.getIntentSenderLocked(
                    ActivityManager.INTENT_SENDER_ACTIVITY, callingPackage,
                    callingUid, userId, null, null, 0, new Intent[]{intent},
                    new String[]{resolvedType}, PendingIntent.FLAG_CANCEL_CURRENT
                            | PendingIntent.FLAG_ONE_SHOT, null);

            final int flags = intent.getFlags();
            Intent newIntent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
            newIntent.setFlags(flags
                    | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
            newIntent.putExtra(Intent.EXTRA_PACKAGE_NAME, aInfo.packageName);
            newIntent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
            if (resultRecord != null) {
                newIntent.putExtra(Intent.EXTRA_RESULT_NEEDED, true);
            }
            intent = newIntent;

            resolvedType = null;
            callingUid = realCallingUid;
            callingPid = realCallingPid;

            rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
            aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags,
                    null /*profilerInfo*/);

            if (DEBUG_PERMISSIONS_REVIEW) {
                Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true,
                        true, false) + "} from uid " + callingUid + " on display "
                        + (container == null ? (mSupervisor.mFocusedStack == null ?
                        DEFAULT_DISPLAY : mSupervisor.mFocusedStack.mDisplayId) :
                        (container.mActivityDisplay == null ? DEFAULT_DISPLAY :
                                container.mActivityDisplay.mDisplayId)));
            }
        }
    }

    // If we have an ephemeral app, abort the process of launching the resolved intent.
    // Instead, launch the ephemeral installer. Once the installer is finished, it
    // starts either the intent we resolved here [on install error] or the ephemeral
    // app [on install success].
    if (rInfo != null && rInfo.auxiliaryInfo != null) {
        intent = createLaunchIntent(rInfo.auxiliaryInfo, ephemeralIntent,
                callingPackage, verificationBundle, resolvedType, userId);
        resolvedType = null;
        callingUid = realCallingUid;
        callingPid = realCallingPid;

        aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, null /*profilerInfo*/);
    }

    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, container, options, sourceRecord);
    if (outActivity != null) {
        outActivity[0] = r;
    }

    if (r.appTimeTracker == null && sourceRecord != null) {
        // If the caller didn't specify an explicit time tracker, we want to continue
        // tracking under any it has.
        r.appTimeTracker = sourceRecord.appTimeTracker;
    }

    final ActivityStack stack = mSupervisor.mFocusedStack;
    if (voiceSession == null && (stack.mResumedActivity == null
            || stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
        if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                realCallingPid, realCallingUid, "Activity start")) {
            PendingActivityLaunch pal =  new PendingActivityLaunch(r,
                    sourceRecord, startFlags, stack, callerApp);
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

    doPendingActivityLaunchesLocked(false);

    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
            options, inTask, outActivity);
}


private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    } finally {
        // If we are not able to proceed, disassociate the activity from the task. Leaving an
        // activity in an incomplete state can lead to issues, such as performing operations
        // without a window container.
        if (!ActivityManager.isStartResultSuccessful(result)
                && mStartActivity.getTask() != null) {
            mStartActivity.getTask().removeActivity(mStartActivity);
        }
        mService.mWindowManager.continueSurfaceLayout();
    }

    postStartActivityProcessing(r, result, mSupervisor.getLastStack().mStackId,  mSourceRecord,
            mTargetStack);

    return result;
}

// Note: This method should only be called from {@link startActivity}.
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {

    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor);

    computeLaunchingTaskFlags();

    computeSourceStack();

    mIntent.setFlags(mLaunchFlags);

    ActivityRecord reusedActivity = getReusableIntentActivity();

    final int preferredLaunchStackId =
            (mOptions != null) ? mOptions.getLaunchStackId() : INVALID_STACK_ID;
    final int preferredLaunchDisplayId =
            (mOptions != null) ? mOptions.getLaunchDisplayId() : DEFAULT_DISPLAY;

    if (reusedActivity != null) {
        // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
        // still needs to be a lock task mode violation since the task gets cleared out and
        // the device would otherwise leave the locked task.
        if (mSupervisor.isLockTaskModeViolation(reusedActivity.getTask(),
                (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
            mSupervisor.showLockTaskToast();
            Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }

        if (mStartActivity.getTask() == null) {
            mStartActivity.setTask(reusedActivity.getTask());
        }
        if (reusedActivity.getTask().intent == null) {
            // This task was started because of movement of the activity based on affinity...
            // Now that we are actually launching it, we can assign the base intent.
            reusedActivity.getTask().setIntent(mStartActivity);
        }

        // This code path leads to delivering a new intent, we want to make sure we schedule it
        // as the first operation, in case the activity will be resumed as a result of later
        // operations.
        if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || mLaunchSingleInstance || mLaunchSingleTask) {
            final TaskRecord task = reusedActivity.getTask();

            // In this situation we want to remove all activities from the task up to the one
            // being started. In most cases this means we are resetting the task to its initial
            // state.
            final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                    mLaunchFlags);

            // The above code can remove {@code reusedActivity} from the task, leading to the
            // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}. The
            // task reference is needed in the call below to
            // {@link setTargetStackAndMoveToFrontIfNeeded}.
            if (reusedActivity.getTask() == null) {
                reusedActivity.setTask(task);
            }

            if (top != null) {
                if (top.frontOfTask) {
                    // Activity aliases may mean we use different intents for the top activity,
                    // so make sure the task now has the identity of the new intent.
                    top.getTask().setIntent(mStartActivity);
                }
                ActivityStack.logStartActivity(AM_NEW_INTENT, mStartActivity, top.getTask());
                top.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                        mStartActivity.launchedFromPackage);
            }
        }

        sendPowerHintForLaunchStartIfNeeded(false /* forceSend */);

        reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

        final ActivityRecord outResult =
                outActivity != null && outActivity.length > 0 ? outActivity[0] : null;

        // When there is a reused activity and the current result is a trampoline activity,
        // set the reused activity as the result.
        if (outResult != null && (outResult.finishing || outResult.noDisplay)) {
            outActivity[0] = reusedActivity;
        }

        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do anything
            // if that is the case, so this is it!  And for paranoia, make sure we have
            // correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            return START_RETURN_INTENT_TO_CALLER;
        }
        setTaskFromIntentActivity(reusedActivity);

        if (!mAddingToTask && mReuseTask == null) {
            // We didn't do anything...  but it was needed (a.k.a., client don't use that
            // intent!)  And for paranoia, make sure we have correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            if (outActivity != null && outActivity.length > 0) {
                outActivity[0] = reusedActivity;
            }
            return START_TASK_TO_FRONT;
        }
    }

    if (mStartActivity.packageName == null) {
        final ActivityStack sourceStack = mStartActivity.resultTo != null
                ? mStartActivity.resultTo.getStack() : null;
        if (sourceStack != null) {
            sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.resultTo,
                    mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                    null /* data */);
        }
        ActivityOptions.abort(mOptions);
        return START_CLASS_NOT_FOUND;
    }

    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    final ActivityStack topStack = mSupervisor.mFocusedStack;
    final ActivityRecord topFocused = topStack.topActivity();
    final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || mLaunchSingleTop || mLaunchSingleTask);
    if (dontStart) {
        ActivityStack.logStartActivity(AM_NEW_INTENT, top, top.getTask());
        // For paranoia, make sure we have correctly resumed the top activity.
        topStack.mLastPausedActivity = null;
        if (mDoResume) {
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }
        ActivityOptions.abort(mOptions);
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do
            // anything if that is the case, so this is it!
            return START_RETURN_INTENT_TO_CALLER;
        }
        top.deliverNewIntentLocked(
                mCallingUid, mStartActivity.intent, mStartActivity.launchedFromPackage);

        // Don't use mStartActivity.task to show the toast. We're not starting a new activity
        // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
        mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredLaunchStackId,
                preferredLaunchDisplayId, topStack.mStackId);

        return START_DELIVERED_TO_TOP;
    }

    boolean newTask = false;
    final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
            ? mSourceRecord.getTask() : null;

    // Should this be considered a new task?
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(
                taskToAffiliate, preferredLaunchStackId, topStack);
    } else if (mSourceRecord != null) {
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        result = setTaskFromInTask();
    } else {
        // This not being started from an existing activity, and not part of a new task...
        // just put it in the top task, though these days this case should never happen.
        setTaskToCurrentTopOrCreateNewTask();
    }
    if (result != START_SUCCESS) {
        return result;
    }

    mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
            mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);
    mService.grantEphemeralAccessLocked(mStartActivity.userId, mIntent,
            mStartActivity.appInfo.uid, UserHandle.getAppId(mCallingUid));
    if (mSourceRecord != null) {
        mStartActivity.getTask().setTaskToReturnTo(mSourceRecord);
    }
    if (newTask) {
        EventLog.writeEvent(
                EventLogTags.AM_CREATE_TASK, mStartActivity.userId,
                mStartActivity.getTask().taskId);
    }
    ActivityStack.logStartActivity(
            EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTask());
    mTargetStack.mLastPausedActivity = null;

    sendPowerHintForLaunchStartIfNeeded(false /* forceSend */);

    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
            mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            // If the activity is not focusable, we can't resume it, but still would like to
            // make sure it becomes visible as it starts (this will also trigger entry
            // animation). An example of this are PIP activities.
            // Also, we don't want to resume activities in a task that currently has an overlay
            // as the starting activity just needs to be in the visible paused state until the
            // over is removed.
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            // Go ahead and tell window manager to execute app transition for this activity
            // since the app transition will not be triggered through the resume channel.
            mWindowManager.executeAppTransition();
        } else {
            // If the target stack was not previously focusable (previous top running activity
            // on that stack was not visible) then any prior calls to move the stack to the
            // will not update the focused stack.  If starting the new activity now allows the
            // task stack to be focusable, then ensure that we now update the focused stack
            // accordingly.
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else {
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredLaunchStackId,
            preferredLaunchDisplayId, mTargetStack.mStackId);

    return START_SUCCESS;
}
```

> frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || r.state != RESUMED) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    } else if (r.state == RESUMED) {
        // Kick off any lingering app transitions form the MoveTaskToFront operation.
        mFocusedStack.executeAppTransition(targetOptions);
    }
    return false;
}
```

> frameworks/base/services/core/java/com/android/server/am/ActivityStack.java

```java
/**
 * Ensure that the top activity in the stack is resumed.
 *
 * @param prev The previously resumed activity, for when in the process
 * of pausing; can be null to call from elsewhere.
 * @param options Activity options.
 *
 * @return Returns true if something is being resumed, or false if
 * nothing happened.
 *
 * NOTE: It is not safe to call this method directly as it can cause an activity in a
 *       non-focused stack to be resumed.
 *       Use {@link ActivityStackSupervisor#resumeFocusedStackTopActivityLocked} to resume the
 *       right activity for the current system state.
 */
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    // When resuming the top activity, it may be necessary to pause the top activity (for
    // example, returning to the lock screen. We suppress the normal pause logic in
    // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the end.
    // We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here to ensure
    // any necessary pause logic occurs.
    mStackSupervisor.checkReadyForSleepLocked();

    return result;
}
```

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    if (!mService.mBooting && !mService.mBooted) {
        // Not ready yet!
        return false;
    }

    // Find the next top-most activity to resume in this stack that is not finishing and is
    // focusable. If it is not focusable, we will fall into the case below to resume the
    // top activity in the next focusable task.
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

    final boolean hasRunningActivity = next != null;

    final ActivityRecord parent = mActivityContainer.mParentActivity;
    final boolean isParentNotResumed = parent != null && parent.state != ActivityState.RESUMED;
    if (hasRunningActivity
            && (isParentNotResumed || !mActivityContainer.isAttachedLocked())) {
        // Do not resume this stack if its parent is not resumed.
        // TODO: If in a loop, make sure that parent stack resumeTopActivity is called 1st.
        return false;
    }

    mStackSupervisor.cancelInitializingActivities();

    // Remember how we'll process this pause/resume situation, and ensure
    // that the state is reset however we wind up proceeding.
    final boolean userLeaving = mStackSupervisor.mUserLeaving;
    mStackSupervisor.mUserLeaving = false;

    if (!hasRunningActivity) {
        // There are no activities left in the stack, let's look somewhere else.
        return resumeTopActivityInNextFocusableStack(prev, options, "noMoreActivities");
    }

    next.delayedResume = false;

    // If the top activity is the resumed one, nothing to do.
    if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                mStackSupervisor.allResumedActivitiesComplete()) {
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        executeAppTransition(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Top activity resumed " + next);
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    final TaskRecord nextTask = next.getTask();
    final TaskRecord prevTask = prev != null ? prev.getTask() : null;
    if (prevTask != null && prevTask.getStack() == this &&
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
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(prev, "prevFinished");
        }
    }

    // If we are sleeping, and there is no resumed activity, and the top
    // activity is paused, well that is the state we want.
    if (mService.isSleepingOrShuttingDownLocked()
            && mLastPausedActivity == next
            && mStackSupervisor.allPausedActivitiesComplete()) {
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        executeAppTransition(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Going to sleep and all paused");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    // Make sure that the user who owns this activity is started.  If not,
    // we will just leave it as is because someone should be bringing
    // another user's activities to the top of the stack.
    if (!mService.mUserController.hasStartedUserState(next.userId)) {
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
    mStackSupervisor.mActivitiesWaitingForVisibleActivity.remove(next);

    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resuming " + next);

    // If we are currently pausing an activity, then don't do anything until that is done.
    if (!mStackSupervisor.allPausedActivitiesComplete()) {
        if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                "resumeTopActivityLocked: Skip resume: some activity pausing.");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return false;
    }

    mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

    boolean lastResumedCanPip = false;
    final ActivityStack lastFocusedStack = mStackSupervisor.getLastStack();
    if (lastFocusedStack != null && lastFocusedStack != this) {
        // So, why aren't we using prev here??? See the param comment on the method. prev doesn't
        // represent the last resumed activity. However, the last focus stack does if it isn't null.
        final ActivityRecord lastResumed = lastFocusedStack.mResumedActivity;
        lastResumedCanPip = lastResumed != null && lastResumed.checkEnterPictureInPictureState(
                "resumeTopActivity", true /* noThrow */, userLeaving /* beforeStopping */);
    }
    // If the flag RESUME_WHILE_PAUSING is set, then continue to schedule the previous activity
    // to be paused, while at the same time resuming the new resume activity only if the
    // previous activity can't go into Pip since we want to give Pip activities a chance to
    // enter Pip before resuming the next activity.
    final boolean resumeWhilePausing = (next.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0
            && !lastResumedCanPip;

    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
    if (mResumedActivity != null) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Pausing " + mResumedActivity);
        pausing |= startPausingLocked(userLeaving, false, next, false);
    }
    if (pausing && !resumeWhilePausing) {
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
    } else if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
            mStackSupervisor.allResumedActivitiesComplete()) {
        // It is possible for the activity to be resumed when we paused back stacks above if the
        // next activity doesn't have to wait for pause to complete.
        // So, nothing else to-do except:
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        executeAppTransition(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Top activity resumed (dontWaitForPause) " + next);
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
    }

    // If the most recent activity was noHistory but was only stopped rather
    // than stopped+finished because the device went to sleep, we need to make
    // sure to finish it as we're making a new activity topmost.
    if (mService.isSleepingLocked() && mLastNoHistoryActivity != null &&
            !mLastNoHistoryActivity.finishing) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "no-history finish of " + mLastNoHistoryActivity + " on new resume");
        requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                null, "resume-no-history", false);
        mLastNoHistoryActivity = null;
    }

    if (prev != null && prev != next) {
        if (!mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                && next != null && !next.nowVisible) {
            mStackSupervisor.mActivitiesWaitingForVisibleActivity.add(prev);
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
                prev.setVisibility(false);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Not waiting for visible to hide: " + prev + ", waitingVisible="
                        + mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                        + ", nowVisible=" + next.nowVisible);
            } else {
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Previous already visible but still waiting to hide: " + prev
                        + ", waitingVisible="
                        + mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
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
                mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(prev.getTask() == next.getTask()
                        ? TRANSIT_ACTIVITY_CLOSE
                        : TRANSIT_TASK_CLOSE, false);
            }
            prev.setVisibility(false);
        } else {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: prev=" + prev);
            if (mNoAnimActivities.contains(next)) {
                anim = false;
                mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(prev.getTask() == next.getTask()
                        ? TRANSIT_ACTIVITY_OPEN
                        : next.mLaunchTaskBehind
                                ? TRANSIT_TASK_OPEN_BEHIND
                                : TRANSIT_TASK_OPEN, false);
            }
        }
    } else {
        if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, "Prepare open transition: no previous");
        if (mNoAnimActivities.contains(next)) {
            anim = false;
            mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
        } else {
            mWindowManager.prepareAppTransition(TRANSIT_ACTIVITY_OPEN, false);
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
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next
                + " stopped=" + next.stopped + " visible=" + next.visible);

        // If the previous activity is translucent, force a visibility update of
        // the next activity, so that it's added to WM's opening app list, and
        // transition animation can be set up properly.
        // For example, pressing Home button with a translucent activity in focus.
        // Launcher is already visible in this case. If we don't add it to opening
        // apps, maybeUpdateTransitToWallpaper() will fail to identify this as a
        // TRANSIT_WALLPAPER_OPEN animation, and run some funny animation.
        final boolean lastActivityTranslucent = lastStack != null
                && (!lastStack.mFullscreen
                || (lastStack.mLastPausedActivity != null
                && !lastStack.mLastPausedActivity.fullscreen));

        // This activity is now becoming visible.
        if (!next.visible || next.stopped || lastActivityTranslucent) {
            next.setVisibility(true);
        }

        // schedule launch ticks to collect information about slow apps.
        next.startLaunchTickingLocked();

        ActivityRecord lastResumedActivity =
                lastStack == null ? null :lastStack.mResumedActivity;
        ActivityState lastState = next.state;

        mService.updateCpuStats();

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next + " (in existing)");

        setResumedActivityLocked(next, "resumeTopActivityInnerLocked");

        mService.updateLruProcessLocked(next.app, true, null);
        updateLRUListLocked(next);
        mService.updateOomAdjLocked();

        // Have the window manager re-evaluate the orientation of
        // the screen based on the new activity order.
        boolean notUpdated = true;
        if (mStackSupervisor.isFocusedStack(this)) {
            final Configuration config = mWindowManager.updateOrientationFromAppTokens(
                    mStackSupervisor.getDisplayOverrideConfiguration(mDisplayId),
                    next.mayFreezeScreenLocked(next.app) ? next.appToken : null, mDisplayId);
            if (config != null) {
                next.frozenBeforeDestroy = true;
            }
            notUpdated = !mService.updateDisplayOverrideConfigurationLocked(config, next,
                    false /* deferResume */, mDisplayId);
        }

        if (notUpdated) {
            // The configuration update wasn't able to keep the existing
            // instance of the activity, and instead started a new one.
            // We should be all done, but let's just make sure our activity
            // is still at the top and schedule another run if something
            // weird happened.
            ActivityRecord nextNext = topRunningActivityLocked();
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                    "Activity config changed during resume: " + next
                    + ", new next: " + nextNext);
            if (nextNext != next) {
                // Do over!
                mStackSupervisor.scheduleResumeTopActivities();
            }
            if (!next.visible || next.stopped) {
                next.setVisibility(true);
            }
            next.completeResumeLocked();
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
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
                next.app.thread.scheduleNewIntent(
                        next.newIntents, next.appToken, false /* andPause */);
            }

            // Well the app will no longer be stopped.
            // Clear app token stopped state in window manager if needed.
            next.notifyAppResumed(next.stopped);

            EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                    System.identityHashCode(next), next.getTask().taskId,
                    next.shortComponentName);

            next.sleeping = false;
            mService.showUnsupportedZoomDialogIfNeededLocked(next);
            mService.showAskCompatModeDialogLocked(next);
            next.app.pendingUiClean = true;
            next.app.forceProcessStateUpTo(mService.mTopProcessState);
            next.clearOptionsLocked();
            next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                    mService.isNextTransitionForward(), resumeAnimOptions);

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
                    mStackSupervisor.isFrontStackOnDisplay(lastStack)) {
                next.showStartingWindow(null /* prev */, false /* newTask */,
                        false /* taskSwitch */);
            }
            mStackSupervisor.startSpecificActivityLocked(next, true, false);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        // From this point on, if something goes wrong there is no way
        // to recover the activity.
        try {
            next.completeResumeLocked();
        } catch (Exception e) {
            // If any exception gets thrown, toss away this
            // activity and try the next one.
            Slog.w(TAG, "Exception thrown during resume of " + next, e);
            requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                    "resume-exception", true);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }
    } else {
        // Whoops, need to restart this activity!
        if (!next.hasBeenLaunched) {
            next.hasBeenLaunched = true;
        } else {
            if (SHOW_APP_STARTING_PREVIEW) {
                next.showStartingWindow(null /* prev */, false /* newTask */,
                        false /* taskSwich */);
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

> frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

```java
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    r.getStack().setLaunchTime(r);

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
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}

final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {

    if (!allPausedActivitiesComplete()) {
        // While there are activities pausing we skipping starting any new activities until
        // pauses are complete. NOTE: that we also do this for activities that are starting in
        // the paused state because they will first be resumed then paused on the client side.
        if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                "realStartActivityLocked: Skipping start of r=" + r
                + " some activities pausing...");
        return false;
    }

    r.startFreezingScreenLocked(app, 0);
    if (r.getStack().checkKeyguardVisibility(r, true /* shouldBeVisible */, true /* isTop */)) {
        // We only set the visibility to true if the activity is allowed to be visible based on
        // keyguard state. This avoids setting this into motion in window manager that is later
        // cancelled due to later calls to ensure visible activities that set visibility back to
        // false.
        r.setVisibility(true);
    }

    // schedule launch ticks to collect information about slow apps.
    r.startLaunchTickingLocked();

    // Have the window manager re-evaluate the orientation of the screen based on the new
    // activity order.  Note that as a result of this, it can call back into the activity
    // manager with a new orientation.  We don't care about that, because the activity is not
    // currently running so we are just restarting it anyway.
    if (checkConfig) {
        final int displayId = r.getDisplayId();
        final Configuration config = mWindowManager.updateOrientationFromAppTokens(
                getDisplayOverrideConfiguration(displayId),
                r.mayFreezeScreenLocked(app) ? r.appToken : null, displayId);
        // Deferring resume here because we're going to launch new activity shortly.
        // We don't want to perform a redundant launch of the same record while ensuring
        // configurations and trying to resume top activity of focused stack.
        mService.updateDisplayOverrideConfigurationLocked(config, r, true /* deferResume */,
                displayId);
    }

    if (mKeyguardController.isKeyguardLocked()) {
        r.notifyUnknownVisibilityLaunched();
    }
    final int applicationInfoUid =
            (r.info.applicationInfo != null) ? r.info.applicationInfo.uid : -1;
    if ((r.userId != app.userId) || (r.appInfo.uid != applicationInfoUid)) {
        Slog.wtf(TAG,
                "User ID for activity changing for " + r
                        + " appInfo.uid=" + r.appInfo.uid
                        + " info.ai.uid=" + applicationInfoUid
                        + " old=" + r.app + " new=" + app);
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

    final TaskRecord task = r.getTask();
    if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE ||
            task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV) {
        setLockTaskModeLocked(task, LOCK_TASK_MODE_LOCKED, "mLockTaskAuth==LAUNCHABLE", false);
    }

    final ActivityStack stack = task.getStack();
    try {
        if (app.thread == null) {
            throw new RemoteException();
        }
        List<ResultInfo> results = null;
        List<ReferrerIntent> newIntents = null;
        if (andResume) {
            // We don't need to deliver new intents and/or set results if activity is going
            // to pause immediately after launch.
            results = r.results;
            newIntents = r.newIntents;
        }
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                + " newIntents=" + newIntents + " andResume=" + andResume);
        EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY, r.userId,
                System.identityHashCode(r), task.taskId, r.shortComponentName);
        if (r.isHomeActivity()) {
            // Home process is the root process of the task.
            mService.mHomeProcess = task.mActivities.get(0).app;
        }
        mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                                  PackageManager.NOTIFY_PACKAGE_USE_ACTIVITY);
        r.sleeping = false;
        r.forceNewConfig = false;
        mService.showUnsupportedZoomDialogIfNeededLocked(r);
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
                            mService.mSamplingInterval, mService.mAutoStopProfiler,
                            mService.mStreamingOutput);
                }
            }
        }

        app.hasShownUi = true;
        app.pendingUiClean = true;
        app.forceProcessStateUpTo(mService.mTopProcessState);
        // Because we could be starting an Activity in the system process this may not go across
        // a Binder interface which would create a new Configuration. Consequently we have to
        // always create a new Configuration here.

        final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                mService.getGlobalConfiguration(), r.getMergedOverrideConfiguration());
        r.setLastReportedConfiguration(mergedConfiguration);

        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,
                // TODO: Have this take the merged configuration instead of separate global and
                // override configs.
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                r.persistentState, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profilerInfo);

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
        r.launchFailed = true;
        app.activities.remove(r);
        throw e;
    }

    r.launchFailed = false;
    if (stack.updateLRUListLocked(r)) {
        Slog.w(TAG, "Activity " + r + " being launched, but already in LRU list");
    }

    if (andResume) {
        // As part of the process of launching, ActivityThread also performs
        // a resume.
        stack.minimalResumeActivityLocked(r);
    } else {
        // This activity is not starting in the resumed state... which should look like we asked
        // it to pause+stop (but remain visible), and it has done so and reported back the
        // current icicle and other state.
        if (DEBUG_STATES) Slog.v(TAG_STATES,
                "Moving to PAUSED: " + r + " (starting in paused state)");
        r.state = PAUSED;
    }

    // Launch the new version setup screen if needed.  We do this -after-
    // launching the initial activity (that is, home), so that it can have
    // a chance to initialize itself while in the background, making the
    // switch back to it faster and look better.
    if (isFocusedStack(stack)) {
        mService.startSetupActivityLocked();
    }

    // Update any services we are bound to that might care about whether
    // their client may have activities.
    if (r.app != null) {
        mService.mServices.updateServiceConnectionActivitiesLocked(r.app);
    }

    return true;
}
```

## AMS到应用进程

> frameworks/base/core/java/android/app/ActivityThread.java

内部类：ApplicationThread

```java
public final class ActivityThread {
    private class ApplicationThread extends IApplicationThread.Stub {
        ....

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

        ....
    }
}
```

> frameworks/base/core/java/android/app/ActivityThread.java

内部类H

```java
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    ....

    String codeToString(int code) {
        if (DEBUG_MESSAGES) {
            switch (code) {
                case LAUNCH_ACTIVITY: return "LAUNCH_ACTIVITY";
                ....
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
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            ....
        }
        Object obj = msg.obj;
        if (obj instanceof SomeArgs) {
            ((SomeArgs) obj).recycle();
        }
        if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
    }

    ....
}
```

> frameworks/base/core/java/android/app/ActivityThread.java

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
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

    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        if (!r.activity.mFinished && r.startsNotResumed) {
            // The activity manager actually wants this one to start out paused, because it
            // needs to be visible but isn't in the foreground. We accomplish this by going
            // through the normal startup (because activities expect to go through onResume()
            // the first time they run, before their window is displayed), and then pausing it.
            // However, in this case we do -not- need to do the full pause cycle (of freezing
            // and such) because the activity manager assumes it can just retain the current
            // state it has.
            performPauseActivityIfNeeded(r, reason);

            // We need to keep around the original state, in case we need to be created again.
            // But we only do this for pre-Honeycomb apps, which always save their state when
            // pausing, so we can not have them save their state when restarting from a paused
            // state. For HC and later, we want to (and can) let the state be saved as the
            // normal part of stopping the activity.
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
        try {
            ActivityManager.getService()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

> frameworks/base/core/java/android/app/ActivityThread.java

方法

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

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

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
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
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
        if (localLOGV) Slog.v(
                TAG, r + ": app=" + app
                + ", appName=" + app.getPackageName()
                + ", pkg=" + r.packageInfo.getPackageName()
                + ", comp=" + r.intent.getComponent().toShortString()
                + ", dir=" + r.packageInfo.getAppDir());

        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
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

> frameworks/base/core/java/android/app/Instrumentation.java

```java
/**
 * Perform calling of an activity's {@link Activity#onCreate}
 * method.  The default implementation simply calls through to that method.
 *  @param activity The activity being created.
 * @param icicle The previously frozen state (or null) to pass through to
 * @param persistentState The previously persisted state (or null)
 */
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

> frameworks/base/core/java/android/app/Activity.java

```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle, persistentState);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```

