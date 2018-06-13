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

### 类签名

HandlerThread是Android提供的封装类，父类是Thread。HandlerThread实例包含一个[Looper](https://phantomvk.github.io/2016/12/03/Android_Looper/)并用于构建[Handler](https://phantomvk.github.io/2016/12/01/Android_Handler/)。由于这是一个线程类，所以必须通过Thread.start()启动线程。

```java
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
// 通过上述Looper构建的Handler
private @Nullable Handler mHandler;
```

### 构造方法

```java
// 构造方法，默认线程优先级
public HandlerThread(String name) {
    super(name);
    mPriority = Process.THREAD_PRIORITY_DEFAULT;
}

// 构造方法，自定义线程优先级
public HandlerThread(String name, int priority) {
    super(name);
    mPriority = priority;
}
```

### 成员方法

```java
// 重写此方法以便在Looper开始loop前执行自定义操作
protected void onLooperPrepared() {
}

// 此类本质是线程，必须使用Thread.start()启动
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

// 获取线程中的Looper
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

// 从HandlerThread里面获取共享的Handler实例
@NonNull
public Handler getThreadHandler() {
    if (mHandler == null) {
        mHandler = new Handler(getLooper());
    }
    return mHandler;
}

// 调用Looper的退出方法quit()
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}

// 调用Looper的安全退出方法quitSafely()
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

- [Handler源码分析 - phantomVK](https://phantomvk.github.io/2016/12/01/Android_Handler/)

- [Looper源码分析 - phantomVK](https://phantomvk.github.io/2016/12/03/Android_Looper/)