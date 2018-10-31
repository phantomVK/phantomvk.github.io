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

所有工作由一个工作线程串行处理，不会阻塞主线程。处理逻辑需要实现抽象方法 __onHandleIntent__ 实现。

源码来自Android 28

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

绑定到 __Looper__ 的 __Handler__ 是一个静态内部类。 轮到对应 __Intent__ 处理时，实例送到 __handleMessage(Message msg)__ 并调用 __onHandleIntent()__ 。

```java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1); // 方法内检查是否需要结束
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

若 __enabled__ 为 __true__，__onStartCommand(Intent, int, int)__ 返回 __Service.START_REDELIVER_INTENT__。当 __onHandleIntent(Intent)__ 执行结束前服务死亡，服务会重启并重分发最近那个intent。
若 __enabled__ 为 __false__，服务死亡时所有Intent一并销毁。

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

当 __Service__ 销毁时调用 __Looper.quit()__ ，此方法内部调用 __MessageQueue.quit(false)__ 关闭消息队列。

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

工作线程分派任务时调用此方法，需在子类实现。每次只有一个 __Intent__ 执行，但执行独立于其他应用逻辑的工作线程。运行需较长时间会阻塞相同 __IntentService__ 中的其他请求，不会阻塞其他东西。

所有请求成功处理后 __IntentService__ 自动停止运行，开发者不需要调用 __stopSelf__ 方法。

```java
// @param intent 传递给Context#startService(Intent)的实例
@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);
```

## 六、相关链接

[Android源码系列(4) -- Handler](https://phantomvk.github.io/2016/12/01/Android_Handler/)

[Android源码系列(5) -- Looper](https://phantomvk.github.io/2016/12/03/Android_Looper/)

[Android源码系列(11) -- HandlerThread](https://phantomvk.github.io/2018/06/13/HandlerThread/)

