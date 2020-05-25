---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---



> frameworks/base/core/java/android/content/ContextWrapper.java

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

    @Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
}
```





> frameworks/base/core/java/android/app/ContextImpl.java

```java
@Override
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, false, mUser);
}

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        ComponentName cn = ActivityManager.getService().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), requireForeground,
                        getOpPackageName(), user.getIdentifier());
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException(
                        "Unable to start service " + service
                        + ": " + cn.getClassName());
            } else if (cn.getPackageName().equals("?")) {
                throw new IllegalStateException(
                        "Not allowed to start service " + service + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```





> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
    @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, boolean requireForeground, String callingPackage, int userId)
            throws TransactionTooLargeException {
        enforceNotIsolatedCaller("startService");
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                "*** startService: " + service + " type=" + resolvedType + " fg=" + requireForeground);
        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res;
            try {
                res = mServices.startServiceLocked(caller, service,
                        resolvedType, callingPid, callingUid,
                        requireForeground, callingPackage, userId);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return res;
        }
    }
```





> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "startService: " + service
            + " type=" + resolvedType + " args=" + service.getExtras());

    final boolean callerFg;
    if (caller != null) {
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + callingPid
                    + ") when starting service " + service);
        }
        callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
    } else {
        callerFg = true;
    }

    ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false);
    if (res == null) {
        return null;
    }
    if (res.record == null) {
        return new ComponentName("!", res.permission != null
                ? res.permission : "private to package");
    }

    ServiceRecord r = res.record;

    if (!mAm.mUserController.exists(r.userId)) {
        Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
        return null;
    }

    // If this isn't a direct-to-foreground start, check our ability to kick off an
    // arbitrary service
    if (!r.startRequested && !fgRequired) {
        // Before going further -- if this app is not allowed to start services in the
        // background, then at this point we aren't going to let it period.
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            Slog.w(TAG, "Background start not allowed: service "
                    + service + " to " + r.name.flattenToShortString()
                    + " from pid=" + callingPid + " uid=" + callingUid
                    + " pkg=" + callingPackage);
            if (allowed == ActivityManager.APP_START_MODE_DELAYED) {
                // In this case we are silently disabling the app, to disrupt as
                // little as possible existing apps.
                return null;
            }
            // This app knows it is in the new model where this operation is not
            // allowed, so tell it what has happened.
            UidRecord uidRec = mAm.mActiveUids.get(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
        }
    }

    NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
            callingUid, r.packageName, service, service.getFlags(), null, r.userId);

    // If permissions need a review before any of the app components can run,
    // we do not start the service and launch a review activity if the calling app
    // is in the foreground passing it a pending intent to start the service when
    // review is completed.
    if (mAm.mPermissionReviewRequired) {
        if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                callingUid, service, callerFg, userId)) {
            return null;
        }
    }

    if (unscheduleServiceRestartLocked(r, callingUid, false)) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
    }
    r.lastActivity = SystemClock.uptimeMillis();
    r.startRequested = true;
    r.delayedStop = false;
    r.fgRequired = fgRequired;
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants, callingUid));

    final ServiceMap smap = getServiceMapLocked(r.userId);
    boolean addToStarting = false;
    if (!callerFg && !fgRequired && r.app == null
            && mAm.mUserController.hasStartedUserState(r.userId)) {
        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
        if (proc == null || proc.curProcState > ActivityManager.PROCESS_STATE_RECEIVER) {
            // If this is not coming from a foreground caller, then we may want
            // to delay the start if there are already other background services
            // that are starting.  This is to avoid process start spam when lots
            // of applications are all handling things like connectivity broadcasts.
            // We only do this for cached processes, because otherwise an application
            // can have assumptions about calling startService() for a service to run
            // in its own process, and for that process to not be killed before the
            // service is started.  This is especially the case for receivers, which
            // may start a service in onReceive() to do some additional work and have
            // initialized some global state as part of that.
            if (DEBUG_DELAYED_SERVICE) Slog.v(TAG_SERVICE, "Potential start delay of "
                    + r + " in " + proc);
            if (r.delayed) {
                // This service is already scheduled for a delayed start; just leave
                // it still waiting.
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Continuing to delay: " + r);
                return r.name;
            }
            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                // Something else is starting, delay!
                Slog.i(TAG_SERVICE, "Delaying start of: " + r);
                smap.mDelayedStartList.add(r);
                r.delayed = true;
                return r.name;
            }
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Not delaying: " + r);
            addToStarting = true;
        } else if (proc.curProcState >= ActivityManager.PROCESS_STATE_SERVICE) {
            // We slightly loosen when we will enqueue this new service as a background
            // starting service we are waiting for, to also include processes that are
            // currently running other services or receivers.
            addToStarting = true;
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Not delaying, but counting as bg: " + r);
        } else if (DEBUG_DELAYED_STARTS) {
            StringBuilder sb = new StringBuilder(128);
            sb.append("Not potential delay (state=").append(proc.curProcState)
                    .append(' ').append(proc.adjType);
            String reason = proc.makeAdjReason();
            if (reason != null) {
                sb.append(' ');
                sb.append(reason);
            }
            sb.append("): ");
            sb.append(r.toString());
            Slog.v(TAG_SERVICE, sb.toString());
        }
    } else if (DEBUG_DELAYED_STARTS) {
        if (callerFg || fgRequired) {
            Slog.v(TAG_SERVICE, "Not potential delay (callerFg=" + callerFg + " uid="
                    + callingUid + " pid=" + callingPid + " fgRequired=" + fgRequired + "): " + r);
        } else if (r.app != null) {
            Slog.v(TAG_SERVICE, "Not potential delay (cur app=" + r.app + "): " + r);
        } else {
            Slog.v(TAG_SERVICE,
                    "Not potential delay (user " + r.userId + " not started): " + r);
        }
    }

    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    return cmp;
}

ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    ServiceState stracker = r.getTracker();
    if (stracker != null) {
        stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
    }
    r.callStart = false;
    synchronized (r.stats.getBatteryStats()) {
        r.stats.startRunningLocked();
    }
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    if (r.startRequested && addToStarting) {
        boolean first = smap.mStartingBackground.size() == 0;
        smap.mStartingBackground.add(r);
        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
        if (DEBUG_DELAYED_SERVICE) {
            RuntimeException here = new RuntimeException("here");
            here.fillInStackTrace();
            Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r, here);
        } else if (DEBUG_DELAYED_STARTS) {
            Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r);
        }
        if (first) {
            smap.rescheduleDelayedStartsLocked();
        }
    } else if (callerFg || r.fgRequired) {
        smap.ensureNotStartingBackgroundLocked(r);
    }

    return r.name;
}


private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
    //Slog.i(TAG, "Bring up service:");
    //r.dump("  ");

    if (r.app != null && r.app.thread != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }

    if (!whileRestarting && mRestartingServices.contains(r)) {
        // If waiting for a restart, then do nothing.
        return null;
    }

    if (DEBUG_SERVICE) {
        Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent + " fg=" + r.fgRequired);
    }

    // We are now bringing the service up, so no longer in the
    // restarting state.
    if (mRestartingServices.remove(r)) {
        clearRestartingIfNeededLocked(r);
    }

    // Make sure this service is no longer considered delayed, we are starting it now.
    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    // Make sure that the user who owns this service is started.  If not,
    // we don't want to allow it to run.
    if (!mAm.mUserController.hasStartedUserState(r.userId)) {
        String msg = "Unable to launch app "
                + r.appInfo.packageName + "/"
                + r.appInfo.uid + " for service "
                + r.intent.getIntent() + ": user " + r.userId + " is stopped";
        Slog.w(TAG, msg);
        bringDownServiceLocked(r);
        return msg;
    }

    // Service is now being launched, its package can't be stopped.
    try {
        AppGlobals.getPackageManager().setPackageStoppedState(
                r.packageName, false, r.userId);
    } catch (RemoteException e) {
    } catch (IllegalArgumentException e) {
        Slog.w(TAG, "Failed trying to unstop package "
                + r.packageName + ": " + e);
    }

    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    String hostingType = "service";
    ProcessRecord app;

    if (!isolated) {
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
        if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                    + " app=" + app);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                realStartServiceLocked(r, app, execInFg);
                return null;
            } catch (TransactionTooLargeException e) {
                throw e;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting service " + r.shortName, e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
    } else {
        // If this service runs in an isolated process, then each time
        // we call startProcessLocked() we will get a new isolated
        // process, starting another process if we are currently waiting
        // for a previous process to come up.  To deal with this, we store
        // in the service any current isolated process it is running in or
        // waiting to have come up.
        app = r.isolatedProc;
        if (WebViewZygote.isMultiprocessEnabled()
                && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
            hostingType = "webview_service";
        }
    }

    // Not running -- get it started, and enqueue this service record
    // to be executed when the app comes up.
    if (app == null && !permissionsReviewRequired) {
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                hostingType, r.name, false, isolated, false)) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }
        if (isolated) {
            r.isolatedProc = app;
        }
    }

    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }

    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (in bring up): " + r);
            stopServiceLocked(r);
        }
    }

    return null;
}

private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    if (app.thread == null) {
        throw new RemoteException();
    }
    if (DEBUG_MU)
        Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                + ", ProcessRecord.uid = " + app.uid);
    r.app = app;
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

    final boolean newService = app.services.add(r);
    bumpServiceExecutingLocked(r, execInFg, "create");
    mAm.updateLruProcessLocked(app, false, null);
    updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
    mAm.updateOomAdjLocked();

    boolean created = false;
    try {
        if (LOG_SERVICE_START_STOP) {
            String nameTerm;
            int lastPeriod = r.shortName.lastIndexOf('.');
            nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
            EventLogTags.writeAmCreateService(
                    r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
        }
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startLaunchedLocked();
        }
        mAm.notifyPackageUse(r.serviceInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        Slog.w(TAG, "Application dead when creating service " + r);
        mAm.appDiedLocked(app);
        throw e;
    } finally {
        if (!created) {
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            // Cleanup.
            if (newService) {
                app.services.remove(r);
                r.app = null;
            }

            // Retry.
            if (!inDestroying) {
                scheduleServiceRestartLocked(r, false);
            }
        }
    }

    if (r.whitelistManager) {
        app.whitelistManager = true;
    }

    requestServiceBindingsLocked(r, execInFg);

    updateServiceClientActivitiesLocked(app, null, true);

    // If the service is in the started state, and there are no
    // pending arguments, then fake up one so its onStartCommand() will
    // be called.
    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                null, null, 0));
    }

    sendServiceArgsLocked(r, execInFg, true);

    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (from start): " + r);
            stopServiceLocked(r);
        }
    }
}
```

