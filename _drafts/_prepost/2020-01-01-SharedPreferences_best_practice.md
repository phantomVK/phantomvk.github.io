---
layout:     post
title:      "SharedPreferences最佳实践"
date:       2020-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

[Android源码系列(12) -- SharedPreferences](/2018/09/14/SharedPreferences/)

[Android源码系列(17) -- QueuedWork](/2018/11/05/QueuedWork/)



Android 8.1.0 ActivityThread

```java
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            if (data.args != null) {
                data.args.setExtrasClassLoader(s.getClassLoader());
                data.args.prepareToEnterProcess();
            }
            int res;
            if (!data.taskRemoved) {
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }

            QueuedWork.waitToFinish();

            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
            ensureJitEnabled();
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to start service " + s
                        + " with " + data.args + ": " + e.toString(), e);
            }
        }
    }
}
```



```java
private void handleStopService(IBinder token) {
    Service s = mServices.remove(token);
    if (s != null) {
        try {
            if (localLOGV) Slog.v(TAG, "Destroying service " + s);
            s.onDestroy();
            s.detachAndCleanUp();
            Context context = s.getBaseContext();
            if (context instanceof ContextImpl) {
                final String who = s.getClassName();
                ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
            }

            QueuedWork.waitToFinish();

            try {
                ActivityManager.getService().serviceDoneExecuting(
                        token, SERVICE_DONE_EXECUTING_STOP, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to stop service " + s
                        + ": " + e.toString(), e);
            }
            Slog.i(TAG, "handleStopService: exception for " + token, e);
        }
    } else {
        Slog.i(TAG, "handleStopService: token=" + token + " not found.");
    }
    //Slog.i(TAG, "Running services: " + mServices);
}
```

```java
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport, int seq) {
    ActivityClientRecord r = mActivities.get(token);
    if (DEBUG_ORDER) Slog.d(TAG, "handlePauseActivity " + r + ", seq: " + seq);
    if (!checkAndUpdateLifecycleSeq(seq, r, "pauseActivity")) {
        return;
    }
    if (r != null) {
        //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");

        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }

        // Tell the activity manager we have paused.
        if (!dontReport) {
            try {
                ActivityManager.getService().activityPaused(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```

```java
private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {
    ActivityClientRecord r = mActivities.get(token);
    if (!checkAndUpdateLifecycleSeq(seq, r, "stopActivity")) {
        return;
    }
    r.activity.mConfigChangeFlags |= configChanges;

    StopInfo info = new StopInfo();
    performStopActivityInner(r, info, show, true, "handleStopActivity");

    if (localLOGV) Slog.v(
        TAG, "Finishing stop of " + r + ": show=" + show
        + " win=" + r.window);

    updateVisibility(r, show);

    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    // Schedule the call to tell the activity manager we have
    // stopped.  We don't do this immediately, because we want to
    // have a chance for any other pending work (in particular memory
    // trim requests) to complete before you tell the activity
    // manager to proceed and allow us to go fully into the background.
    info.activity = r;
    info.state = r.state;
    info.persistentState = r.persistentState;
    mH.post(info);
    mSomeActivitiesChanged = true;
}
```



```java
// TODO: This method should be changed to use {@link #performStopActivityInner} to perform to
// stop operation on the activity to reduce code duplication and the chance of fixing a bug in
// one place and missing the other.
private void handleSleeping(IBinder token, boolean sleeping) {
    ActivityClientRecord r = mActivities.get(token);

    if (r == null) {
        Log.w(TAG, "handleSleeping: no activity for token " + token);
        return;
    }

    if (sleeping) {
        if (!r.stopped && !r.isPreHoneycomb()) {
            if (!r.activity.mFinished && r.state == null) {
                callCallActivityOnSaveInstanceState(r);
            }

            try {
                // Now we are idle.
                r.activity.performStop(false /*preserveWindow*/);
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                            "Unable to stop activity "
                            + r.intent.getComponent().toShortString()
                            + ": " + e.toString(), e);
                }
            }
            r.stopped = true;
            EventLog.writeEvent(LOG_AM_ON_STOP_CALLED, UserHandle.myUserId(),
                    r.activity.getComponentName().getClassName(), "sleeping");
        }

        // Make sure any pending writes are now committed.
        if (!r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }

        // Tell activity manager we slept.
        try {
            ActivityManager.getService().activitySlept(r.token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    } else {
        if (r.stopped && r.activity.mVisibleFromServer) {
            r.activity.performRestart();
            r.stopped = false;
        }
    }
}
```

