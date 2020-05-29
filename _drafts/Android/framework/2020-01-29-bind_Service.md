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



## 发起绑定

> frameworks/base/core/java/android/content/ContextWrapper.java

```java
    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }
```

>frameworks/base/core/java/android/app/ContextImpl.java

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
            Process.myUserHandle());
}

private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
        handler, UserHandle user) {
    // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
    IServiceConnection sd;
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    } else {
        throw new RuntimeException("Not supported in system context");
    }
    validateServiceIntent(service);
    try {
        IBinder token = getActivityToken();
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        service.prepareToLeaveProcess(this);
        int res = ActivityManager.getService().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

## 调用AMS

> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
    public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {
        enforceNotIsolatedCaller("bindService");

        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);
        }
    }
```

> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection, int flags,
        String callingPackage, final int userId) throws TransactionTooLargeException {
    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "bindService: " + service
            + " type=" + resolvedType + " conn=" + connection.asBinder()
            + " flags=0x" + Integer.toHexString(flags));
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
    if (callerApp == null) {
        throw new SecurityException(
                "Unable to find app for caller " + caller
                + " (pid=" + Binder.getCallingPid()
                + ") when binding service " + service);
    }

    ActivityRecord activity = null;
    if (token != null) {
        activity = ActivityRecord.isInStackLocked(token);
        if (activity == null) {
            Slog.w(TAG, "Binding with unknown activity: " + token);
            return 0;
        }
    }

    int clientLabel = 0;
    PendingIntent clientIntent = null;
    final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

    if (isCallerSystem) {
        // Hacky kind of thing -- allow system stuff to tell us
        // what they are, so we can report this elsewhere for
        // others to know why certain services are running.
        service.setDefusable(true);
        clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);
        if (clientIntent != null) {
            clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
            if (clientLabel != 0) {
                // There are no useful extras in the intent, trash them.
                // System code calling with this stuff just needs to know
                // this will happen.
                service = service.cloneFilter();
            }
        }
    }

    if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
        mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                "BIND_TREAT_LIKE_ACTIVITY");
    }

    if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {
        throw new SecurityException(
                "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
    }

    final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
    final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;

    ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                Binder.getCallingUid(), userId, true, callerFg, isBindExternal);
    if (res == null) {
        return 0;
    }
    if (res.record == null) {
        return -1;
    }
    ServiceRecord s = res.record;

    boolean permissionsReviewRequired = false;

    // If permissions need a review before any of the app components can run,
    // we schedule binding to the service but do not start its process, then
    // we launch a review activity to which is passed a callback to invoke
    // when done to start the bound service's process to completing the binding.
    if (mAm.mPermissionReviewRequired) {
        if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                s.packageName, s.userId)) {

            permissionsReviewRequired = true;

            // Show a permission review UI only for binding from a foreground app
            if (!callerFg) {
                Slog.w(TAG, "u" + s.userId + " Binding to a service in package"
                        + s.packageName + " requires a permissions review");
                return 0;
            }

            final ServiceRecord serviceRecord = s;
            final Intent serviceIntent = service;

            RemoteCallback callback = new RemoteCallback(
                    new RemoteCallback.OnResultListener() {
                @Override
                public void onResult(Bundle result) {
                    synchronized(mAm) {
                        final long identity = Binder.clearCallingIdentity();
                        try {
                            if (!mPendingServices.contains(serviceRecord)) {
                                return;
                            }
                            // If there is still a pending record, then the service
                            // binding request is still valid, so hook them up. We
                            // proceed only if the caller cleared the review requirement
                            // otherwise we unbind because the user didn't approve.
                            if (!mAm.getPackageManagerInternalLocked()
                                    .isPermissionsReviewRequired(
                                            serviceRecord.packageName,
                                            serviceRecord.userId)) {
                                try {
                                    bringUpServiceLocked(serviceRecord,
                                            serviceIntent.getFlags(),
                                            callerFg, false, false);
                                } catch (RemoteException e) {
                                    /* ignore - local call */
                                }
                            } else {
                                unbindServiceLocked(connection);
                            }
                        } finally {
                            Binder.restoreCallingIdentity(identity);
                        }
                    }
                }
            });

            final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                    | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
            intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
            intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);

            if (DEBUG_PERMISSIONS_REVIEW) {
                Slog.i(TAG, "u" + s.userId + " Launching permission review for package "
                        + s.packageName);
            }

            mAm.mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                }
            });
        }
    }

    final long origId = Binder.clearCallingIdentity();

    try {
        if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: "
                    + s);
        }

        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            if (!s.hasAutoCreateConnections()) {
                // This is the first binding, let the tracker know.
                ServiceState stracker = s.getTracker();
                if (stracker != null) {
                    stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                            s.lastActivity);
                }
            }
        }

        mAm.startAssociationLocked(callerApp.uid, callerApp.processName, callerApp.curProcState,
                s.appInfo.uid, s.name, s.processName);
        // Once the apps have become associated, if one of them is caller is ephemeral
        // the target app should now be able to see the calling app
        mAm.grantEphemeralAccessLocked(callerApp.userId, service,
                s.appInfo.uid, UserHandle.getAppId(callerApp.uid));

        AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
        ConnectionRecord c = new ConnectionRecord(b, activity,
                connection, flags, clientLabel, clientIntent);

        IBinder binder = connection.asBinder();
        ArrayList<ConnectionRecord> clist = s.connections.get(binder);
        if (clist == null) {
            clist = new ArrayList<ConnectionRecord>();
            s.connections.put(binder, clist);
        }
        clist.add(c);
        b.connections.add(c);
        if (activity != null) {
            if (activity.connections == null) {
                activity.connections = new HashSet<ConnectionRecord>();
            }
            activity.connections.add(c);
        }
        b.client.connections.add(c);
        if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
            b.client.hasAboveClient = true;
        }
        if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
            s.whitelistManager = true;
        }
        if (s.app != null) {
            updateServiceClientActivitiesLocked(s.app, c, true);
        }
        clist = mServiceConnections.get(binder);
        if (clist == null) {
            clist = new ArrayList<ConnectionRecord>();
            mServiceConnections.put(binder, clist);
        }
        clist.add(c);

        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                    permissionsReviewRequired) != null) {
                return 0;
            }
        }

        if (s.app != null) {
            if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                s.app.treatLikeActivity = true;
            }
            if (s.whitelistManager) {
                s.app.whitelistManager = true;
            }
            // This could have made the service more important.
            mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                    || s.app.treatLikeActivity, b.client);
            mAm.updateOomAdjLocked(s.app, true);
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bind " + s + " with " + b
                + ": received=" + b.intent.received
                + " apps=" + b.intent.apps.size()
                + " doRebind=" + b.intent.doRebind);

        if (s.app != null && b.intent.received) {
            // Service is already running, so we can immediately
            // publish the connection.
            try {
                c.conn.connected(s.name, b.intent.binder, false);
            } catch (Exception e) {
                Slog.w(TAG, "Failure sending service " + s.shortName
                        + " to connection " + c.conn.asBinder()
                        + " (in " + c.binding.client.processName + ")", e);
            }

            // If this is the first app connected back to this binding,
            // and the service had previously asked to be told when
            // rebound, then do so.
            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                requestServiceBindingLocked(s, b.intent, callerFg, true);
            }
        } else if (!b.intent.requested) {
            requestServiceBindingLocked(s, b.intent, callerFg, false);
        }

        getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    return 1;
}
```

