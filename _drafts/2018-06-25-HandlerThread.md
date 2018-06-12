---
layout:     post
title:      "HandlerThread源码"
date:       2018-06-25
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

```java
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread
```

### 数据成员

```java
// 线程优先级
int mPriority;
// 线程id
int mTid = -1;
// 线程Looper
Looper mLooper;
private @Nullable Handler mHandler;
```

### 构造方法

```java
// 构造方法，使用默认线程优先级
public HandlerThread(String name) {
    super(name);
    mPriority = Process.THREAD_PRIORITY_DEFAULT;
}

// 构造，可自定义线程优先级
public HandlerThread(String name, int priority) {
    super(name);
    mPriority = priority;
}
```

### 成员方法

```java
/**
 * Call back method that can be explicitly overridden if needed to execute some
 * setup before Looper loops.
 */
protected void onLooperPrepared() {
}

@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

/**
 * This method returns the Looper associated with this thread. If this thread not been started
 * or for any reason isAlive() returns false, this method will return null. If this thread
 * has been started, this method will block until the looper has been initialized.  
 * @return The looper.
 */
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }

    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}

/**
 * @return a shared {@link Handler} associated with this thread
 * @hide
 */
@NonNull
public Handler getThreadHandler() {
    if (mHandler == null) {
        mHandler = new Handler(getLooper());
    }
    return mHandler;
}

/**
 * Quits the handler thread's looper.
 * <p>
 * Causes the handler thread's looper to terminate without processing any
 * more messages in the message queue.
 * </p><p>
 * Any attempt to post messages to the queue after the looper is asked to quit will fail.
 * For example, the {@link Handler#sendMessage(Message)} method will return false.
 * </p><p class="note">
 * Using this method may be unsafe because some messages may not be delivered
 * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
 * that all pending work is completed in an orderly manner.
 * </p>
 *
 * @return True if the looper looper has been asked to quit or false if the
 * thread had not yet started running.
 *
 * @see #quitSafely
 */
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}

/**
 * Quits the handler thread's looper safely.
 * <p>
 * Causes the handler thread's looper to terminate as soon as all remaining messages
 * in the message queue that are already due to be delivered have been handled.
 * Pending delayed messages with due times in the future will not be delivered.
 * </p><p>
 * Any attempt to post messages to the queue after the looper is asked to quit will fail.
 * For example, the {@link Handler#sendMessage(Message)} method will return false.
 * </p><p>
 * If the thread has not been started or has finished (that is if
 * {@link #getLooper} returns null), then false is returned.
 * Otherwise the looper is asked to quit and true is returned.
 * </p>
 *
 * @return True if the looper looper has been asked to quit or false if the
 * thread had not yet started running.
 */
public boolean quitSafely() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quitSafely();
        return true;
    }
    return false;
}

// 获取线程id，即Process.myTid()
public int getThreadId() {
    return mTid;
}
```


## 参考链接

https://phantomvk.github.io/2016/12/01/Android_Handler/

https://phantomvk.github.io/2016/12/03/Android_Looper/