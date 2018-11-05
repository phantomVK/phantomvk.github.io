---
layout:     post
title:      "Android源码系列(17) -- QueuedWork"
date:       2018-11-05
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

这是个内部工具类，用于跟踪那些未完成的或尚未结束的全局任务，新任务通过方法 __queue__ 加入。添加 __finisher__ 的runnables，由 __waitToFinish__ 方法保证执行，用于保证任务已被处理完成。

这个类用于 __SharedPreference__ 编辑后修改异步写入到磁盘，所以设计一个在 __Activity.onPause__ 或类似地方等待写入操作机制，而这个机制也能用于其他功能。所有排队的异步任务都在一个独立、专用的线程上处理。

```java
public class QueuedWork
```

## 二、常量

可延迟 __Runnable__ 的延迟值。该值设置得尽可能大，但也需要足够小，令延迟难以察觉。

```java
private static final long DELAY = 100;
```

当 __waitToFinish()__ 运行超过 __MAX_WAIT_TIME_MILLIS__ 毫秒，发出警告

```java
private static final long MAX_WAIT_TIME_MILLIS = 512;
```

本类使用的锁对象

```java
private static final Object sLock = new Object();
```

## 三、静态变量

用此锁约束任何时候都只有一个线程在处理一个任务，意味着执行顺序就是任务添加的顺序。

因为任务执行过程中会一直持有 __sProcessingWork__ 锁，所以 __sProcessingWork__ 独立于 __sLock__ 避免阻塞整个类的运转

```java
private static Object sProcessingWork = new Object();
```

存放 __Finisher__ 的链表。通过 __added__ 方法添加，或 __removed__ 方法移除

```java
@GuardedBy("sLock")
private static final LinkedList<Runnable> sFinishers = new LinkedList<>();
```

__QueuedWork__ 内部的静态 __HandlerThread__

```java
@GuardedBy("sLock")
private static Handler sHandler = null;
```

存放通过 __queue__ 方法添加任务的队列

```java
@GuardedBy("sLock")
private static final LinkedList<Runnable> sWork = new LinkedList<>();
```

指示新任务是否能被延迟，默认为 __true__

```java
@GuardedBy("sLock")
private static boolean sCanDelay = true;
```

任务执行前等待的时间

```java
@GuardedBy("sLock")
private final static ExponentiallyBucketedHistogram
        mWaitTimes = new ExponentiallyBucketedHistogram(
        16);
```

正在等待处理的任务数

```java
private static int mNumWaits = 0;
```

## 四、静态方法

#### 4.1 getHandler()

__HandlerThread__ 实现细节请参考 [Android源码系列(11) -- HandlerThread](/2018/06/13/HandlerThread/)。该文章已经提到 __HandlerThread__ 本质是线程，启动 __HandlerThread__ 即启动子线程，并让任务在该线程上执行。

```java
private static Handler getHandler() {
    synchronized (sLock) {
        if (sHandler == null) {
            HandlerThread handlerThread = new HandlerThread("queued-work-looper",
                    Process.THREAD_PRIORITY_FOREGROUND);
            handlerThread.start();

            sHandler = new QueuedWorkHandler(handlerThread.getLooper());
        }
        return sHandler;
    }
}
```

#### 4.2 addFinisher(Runnable finisher)

添加 __finisher-runnable__ 等待 __queue()__ 异步处理任务，由 __SharedPreferences$Editor.startCommit()__ 使用

应注意的是，这个方法不会引起 __finisher__ 的执行。这只是调用者的集合，让调用者异步跟踪任务的最新状态。在大多数情况下，调用者(如：SharedPreferences) 会在 __add()__ 后很快又调用 __remove()__。这些任务实际在 __waitToFinish()__ 执行

```java
public static void addFinisher(Runnable finisher) {
    synchronized (sLock) {
        sFinishers.add(finisher);
    }
}
```

#### 4.3 removeFinisher(Runnable finisher)

移除一个已经添加的 __finisher-runnable__ 

```java
public static void removeFinisher(Runnable finisher) {
    synchronized (sLock) {
        sFinishers.remove(finisher);
    }
}
```

#### 4.4 waitToFinish()

触发排队任务马上执行。排队任务在独立的线程异步执行。任务和 __finishers__ 都在这个线程上运行，__finishers__ 通过这种方式实现检查排队任务是否完成。

__Activity.onPause()__、__BroadcastReceiver.onReceive()之后__、__处理Service.command之后__ 都会调用此方法，所以异步任务永远不会丢失

```java
public static void waitToFinish() {
    long startTime = System.currentTimeMillis();
    boolean hadMessages = false;

    Handler handler = getHandler();

    synchronized (sLock) {
        if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {
            // 延迟任务会在processPendingWork()中处理
            handler.removeMessages(QueuedWorkHandler.MSG_RUN);
        }

        // We should not delay any work as this might delay the finishers
        sCanDelay = false;
    }

    StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
    try {
        processPendingWork();
    } finally {
        StrictMode.setThreadPolicy(oldPolicy);
    }

    try {
        while (true) {
            Runnable finisher;
            
            // 从sFinishers获取一个Finisher
            synchronized (sLock) {
                finisher = sFinishers.poll();
            }

            if (finisher == null) {
                break;
            }
            
            // 执行Finisher
            finisher.run();
        }
    } finally {
        sCanDelay = true;
    }

    // 统计任务执行的总时间和等待次数
    synchronized (sLock) {
        long waitTime = System.currentTimeMillis() - startTime;

        if (waitTime > 0 || hadMessages) {
            mWaitTimes.add(Long.valueOf(waitTime).intValue());
            mNumWaits++;
        }
    }
}
```

在 __ActivityThread__ 类内共有五段源码调用 __QueuedWork.waitToFinish();__

位置一：

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

位置二：

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

位置三：

```java
@Override
public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
        int configChanges, PendingTransactionActions pendingActions, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(r, finished, reason, pendingActions);

        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        mSomeActivitiesChanged = true;
    }
}
```

位置四：

```java
@Override
public void handleStopActivity(IBinder token, boolean show, int configChanges,
        PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    r.activity.mConfigChangeFlags |= configChanges;

    final StopInfo stopInfo = new StopInfo();
    performStopActivityInner(r, stopInfo, show, true /* saveState */, finalStateRequest,
            reason);

    if (localLOGV) Slog.v(
        TAG, "Finishing stop of " + r + ": show=" + show
        + " win=" + r.window);

    updateVisibility(r, show);

    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    stopInfo.setActivity(r);
    stopInfo.setState(r.state);
    stopInfo.setPersistentState(r.persistentState);
    pendingActions.setStopInfo(stopInfo);
    mSomeActivitiesChanged = true;
}
```

位置五：

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
            callActivityOnStop(r, true /* saveState */, "sleeping");
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
            r.activity.performRestart(true /* start */, "handleSleeping");
            r.setState(ON_START);
        }
    }
}
```

#### 4.5 queue(Runnable work, boolean shouldDelay)

安排任务异步执行。__work__ 是新加入的任务，__shouldDelay__ 指明本任务是否能延迟处理。

__SharedPreferences.Editor.appy()__ 源码间接调用此方法，调用时参数 __shouldDelay__ 为 __true__

```java
public static void queue(Runnable work, boolean shouldDelay) {
    Handler handler = getHandler();

    synchronized (sLock) {
        // 任务放入sWork列表中
        sWork.add(work);

        // 然后放入一个Message作为通知
        if (shouldDelay && sCanDelay) {
            // sCanDelay默认为true
            handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
        } else {
            handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
        }
    }
}
```

#### 4.6 hasPendingWork()

当队列存在等待的异步任务时返回 __true__

```java
public static boolean hasPendingWork() {
    synchronized (sLock) {
        return !sWork.isEmpty();
    }
}
```

#### 4.7 processPendingWork()

从任务队列中取任务执行

```java
private static void processPendingWork() {
    // 此处使用sProcessingWork锁，保证sWork
    synchronized (sProcessingWork) {
        LinkedList<Runnable> work;

        synchronized (sLock) {
            // 克隆sWork为work实例，work内任务将送去执行
            work = (LinkedList<Runnable>) sWork.clone();
            // 清除sWork
            sWork.clear();

            // 利用QueuedWorkHandler.MSG_RUN移除相关Messages，避免以后收到通知但任务早已执行
            getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
        }
        
        // 根据work任务排列顺序依次执行
        if (work.size() > 0) {
            for (Runnable w : work) {
                w.run();
            }
        }
    }
}
```

## 五、QueuedWorkHandler

收到 __msg.what == MSG_RUN__ 消息的通知，去调用[processPendingWork()](/2018/11/05/QueuedWork/#47-processpendingwork)方法

```java
private static class QueuedWorkHandler extends Handler {
    static final int MSG_RUN = 1;

    QueuedWorkHandler(Looper looper) {
        super(looper);
    }

    public void handleMessage(Message msg) {
        // 收到Message
        if (msg.what == MSG_RUN) {
            // 触发方法调用，把sWork内所有任务批量送去执行
            processPendingWork();
        }
    }
}
```
