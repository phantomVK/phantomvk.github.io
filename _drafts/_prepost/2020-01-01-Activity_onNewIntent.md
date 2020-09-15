---
layout:     post
title:      "Android onNewIntent()"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---



[Android Developers Activity.onNewIntent](https://developer.android.com/reference/android/app/Activity#onNewIntent(android.content.Intent))



### Activity

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient {

    protected void onNewIntent(Intent intent) {
    }
            
    final void performNewIntent(Intent intent) {
        mCanEnterPictureInPicture = true;
        onNewIntent(intent);
    }
}
```

### Instrumentation

```java
/**
 * Perform calling of an activity's {@link Activity#onNewIntent}
 * method.  The default implementation simply calls through to that method.
 * 
 * @param activity The activity receiving a new Intent.
 * @param intent The new intent being received.
 */
public void callActivityOnNewIntent(Activity activity, Intent intent) {
    activity.performNewIntent(intent);
}
```

```java
public void callActivityOnNewIntent(Activity activity, ReferrerIntent intent) {
    final String oldReferrer = activity.mReferrer;
    try {
        if (intent != null) {
            activity.mReferrer = intent.mReferrer;
        }
        callActivityOnNewIntent(activity, intent != null ? new Intent(intent) : null);
    } finally {
        activity.mReferrer = oldReferrer;
    }
}
```

### ActivityThread

```java
public final class ActivityThread
```



```java
private void deliverNewIntents(ActivityClientRecord r, List<ReferrerIntent> intents) {
    final int N = intents.size();
    for (int i=0; i<N; i++) {
        ReferrerIntent intent = intents.get(i);
        intent.setExtrasClassLoader(r.activity.getClassLoader());
        intent.prepareToEnterProcess();
        r.activity.mFragments.noteStateNotSaved();
        mInstrumentation.callActivityOnNewIntent(r.activity, intent);
    }
}
```

```java
void performNewIntents(IBinder token, List<ReferrerIntent> intents, boolean andPause) {
    final ActivityClientRecord r = mActivities.get(token);
    if (r == null) {
        return;
    }

    final boolean resumed = !r.paused;
    if (resumed) {
        r.activity.mTemporaryPause = true;
        mInstrumentation.callActivityOnPause(r.activity);
    }
    checkAndBlockForNetworkAccess();
    deliverNewIntents(r, intents);
    if (resumed) {
        r.activity.performResume();
        r.activity.mTemporaryPause = false;
    }

    if (r.paused && andPause) {
        // In this case the activity was in the paused state when we delivered the intent,
        // to guarantee onResume gets called after onNewIntent we temporarily resume the
        // activity and pause again as the caller wanted.
        performResumeActivity(token, false, "performNewIntents");
        performPauseActivityIfNeeded(r, "performNewIntents");
    }
}
```

```java
public final ActivityClientRecord performResumeActivity(IBinder token,
        boolean clearHide, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (localLOGV) Slog.v(TAG, "Performing resume of " + r
            + " finished=" + r.activity.mFinished);
    if (r != null && !r.activity.mFinished) {
        if (clearHide) {
            r.hideForNow = false;
            r.activity.mStartedActivity = false;
        }
        try {
            r.activity.onStateNotSaved();
            r.activity.mFragments.noteStateNotSaved();
            checkAndBlockForNetworkAccess();
            if (r.pendingIntents != null) {
                deliverNewIntents(r, r.pendingIntents);
                r.pendingIntents = null;
            }
            if (r.pendingResults != null) {
                deliverResults(r, r.pendingResults);
                r.pendingResults = null;
            }
            r.activity.performResume();

            synchronized (mResourcesManager) {
                // If there is a pending local relaunch that was requested when the activity was
                // paused, it will put the activity into paused state when it finally happens.
                // Since the activity resumed before being relaunched, we don't want that to
                // happen, so we need to clear the request to relaunch paused.
                for (int i = mRelaunchingActivities.size() - 1; i >= 0; i--) {
                    final ActivityClientRecord relaunching = mRelaunchingActivities.get(i);
                    if (relaunching.token == r.token
                            && relaunching.onlyLocalRequest && relaunching.startsNotResumed) {
                        relaunching.startsNotResumed = false;
                    }
                }
            }

            EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED, UserHandle.myUserId(),
                    r.activity.getComponentName().getClassName(), reason);

            r.paused = false;
            r.stopped = false;
            r.state = null;
            r.persistentState = null;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                    "Unable to resume activity "
                    + r.intent.getComponent().toShortString()
                    + ": " + e.toString(), e);
            }
        }
    }
    return r;
}
```



```java
private void handleNewIntent(NewIntentData data) {
    performNewIntents(data.token, data.intents, data.andPause);
}
```



```java
private class H extends Handler {
    public static final int NEW_INTENT              = 112;

    public void handleMessage(Message msg) {
            case NEW_INTENT:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityNewIntent");
                handleNewIntent((NewIntentData)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
    }
}
```



```java
private class ApplicationThread extends IApplicationThread.Stub {
    public final void scheduleNewIntent(
            List<ReferrerIntent> intents, IBinder token, boolean andPause) {
        NewIntentData data = new NewIntentData();
        data.intents = intents;
        data.token = token;
        data.andPause = andPause;

        sendMessage(H.NEW_INTENT, data);
    }
}
```

### ActivityRecord

```java
final class ActivityRecord extends ConfigurationContainer implements AppWindowContainerListener {
    
    /**
     * Deliver a new Intent to an existing activity, so that its onNewIntent()
     * method will be called at the proper time.
     */
    final void deliverNewIntentLocked(int callingUid, Intent intent, String referrer) {
        // The activity now gets access to the data associated with this Intent.
        service.grantUriPermissionFromIntentLocked(callingUid, packageName,
                intent, getUriPermissionsLocked(), userId);
        final ReferrerIntent rintent = new ReferrerIntent(intent, referrer);
        boolean unsent = true;
        final ActivityStack stack = getStack();
        final boolean isTopActivityWhileSleeping = isTopRunningActivity()
                && (stack != null ? stack.shouldSleepActivities() : service.isSleepingLocked());

        // We want to immediately deliver the intent to the activity if:
        // - It is currently resumed or paused. i.e. it is currently visible to the user and we want
        //   the user to see the visual effects caused by the intent delivery now.
        // - The device is sleeping and it is the top activity behind the lock screen (b/6700897).
        if ((state == RESUMED || state == PAUSED
                || isTopActivityWhileSleeping) && app != null && app.thread != null) {
            try {
                ArrayList<ReferrerIntent> ar = new ArrayList<>(1);
                ar.add(rintent);
                app.thread.scheduleNewIntent(
                        ar, appToken, state == PAUSED /* andPause */);
                unsent = false;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            } catch (NullPointerException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            }
        }
        if (unsent) {
            addNewIntentLocked(rintent);
        }
    }
}
```

### ActivityStarter

```java
private int setTaskFromInTask() {
    // The caller is asking that the new activity be started in an explicit
    // task it has provided to us.
    if (mSupervisor.isLockTaskModeViolation(mInTask)) {
        Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
        return START_RETURN_LOCK_TASK_MODE_VIOLATION;
    }

    mTargetStack = mInTask.getStack();

    // Check whether we should actually launch the new activity in to the task,
    // or just reuse the current activity on top.
    ActivityRecord top = mInTask.getTopActivity();
    if (top != null && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId) {
        if ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || mLaunchSingleTop || mLaunchSingleTask) {
            mTargetStack.moveTaskToFrontLocked(mInTask, mNoAnimation, mOptions,
                    mStartActivity.appTimeTracker, "inTaskToFront");
            if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
                // We don't need to start a new activity, and the client said not to do
                // anything if that is the case, so this is it!
                return START_RETURN_INTENT_TO_CALLER;
            }
            deliverNewIntent(top);
            return START_DELIVERED_TO_TOP;
        }
    }

    if (!mAddingToTask) {
        mTargetStack.moveTaskToFrontLocked(mInTask, mNoAnimation, mOptions,
                mStartActivity.appTimeTracker, "inTaskToFront");
        // We don't actually want to have this activity added to the task, so just
        // stop here but still tell the caller that we consumed the intent.
        ActivityOptions.abort(mOptions);
        return START_TASK_TO_FRONT;
    }

    if (mLaunchBounds != null) {
        mInTask.updateOverrideConfiguration(mLaunchBounds);
        int stackId = mInTask.getLaunchStackId();
        if (stackId != mInTask.getStackId()) {
            mInTask.reparent(stackId, ON_TOP, REPARENT_KEEP_STACK_AT_FRONT, !ANIMATE,
                    DEFER_RESUME, "inTaskToFront");
            stackId = mInTask.getStackId();
            mTargetStack = mInTask.getStack();
        }
        if (StackId.resizeStackWithLaunchBounds(stackId)) {
            mService.resizeStack(stackId, mLaunchBounds, true, !PRESERVE_WINDOWS, ANIMATE, -1);
        }
    }

    mTargetStack.moveTaskToFrontLocked(
            mInTask, mNoAnimation, mOptions, mStartActivity.appTimeTracker, "inTaskToFront");

    addOrReparentStartingActivity(mInTask, "setTaskFromInTask");
    if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + mStartActivity
            + " in explicit task " + mStartActivity.getTask());

    return START_SUCCESS;
}
```

```java
private void deliverNewIntent(ActivityRecord activity) {
    if (mIntentDelivered) {
        return;
    }

    ActivityStack.logStartActivity(AM_NEW_INTENT, activity, activity.getTask());
    activity.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
            mStartActivity.launchedFromPackage);
    mIntentDelivered = true;
}
```





第259行、第261行

```java
private int setTaskFromSourceRecord() {
    if (mSupervisor.isLockTaskModeViolation(mSourceRecord.getTask())) {
        Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
        return START_RETURN_LOCK_TASK_MODE_VIOLATION;
    }

    final TaskRecord sourceTask = mSourceRecord.getTask();
    final ActivityStack sourceStack = mSourceRecord.getStack();
    // We only want to allow changing stack in two cases:
    // 1. If the target task is not the top one. Otherwise we would move the launching task to
    //    the other side, rather than show two side by side.
    // 2. If activity is not allowed on target display.
    final int targetDisplayId = mTargetStack != null ? mTargetStack.mDisplayId
            : sourceStack.mDisplayId;
    final boolean moveStackAllowed = sourceStack.topTask() != sourceTask
            || !mStartActivity.canBeLaunchedOnDisplay(targetDisplayId);
    if (moveStackAllowed) {
        mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, mStartActivity.getTask(),
                mOptions);
        // If target stack is not found now - we can't just rely on the source stack, as it may
        // be not suitable. Let's check other displays.
        if (mTargetStack == null && targetDisplayId != sourceStack.mDisplayId) {
            // Can't use target display, lets find a stack on the source display.
            mTargetStack = mService.mStackSupervisor.getValidLaunchStackOnDisplay(
                    sourceStack.mDisplayId, mStartActivity);
        }
        if (mTargetStack == null) {
            // There are no suitable stacks on the target and source display(s). Look on all
            // displays.
            mTargetStack = mService.mStackSupervisor.getNextValidLaunchStackLocked(
                    mStartActivity, -1 /* currentFocus */);
        }
    }

    if (mTargetStack == null) {
        mTargetStack = sourceStack;
    } else if (mTargetStack != sourceStack) {
        sourceTask.reparent(mTargetStack.mStackId, ON_TOP, REPARENT_MOVE_STACK_TO_FRONT,
                !ANIMATE, DEFER_RESUME, "launchToSide");
    }

    final TaskRecord topTask = mTargetStack.topTask();
    if (topTask != sourceTask && !mAvoidMoveToFront) {
        mTargetStack.moveTaskToFrontLocked(sourceTask, mNoAnimation, mOptions,
                mStartActivity.appTimeTracker, "sourceTaskToFront");
    } else if (mDoResume) {
        mTargetStack.moveToFront("sourceStackToFront");
    }

    if (!mAddingToTask && (mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0) {
        // In this case, we are adding the activity to an existing task, but the caller has
        // asked to clear that task if the activity is already running.
        ActivityRecord top = sourceTask.performClearTaskLocked(mStartActivity, mLaunchFlags);
        mKeepCurTransition = true;
        if (top != null) {
            ActivityStack.logStartActivity(AM_NEW_INTENT, mStartActivity, top.getTask());
            deliverNewIntent(top);
            // For paranoia, make sure we have correctly resumed the top activity.
            mTargetStack.mLastPausedActivity = null;
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            ActivityOptions.abort(mOptions);
            return START_DELIVERED_TO_TOP;
        }
    } else if (!mAddingToTask && (mLaunchFlags & FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
        // In this case, we are launching an activity in our own task that may already be
        // running somewhere in the history, and we want to shuffle it to the front of the
        // stack if so.
        final ActivityRecord top = sourceTask.findActivityInHistoryLocked(mStartActivity);
        if (top != null) {
            final TaskRecord task = top.getTask();
            task.moveActivityToFrontLocked(top);
            top.updateOptionsLocked(mOptions);
            ActivityStack.logStartActivity(AM_NEW_INTENT, mStartActivity, task);
            deliverNewIntent(top);
            mTargetStack.mLastPausedActivity = null;
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return START_DELIVERED_TO_TOP;
        }
    }

    // An existing activity is starting this new activity, so we want to keep the new one in
    // the same task as the one that is starting it.
    addOrReparentStartingActivity(sourceTask, "setTaskFromSourceRecord");
    if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + mStartActivity
            + " in existing task " + mStartActivity.getTask() + " from source " + mSourceRecord);
    return START_SUCCESS;
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
                deliverNewIntent(top);
            }
        }

        sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivity);

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

        deliverNewIntent(top);

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

    sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);

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



```java
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
                deliverNewIntent(top);
            }
        }

        sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivity);

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

        deliverNewIntent(top);

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

    sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);

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



```java
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
```



```java
void postStartActivityProcessing(
        ActivityRecord r, int result, int prevFocusedStackId, ActivityRecord sourceRecord,
        ActivityStack targetStack) {

    if (ActivityManager.isStartResultFatalError(result)) {
        return;
    }

    // We're waiting for an activity launch to finish, but that activity simply
    // brought another activity to front. Let startActivityMayWait() know about
    // this, so it waits for the new activity to become visible instead.
    if (result == START_TASK_TO_FRONT && !mSupervisor.mWaitingActivityLaunched.isEmpty()) {
        mSupervisor.reportTaskToFrontNoLaunch(mStartActivity);
    }

    int startedActivityStackId = INVALID_STACK_ID;
    final ActivityStack currentStack = r.getStack();
    if (currentStack != null) {
        startedActivityStackId = currentStack.mStackId;
    } else if (mTargetStack != null) {
        startedActivityStackId = targetStack.mStackId;
    }

    if (startedActivityStackId == DOCKED_STACK_ID) {
        final ActivityStack homeStack = mSupervisor.getStack(HOME_STACK_ID);
        final boolean homeStackVisible = homeStack != null && homeStack.isVisible();
        if (homeStackVisible) {
            // We launch an activity while being in home stack, which means either launcher or
            // recents into docked stack. We don't want the launched activity to be alone in a
            // docked stack, so we want to immediately launch recents too.
            if (DEBUG_RECENTS) Slog.d(TAG, "Scheduling recents launch.");
            mWindowManager.showRecentApps(true /* fromHome */);
        }
        return;
    }

    boolean clearedTask = (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK) && (mReuseTask != null);
    if (startedActivityStackId == PINNED_STACK_ID && (result == START_TASK_TO_FRONT
            || result == START_DELIVERED_TO_TOP || clearedTask)) {
        // The activity was already running in the pinned stack so it wasn't started, but either
        // brought to the front or the new intent was delivered to it since it was already in
        // front. Notify anyone interested in this piece of information.
        mService.mTaskChangeNotificationController.notifyPinnedActivityRestartAttempt(
                clearedTask);
        return;
    }
}
```



### ActivityStackSuprisor

```java
void reportTaskToFrontNoLaunch(ActivityRecord r) {
    boolean changed = false;
    for (int i = mWaitingActivityLaunched.size() - 1; i >= 0; i--) {
        WaitResult w = mWaitingActivityLaunched.remove(i);
        if (w.who == null) {
            changed = true;
            // Set result to START_TASK_TO_FRONT so that startActivityMayWait() knows that
            // the starting activity ends up moving another activity to front, and it should
            // wait for this new activity to become visible instead.
            // Do not modify other fields.
            w.result = START_TASK_TO_FRONT;
        }
    }
    if (changed) {
        mService.notifyAll();
    }
}
```

