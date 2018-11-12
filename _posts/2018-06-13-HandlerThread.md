---
layout:     post
title:      "Android源码系列(11) -- HandlerThread"
date:       2018-06-13
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

HandlerThread是Android提供的封装类，父类是Thread。HandlerThread实例包含一个[Looper](/2016/12/03/Android_Looper/)并用于构建[Handler](/2016/12/01/Android_Handler/)。由于这是一个线程类，所以必须通过Thread.start()启动线程。

```java
public class HandlerThread extends Thread
```

## 二、数据成员

线程优先级

```java
int mPriority;
```

线程id

```java
int mTid = -1;
```

线程Looper

```java
Looper mLooper;
```

通过上述Looper构建的Handler

```java
private @Nullable Handler mHandler;
```

## 三、构造方法

构造方法，默认线程优先级

```java
public HandlerThread(String name) {
    super(name);
    mPriority = Process.THREAD_PRIORITY_DEFAULT;
}
```

构造方法，自定义线程优先级

```java
public HandlerThread(String name, int priority) {
    super(name);
    mPriority = priority;
}
```

## 四、成员方法

此类本质是线程，必须使用Thread.start()启动。

当线程的Looper通过loop()方法启动后，就会在该方法内轮询获取消息。只有在Looper退出后，下一行 __mTid = -1;__ 才会执行，重置线程id后结束run()的运行。

```java
@Override
public void run() {
    // 获取线程Tid
    mTid = Process.myTid();
    
    // 初始化Looper
    Looper.prepare();
    
    // 获取Looper
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    
    // 设置线程优先级
    Process.setThreadPriority(mPriority);
    onLooperPrepared();

    // 启动Looper
    Looper.loop();
    mTid = -1;
}
```

重写此方法以便在Looper开始loop前执行自定义操作

```java
protected void onLooperPrepared() {
}
```

获取线程中的Looper

```java
public Looper getLooper() {
    // 线程不是存活状态则返回null
    if (!isAlive()) {
        return null;
    }

    // 即使thread已经启动，Looper实例也只在其创建完成后才能返回
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
```

从HandlerThread里面获取共享的Handler实例

```java
@NonNull
public Handler getThreadHandler() {
    if (mHandler == null) {
        mHandler = new Handler(getLooper());
    }
    return mHandler;
}
```

调用Looper的退出方法quit()

```java
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}
```

调用Looper的安全退出方法quitSafely()

```java
public boolean quitSafely() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quitSafely();
        return true;
    }
    return false;
}
```

获取线程id，即Process.myTid()

```java
public int getThreadId() {
    return mTid;
}
```
