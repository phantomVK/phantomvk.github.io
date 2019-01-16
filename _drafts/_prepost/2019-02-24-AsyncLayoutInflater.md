---
layout:     post
title:      "Android源码系列 -- AsyncLayoutInflater"
date:       2019-01-24
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 类签名

这是用于异步填充 __View__ 的辅助类。通过在主线程调用 __AsyncLayoutInflater__ 的方法 __inflate(int, ViewGroup, OnInflateFinishedListener) __。当视图填充完毕后会在主线程回调 __OnInflateFinishedListener__。

这便于主线程在继续响应用户的时候，重量级填充操作还能继续执行。异步填充的布局需要一个来自 __ViewGroup#generateLayoutParams(AttributeSet)__ 的父布局，该方法线程安全，且在视图构建过程中禁止创建任何 __Handler__ 或调用 __Looper#myLooper()__ 。若布局无法在子线程进行异步填充，则操作会回退到主线程进行。

注意，视图填充完成后不会加入到父布局中。这相当于调用 __LayoutInflater#inflate(int, ViewGroup, boolean) __ 时参数 __attachToRoot__ 为 __false__。而调用者很可能希望在 __OnInflateFinishedListener__ 中通过调用 __ViewGroup#addView(View)__ 把视图放入父布局。

本填充器不支持设置 __LayoutInflater.Factory__ 或 __LayoutInflater.Factory2__。类似的，也不支持填充包含 __fragment__ 的布局。

源码版本Android 28

```java
/**
 * <p>Helper class for inflating layouts asynchronously. To use, construct
 * an instance of {@link AsyncLayoutInflater} on the UI thread and call
 * {@link #inflate(int, ViewGroup, OnInflateFinishedListener)}. The
 * {@link OnInflateFinishedListener} will be invoked on the UI thread
 * when the inflate request has completed.
 *
 * <p>This is intended for parts of the UI that are created lazily or in
 * response to user interactions. This allows the UI thread to continue
 * to be responsive & animate while the relatively heavy inflate
 * is being performed.
 *
 * <p>For a layout to be inflated asynchronously it needs to have a parent
 * whose {@link ViewGroup#generateLayoutParams(AttributeSet)} is thread-safe
 * and all the Views being constructed as part of inflation must not create
 * any {@link Handler}s or otherwise call {@link Looper#myLooper()}. If the
 * layout that is trying to be inflated cannot be constructed
 * asynchronously for whatever reason, {@link AsyncLayoutInflater} will
 * automatically fall back to inflating on the UI thread.
 *
 * <p>NOTE that the inflated View hierarchy is NOT added to the parent. It is
 * equivalent to calling {@link LayoutInflater#inflate(int, ViewGroup, boolean)}
 * with attachToRoot set to false. Callers will likely want to call
 * {@link ViewGroup#addView(View)} in the {@link OnInflateFinishedListener}
 * callback at a minimum.
 *
 * <p>This inflater does not support setting a {@link LayoutInflater.Factory}
 * nor {@link LayoutInflater.Factory2}. Similarly it does not support inflating
 * layouts that contain fragments.
 */
```

```java
public final class AsyncLayoutInflater
```

示意图：

## 数据成员

```java
LayoutInflater mInflater;
Handler mHandler;
InflateThread mInflateThread;
```

```java
private Callback mHandlerCallback = new Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        InflateRequest request = (InflateRequest) msg.obj;
        if (request.view == null) {
            request.view = mInflater.inflate(
                    request.resid, request.parent, false);
        }
        request.callback.onInflateFinished(
                request.view, request.resid, request.parent);
        mInflateThread.releaseRequest(request);
        return true;
    }
};
```

## 构造方法

构造方法内初始化了实际负责填充工作的填充器 __BasicInflater__。

```java
public AsyncLayoutInflater(@NonNull Context context) {
    mInflater = new BasicInflater(context);
    mHandler = new Handler(mHandlerCallback);
    mInflateThread = InflateThread.getInstance();
}
```

## 成员方法

主线程通过此方法添加填充任务。

```java
@UiThread
public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
        @NonNull OnInflateFinishedListener callback) {
    // 填充完成回调不能为空，以便把填充完成的布局返回给调用者
    if (callback == null) {
        throw new NullPointerException("callback argument may not be null!");
    }
    // 从缓存池中获取一个空的填充请求
    InflateRequest request = mInflateThread.obtainRequest();
    // 把填充需要的数据存入填充请求中
    request.inflater = this;
    request.resid = resid;
    request.parent = parent;
    request.callback = callback;
    // 请求添加到子线程等待处理
    mInflateThread.enqueue(request);
}
```

## OnInflateFinishedListener

在子线程处理完毕后，通过此回调把填充完成的 __View__ 返回给调用者。此时创建完成的 __View__ 需要调用者自行添加到 __ViewGroup__ 中，而不能像主线程那样能直接放入父布局。

```java
public interface OnInflateFinishedListener {
    void onInflateFinished(@NonNull View view, @LayoutRes int resid,
            @Nullable ViewGroup parent);
}
```

## InflateRequest

这个为构建任务的请求，请求中包含：用于测绘宽高的父布局、资源id、成功回调、已填充视图。初始化完成后的请求放入 __InflateThread__ 的阻塞队列等待处理。

```java
private static class InflateRequest {
    // AsyncLayoutInflater
    AsyncLayoutInflater inflater;
    // 父布局
    ViewGroup parent;
    // 需填充资源的id
    int resid;
    // 填充完成的视图，为填充完成则为null
    View view;
    // 填充完成的回调
    OnInflateFinishedListener callback;

    InflateRequest() {
    }
}
```

## BasicInflater

继承父类 __LayoutInflater__，重写父类方法 __onCreateView(String name, AttributeSet attrs)__。在此方法内，利用了父类方法 __createView__ 反射目标视图。当视图构建出来之后，视图会直接返回给调用者，而不会给该视图添加父布局的布局参数。

```java
private static class BasicInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };

    BasicInflater(Context context) {
        super(context);
    }

    @Override
    public LayoutInflater cloneInContext(Context newContext) {
        return new BasicInflater(newContext);
    }

    @Override
    protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }

        return super.onCreateView(name, attrs);
    }
}
```

## InflateThread

负责填充视图的线程。线程内包含任务阻塞队列，这个队列的空间为10，当放入任务数量超过10时，如果主线程继续放入如任务，则任务队列会阻塞主线程直到队列出现空余位置。这个类是单例，所以多个 __AsyncLayoutInflater__ 实例共享一个工作线程及内部的线程池。

如果有非常多任务需要异步填充，由于每个填充的视图平均需要10ms的时间，所以会出现视图排队等待填充完成会耗费非常多时间。

```java
private static class InflateThread extends Thread {
    private static final InflateThread sInstance;
    static {
        sInstance = new InflateThread();
        sInstance.start();
    }

    public static InflateThread getInstance() {
        return sInstance;
    }

    // 等待处理InflateRequest实例的阻塞队列
    private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
    
    // 复用InflateRequest实例的缓存池
    private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);

    // Extracted to its own method to ensure locals have a constrained liveness
    // scope by the GC. This is needed to avoid keeping previous request references
    // alive for an indeterminate amount of time, see b/33158143 for details
    public void runInner() {
        InflateRequest request;
        try {
            request = mQueue.take();
        } catch (InterruptedException ex) {
            // Odd, just continue
            Log.w(TAG, ex);
            return;
        }

        try {
            // 从InflateRequest取出资源id、父布局样式作为参数
            // 通过BasicInflater构建的所需视图
            request.view = request.inflater.mInflater.inflate(
                    request.resid, request.parent, false);
        } catch (RuntimeException ex) {
            // Probably a Looper failure, retry on the UI thread
            Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                    + " thread", ex);
        }
        Message.obtain(request.inflater.mHandler, 0, request)
                .sendToTarget();
    }

    @Override
    public void run() {
        while (true) {
            runInner();
        }
    }

    // 从缓存池中获取一个有效InflateRequest用于构建任务参数
    public InflateRequest obtainRequest() {
        // 先从尝试从缓存池中获取对象
        InflateRequest obj = mRequestPool.acquire();
        if (obj == null) {
            // 若缓存池没有缓存对象则创建新对象
            obj = new InflateRequest();
        }
        return obj;
    }
    
    // 填充任务完成后，释放InflateRequest并放回缓存池中等待复用
    public void releaseRequest(InflateRequest obj) {
        obj.callback = null;
        obj.inflater = null;
        obj.parent = null;
        obj.resid = 0;
        obj.view = null;
        // 已重置对象放回缓存池
        mRequestPool.release(obj);
    }

    // 设置完成的任务通过此方法放入队列中等待处理
    public void enqueue(InflateRequest request) {
        try {
            mQueue.put(request);
        } catch (InterruptedException e) {
            throw new RuntimeException(
                    "Failed to enqueue async inflate request", e);
        }
    }
}
```

## 缺点

1. 任务不支持多线程；
2. 类通过 __final__ 修饰，不能通过继承重写父类实现；
3. 任务进入任务队列失败会直接排除异常，需要自行捕获处理；
4. 任务等待队列不能自定义初始化大小