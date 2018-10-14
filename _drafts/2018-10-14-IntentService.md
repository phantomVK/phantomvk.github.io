---
layout:     post
title:      "Android源码系列(14) -- IntentService"
date:       2018-10-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

源码来自Android 28

## 一、类签名

```java
/**
 * IntentService is a base class for {@link Service}s that handle asynchronous
 * requests (expressed as {@link Intent}s) on demand.  Clients send requests
 * through {@link android.content.Context#startService(Intent)} calls; the
 * service is started as needed, handles each Intent in turn using a worker
 * thread, and stops itself when it runs out of work.
 *
 * <p>This "work queue processor" pattern is commonly used to offload tasks
 * from an application's main thread.  The IntentService class exists to
 * simplify this pattern and take care of the mechanics.  To use it, extend
 * IntentService and implement {@link #onHandleIntent(Intent)}.  IntentService
 * will receive the Intents, launch a worker thread, and stop the service as
 * appropriate.
 *
 * <p>All requests are handled on a single worker thread -- they may take as
 * long as necessary (and will not block the application's main loop), but
 * only one request will be processed at a time.
 *
 * <p class="note"><b>Note:</b> IntentService is subject to all the
 * <a href="/preview/features/background.html">background execution limits</a>
 * imposed with Android 8.0 (API level 26). In most cases, you are better off
 * using {@link android.support.v4.app.JobIntentService}, which uses jobs
 * instead of services when running on Android 8.0 or higher.
 * </p>
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 * <p>For a detailed discussion about how to create services, read the
 * <a href="{@docRoot}guide/components/services.html">Services</a> developer
 * guide.</p>
 * </div>
 *
 * @see android.support.v4.app.JobIntentService
 * @see android.os.AsyncTask
 */
public abstract class IntentService extends Service {
```

## 二、数据成员

```java
// 线程包含的Looper
private volatile Looper mServiceLooper;

// 使用Looper的Handler
private volatile ServiceHandler mServiceHandler;

// 工作线程名称
private String mName;

private boolean mRedelivery;
```

## 三、内部类

```java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
}
```

## 四、构造方法

创建IntentService实例，由子类构造方法调用

```java
// @param name 为工作线程指定名称，主要方便Debug
public IntentService(String name) {
    super();
    mName = name;
}
```

## 五、成员方法

```java
/**
 * Sets intent redelivery preferences.  Usually called from the constructor
 * with your preferred semantics.
 *
 * <p>If enabled is true,
 * {@link #onStartCommand(Intent, int, int)} will return
 * {@link Service#START_REDELIVER_INTENT}, so if this process dies before
 * {@link #onHandleIntent(Intent)} returns, the process will be restarted
 * and the intent redelivered.  If multiple Intents have been sent, only
 * the most recent one is guaranteed to be redelivered.
 *
 * <p>If enabled is false (the default),
 * {@link #onStartCommand(Intent, int, int)} will return
 * {@link Service#START_NOT_STICKY}, and if the process dies, the Intent
 * dies along with it.
 */
public void setIntentRedelivery(boolean enabled) {
    mRedelivery = enabled;
}
```

```java
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    
    // 创建并启动HandlerThread，其实例内部已启动Looper
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    // 从HandlerThread获取Looper
    mServiceLooper = thread.getLooper();
    // 从Looper构建Handler
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

当Service被 __startService()__ 调用，__onStartCommand(...)__ 方法会被回调，并触发 __onStart(...)__ 方法。


```java
/**
 * You should not override this method for your IntentService. Instead,
 * override {@link #onHandleIntent}, which the system calls when the IntentService
 * receives a start request.
 * @see android.app.Service#onStartCommand
 */
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```

此方法封装送入的 __Intent__ 为 __Message__ ，然后送入到 __Handler__ 的消息队列依次等待分发。

```java
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

当 __Service__ 被销毁时调用 __Looper.quit()__ ，此方法内部调用 __MessageQueue..quit(false)__ 立即关闭消息队列。

```java
@Override
public void onDestroy() {
    mServiceLooper.quit();
}
```

```java
/**
 * Unless you provide binding for your service, you don't need to implement this
 * method, because the default implementation returns null.
 * @see android.app.Service#onBind
 */
@Override
@Nullable
public IBinder onBind(Intent intent) {
    return null;
}
```

```java
/**
 * This method is invoked on the worker thread with a request to process.
 * Only one Intent is processed at a time, but the processing happens on a
 * worker thread that runs independently from other application logic.
 * So, if this code takes a long time, it will hold up other requests to
 * the same IntentService, but it will not hold up anything else.
 * When all requests have been handled, the IntentService stops itself,
 * so you should not call {@link #stopSelf}.
 *
 * @param intent The value passed to {@link
 *               android.content.Context#startService(Intent)}.
 *               This may be null if the service is being restarted after
 *               its process has gone away; see
 *               {@link android.app.Service#onStartCommand}
 *               for details.
 */
@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);
```

## 六、相关链接

[Android源码系列(4) -- Handler](https://phantomvk.github.io/2016/12/01/Android_Handler/)

[Android源码系列(5) -- Looper](https://phantomvk.github.io/2016/12/03/Android_Looper/)

[Android源码系列(11) -- HandlerThread](https://phantomvk.github.io/2018/06/13/HandlerThread/)

