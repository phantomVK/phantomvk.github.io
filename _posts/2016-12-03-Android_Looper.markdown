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

// 由Looper.class控制，静态全局的MainLooper
private static Looper sMainLooper;

// Looper持有的MessageQueue
final MessageQueue mQueue;

// Looper所在的线程实例
final Thread mThread;
```

# 二、初始化

每个线程仅能创建一个Looper，如果线程尝试创建第二个Looper会引起异常。

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // ThreadLacal.get()的Looper不为空，表明之前已经初始化过了
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }

    // 之前没有初始化则开始初始化
    sThreadLocal.set(new Looper(quitAllowed));
}
```

先创建[MessageQueue](/2018/11/02/MessageQueue/)，并把线程实例保存到Looper中。方法是私有的，只能由`prepare()`调用。

```java
private Looper(boolean quitAllowed) {
    // 创建Looper时为这个Looper创建对应的MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

运行时通过这个方法自动创建主线程Looper，千万不要在主线程中再次调用这个方法。__sMainLooper__ 是静态变量，所以任意 __Looper__ 都能通过 __sMainLooper__ 向主线程发送消息。

```java
public static void prepareMainLooper() {
    // 主线程的消息队列禁止退出
    prepare(false);
    
    // 且每个进程最多只有一个MainLooper
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

# 三、启动Looper

启动 __Looper__ 的 __MessageQueue__。关于在 __queue.next()__ 上阻塞的详情请看 [MessageQueue源码 - 5.5 next](/2018/11/02/MessageQueue/#55-next)

```java
public static void loop() {
    final Looper me = myLooper();

    // Looper没有通过prepare()方法初始化就抛出异常
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }

    // 从Looper中获取其MessageQueue
    final MessageQueue queue = me.mQueue;
    // 确保线程就是本地线程，并实时跟踪线程身份
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    
    // 循环遍历，从消息队列取消息
    for (;;) {
        // 如果队列没有消息，queue.next()内部会阻塞等待
        Message msg = queue.next();

        // 队列返回null表明消息队列已经关闭
        if (msg == null) {
            // 消息队列关闭，Looper退出
            return;
        }

        // 通过对应的Handler进行回调
        msg.target.dispatchMessage(msg);

        // 确保消息在分发的时候线程没有出错
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        // 消息体回收
        msg.recycleUnchecked();
    }
}
```


# 四、消息队列退出

消息队列关闭

```java
public void quit() {
    mQueue.quit(false);
}
```

消息队列安全关闭

```java
public void quitSafely() {
    mQueue.quit(true);
}
```


# 五、一般方法

返回Looper所属线程和消息队列

```java
public @NonNull Thread getThread() {
    return mThread;
}

public @NonNull MessageQueue getQueue() {
    return mQueue;
}

// 所有Looper实例持有同一个静态sMainLooper
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
