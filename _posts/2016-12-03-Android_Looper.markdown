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
private static Looper sMainLooper;  // 由Looper.class控制

final MessageQueue mQueue;
final Thread mThread;
```

# 二、初始化

每个线程创建一个Looper，如果线程尝试创建第二个Looper就会出现异常。

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

先创建一个消息队列和Looper所运行的线程。方法是私有的，只能由`prepare()`调用。

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

Android环境会通过这个方法自动创建主线程Looper，千万不要在主线程中再次调用这个方法。

```java
public static void prepareMainLooper() {
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

把这个线程Looper的`queue`里面的消息送去处理

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    Binder.clearCallingIdentity(); // 确保线程就是本地线程，并实时跟踪线程身份
    final long ident = Binder.clearCallingIdentity();
    
    // 循环遍历，从消息队列去消息
    for (;;) {
        Message msg = queue.next();
        if (msg == null) {
            return; // 消息队列关闭，Looper退出
        }
        msg.target.dispatchMessage(msg); // 消息发送到Handler回调
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


