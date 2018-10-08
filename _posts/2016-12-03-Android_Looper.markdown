---
layout:     post
title:      "Android源码系列(5) -- Looper"
date:       2016-12-03
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

# 一、数据成员 

每个Looper拥有一个消息队列，归属于一个线程。

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static Looper sMainLooper;  // 由Looper.class控制，全局静态的

final MessageQueue mQueue; // Looper持有的MessageQueue
final Thread mThread; // Looper所在的线程实例
```

# 二、初始化

每个线程创建一个Looper，如果线程尝试创建第二个Looper就会出现异常。

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // ThreadLacal.get()的Looper不为空，表明之前已经初始化过了
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 之前没有初始化们，这里开始初始化
    sThreadLocal.set(new Looper(quitAllowed));
}
```

先创建一个消息队列和Looper所运行的线程。方法是私有的，只能由`prepare()`调用。

```java
private Looper(boolean quitAllowed) {
    // 创建Looper时为这个Looper创建一个对应的MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

Android环境会通过这个方法自动创建主线程Looper，千万不要在主线程中再次调用这个方法。

```java
public static void prepareMainLooper() {
    // 主线程的消息队列禁止退出
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

# 三、启动Looper

把这个线程Looper中`queue`包含的消息送去处理

```java
public static void loop() {
    final Looper me = myLooper();
    // Looper没有通过prepare()方法初始化
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 从Looper中获取其MessageQueue
    final MessageQueue queue = me.mQueue;
    Binder.clearCallingIdentity(); // 确保线程就是本地线程，并实时跟踪线程身份
    final long ident = Binder.clearCallingIdentity();
    
    // 循环遍历，从消息队列取消息
    for (;;) {
        // 如果队列没有消息，会阻塞并等待有效消息
        Message msg = queue.next();
        // 队列返回null表明消息队列已经关闭，退出loop()的死循环
        if (msg == null) {
            return; // 消息队列关闭，Looper退出
        }
        // 通过对应的Handler进行回调：msg.handler.dispatchMessage(msg)
        msg.target.dispatchMessage(msg);
        final long newIdent = Binder.clearCallingIdentity(); // 确保消息在分发的时候线程没有改变
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }
        msg.recycleUnchecked(); // 消息体回收
    }
}
```


# 四、消息队列退出

消息队列关闭

```java
public void quit() {
    mQueue.quit(false);
}

public void quitSafely() {
    mQueue.quit(true);
}
```


# 五、一般方法

返回Looper所属的线程和消息队列

```java
public @NonNull Thread getThread() {
    return mThread;
}

public @NonNull MessageQueue getQueue() {
    return mQueue;
}

// 所有Looper实例只会持有同一个静态sMainLooper
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```

返回所属线程的Looper。如果子线程没有调用Looper.prepare()，sThreadLocal.get()为空。

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

获取线程Looper的消息队列

```java
public static @NonNull MessageQueue myQueue() {
    return myLooper().mQueue;
}
```

返回当前的线程是否就是Looper的线程

```java
public boolean isCurrentThread() {
    return Thread.currentThread() == mThread;
}
```


