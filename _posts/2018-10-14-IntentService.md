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

## 一、类签名

__IntentService__ 是异步处理 __Intent__ 的 __Service__ 抽象子类。客户端通过 __startService(Intent)__ 发送请求，服务根据需要启动，轮流处理收到的 __Intent__ ，所有任务处理完后服务自行结束。

```java
public abstract class IntentService extends Service
```

这种 “工作队列处理器” 模式主要适用于从应用主线程转移任务。 __IntentService__ 是简化这种模式和关注处理者的类，只需继承此类并实现 __onHandleIntent(Intent)__。__IntentService__ 会接收Intent，启动工作线程进行处理，并在适当时候关闭Service（没有任务时）。

所有工作由子线程串行处理，不会阻塞主线程。处理逻辑需要实现抽象方法 __onHandleIntent__ 实现。源码来自Android 28

## 二、数据成员

线程包含的Looper

```java
private volatile Looper mServiceLooper;
```

使用Looper的Handler

```java
private volatile ServiceHandler mServiceHandler;
```

工作线程名称

```java
private String mName;
```

重分发标志位

```java
private boolean mRedelivery;
```

## 三、内部类

#### ServiceHandler

绑定到 __Looper__ 的 __Handler__ 是一个静态内部类。 轮到对应 __Intent__ 处理时，实例送到 __handleMessage(Message msg)__ 并调用 __onHandleIntent()__ 。

```java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        // msg.arg1是新建任务时传入的startId，方法内检查是否需要结束Service
        stopSelf(msg.arg1);
    }
}
```

## 四、构造方法

创建 __IntentService__ 实例，由子类构造方法调用

```java
// @param name 为工作线程指定名称，主要方便Debug
public IntentService(String name) {
    super();
    mName = name;
}
```

## 五、成员方法

若 __enabled__ 为 __true__，__onStartCommand(Intent, int, int)__ 返回 __Service.START_REDELIVER_INTENT__。当 __onHandleIntent(Intent)__ 结束前服务死亡，服务会重启并重分发最近的 __Intent__。
若 __enabled__ 为 __false__，服务死亡时销毁所有 __Intent__。

```java
public void setIntentRedelivery(boolean enabled) {
    mRedelivery = enabled;
}
```

创建 __HandlerThread__ 和 __ServiceHandler__。

```java
@Override
public void onCreate() {
    super.onCreate();
    
    // 创建并启动HandlerThread，实例内部启动Looper
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    // 从HandlerThread获取Looper
    mServiceLooper = thread.getLooper();
    // 从Looper构建Handler
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

__Service__ 被 __startService()__ 触发时回调 __onStartCommand()__ 调用 __onStart()__ 。不应重写此方法，如有必要请重写 __onHandleIntent__ 。


```java
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```

此方法封装目标 __Intent__ 为 __Message__ ，送入 __Handler__ 的消息队列等待分发，处理顺序是FIFO。

```java
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

当 __Service__ 销毁时调用 __Looper.quit()__ ，方法内调用 __MessageQueue.quit(false)__ 关闭消息队列。

```java
@Override
public void onDestroy() {
    mServiceLooper.quit();
}
```

__onBind(Intent intent)__ 默认返回null。若需支持 __bindService()__ 需重写。

```java
@Override
@Nullable
public IBinder onBind(Intent intent) {
    return null;
}
```

分派任务时调用此方法，需在子类实现。每次只有一个 __Intent__ 执行，执行独立于应用其他工作线程。运行任务需较长时间会阻塞相同 __IntentService__ 的其他请求，但不会阻塞其他线程。

所有请求完成处理后 __IntentService__ 自动结束运行，无需主动调用 __stopSelf__ 方法。

```java
@WorkerThread
// @param intent 传递给Context#startService(Intent)的实例
protected abstract void onHandleIntent(@Nullable Intent intent);
```

## 六、自动结束

执行流程(1/2)：

- 假设有 __IntentService__ 实现类 __WorkerService__，通过 __Intent__ 首次发起任务；
-  __WorkerService__ 首次启动进入 __onCreate()__ 初始化；
-  __onStartCommand()__ 接收 __Intent__ 同时，还能收到配对 __startId__；
- __onStart()__ 方法内：__Intent__ 和 __startId__ 构造为 __Message__，放入消息队列等待处理；

```java
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    // 用传入参数构造Message，并加入到ServiceHandler的消息队列中
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```

执行流程(2/2)：

- 当 __ServiceHandler__ 任务完成后，从 __Message__ 取出 __startId__ 去调用 __Service.stopSelf(int)__；
- 系统知道这个传给 __WorkerService__ 的 __Intent__ 已经执行完毕；
- 若还有其他 __startId__ 的 __Intent__ 没调用 __stopSelf__，__WorkerService__ 继续运行直至所有任务完成

```java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        // 执行取出的任务
        onHandleIntent((Intent)msg.obj);
        // 调用Service.stopSelf(int)
        stopSelf(msg.arg1);
    }
}
```

## 七、相关链接

- [Android源码系列(4) -- Handler](/2016/12/01/Android_Handler/)

- [Android源码系列(5) -- Looper](/2016/12/03/Android_Looper/)

- [Android源码系列(11) -- HandlerThread](/2018/06/13/HandlerThread/)

