---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

Android 29



View

```java
/**
 * Call this when something has changed which has invalidated the
 * layout of this view. This will schedule a layout pass of the view
 * tree. This should not be called while the view hierarchy is currently in a layout
 * pass ({@link #isInLayout()}. If layout is happening, the request may be honored at the
 * end of the current layout pass (and then layout will run again) or after the current
 * frame is drawn and the next layout occurs.
 *
 * <p>Subclasses which override this method should call the superclass method to
 * handle possible request-during-layout errors correctly.</p>
 */
@CallSuper
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        // Only trigger request-during-layout logic if this is the view requesting it,
        // not the views in its parent hierarchy
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }

    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

ViewRootImpl

```java
/**
 * Called by {@link android.view.View#requestLayout()} if the view hierarchy is currently
 * undergoing a layout pass. requestLayout() should not generally be called during layout,
 * unless the container hierarchy knows what it is doing (i.e., it is fine as long as
 * all children in that container hierarchy are measured and laid out at the end of the layout
 * pass for that container). If requestLayout() is called anyway, we handle it correctly
 * by registering all requesters during a frame as it proceeds. At the end of the frame,
 * we check all of those views to see if any still have pending layout requests, which
 * indicates that they were not correctly handled by their container hierarchy. If that is
 * the case, we clear all such flags in the tree, to remove the buggy flag state that leads
 * to blank containers, and force a second request/measure/layout pass in this frame. If
 * more requestLayout() calls are received during that second layout pass, we post those
 * requests to the next frame to avoid possible infinite loops.
 *
 * <p>The return value from this method indicates whether the request should proceed
 * (if it is a request during the first layout pass) or should be skipped and posted to the
 * next frame (if it is a request during the second layout pass).</p>
 *
 * @param view the view that requested the layout.
 *
 * @return true if request should proceed, false otherwise.
 */
boolean requestLayoutDuringLayout(final View view) {
    if (view.mParent == null || view.mAttachInfo == null) {
        // Would not normally trigger another layout, so just let it pass through as usual
        return true;
    }
    if (!mLayoutRequesters.contains(view)) {
        mLayoutRequesters.add(view);
    }
    if (!mHandlingLayoutInLayoutRequest) {
        // Let the request proceed normally; it will be processed in a second layout pass
        // if necessary
        return true;
    } else {
        // Don't let the request proceed during the second layout pass.
        // It will post to the next frame instead.
        return false;
    }
}
```

ViewRootImpl

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

```java
@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

__TraversalRunnable__ 回调 __doTraversal()__

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

