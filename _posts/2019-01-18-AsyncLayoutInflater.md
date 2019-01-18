---
layout:     post
title:      "Android源码系列(19) -- AsyncLayoutInflater"
date:       2019-01-18
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:

    - Android源码系列
---

## 一、类签名

这是用于异步填充 __View__ 的帮助类。在主线程构建 __AsyncLayoutInflater__ 实例，并调用方法 __inflate(int, ViewGroup, OnInflateFinishedListener)__。视图填充完毕后在主线程回调 __OnInflateFinishedListener__ 通知调用者。

```java
public final class AsyncLayoutInflater
```

这适用于懒创建或响应用户交互的UI部分。有利于主线程继续响应用户时，重量级填充操作继续执行。填充布局需要来自 __ViewGroup.generateLayoutParams(AttributeSet)__ 的父布局，该方法线程安全。所有视图在构建过程中，禁止创建任何 __Handler__ 或调用 __Looper#myLooper()__ 。若布局无法在子线程进行异步填充，则操作会回退到主线程上执行。

注意，视图填充完成后不会加入到父布局中。这相当于调用 __LayoutInflater.inflate(int, ViewGroup, boolean)__ 时参数 __attachToRoot__ 为 __false__。而调用者很可能希望在 __OnInflateFinishedListener__ 通过调用 __ViewGroup.addView(View)__ 把视图放入父布局。

本填充器不支持设置 __LayoutInflater.Factory__ 或 __LayoutInflater.Factory2__。类似，也不支持填充包含 __fragment__ 的布局。源码版本Android 28

## 二、数据成员

```java
// 负责填充的LayoutInflater实例
LayoutInflater mInflater;

// Handler
Handler mHandler;

// 线程
InflateThread mInflateThread;
```

收到请求后进行填充操作

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

## 三、构造方法

构造方法内初始化了实际负责填充工作的填充器 __BasicInflater__。

```java
public AsyncLayoutInflater(@NonNull Context context) {
    mInflater = new BasicInflater(context);
    mHandler = new Handler(mHandlerCallback);
    mInflateThread = InflateThread.getInstance();
}
```

## 四、成员方法

主线程通过此方法添加新填充任务。

```java
@UiThread
public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
        @NonNull OnInflateFinishedListener callback) {
    // 回调不能为空，以便把填充完成的布局返回给调用者
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

## 五、OnInflateFinishedListener

在子线程处理完毕后，通过此回调把填充完成的 __View__ 返回给调用者。创建完成的 __View__ 需要调用者自行添加到 __ViewGroup__ 中，而不能像普通 __LayoutInflater__ 那样直接加入到父布局。

```java
public interface OnInflateFinishedListener {
    void onInflateFinished(@NonNull View view, @LayoutRes int resid,
            @Nullable ViewGroup parent);
}
```

## 六、InflateRequest

这个为构建任务的请求，请求包含参数：用于测绘的父布局、资源id、回调监听、已填充视图。初始化完成后的请求放入 __InflateThread__ 的阻塞队列等待处理。

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

## 七、BasicInflater

此类继承父类 __LayoutInflater__ 并重写父类方法 __onCreateView(String name, AttributeSet attrs)__。在此方法内，调用父类方法 __createView__ 反射构建目标视图。

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
                // 构建视图
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

## 八、InflateThread

负责填充视图的线程。线程内包含任务阻塞队列，这个队列的空间为10。当放入任务数量超过10时，如果主线程继续放入新任务，则主线程会被阻塞直到队列出现空余位置。这个类是单例，所以多个 __AsyncLayoutInflater__ 实例共享一个工作线程及内部线程池。

如果有非常多任务需要异步填充，由于每个填充的视图平均需要10ms的时间，所以会出现视图排队填充完成，造成长时间等待。

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

    public void runInner() {
        InflateRequest request;
        try {
            // 从队列获取等待任务
            request = mQueue.take();
        } catch (InterruptedException ex) {
            // Odd, just continue
            Log.w(TAG, ex);
            return;
        }

        try {
            // 从InflateRequest取出资源id、父布局样式作为参数，通过BasicInflater构建视图
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
        // 先尝试从缓存池中获取对象
        InflateRequest obj = mRequestPool.acquire();
        if (obj == null) {
            // 若缓存池没有缓存对象才创建新对象
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

    // 初始化完成的任务通过此方法放入队列等待处理
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

## 九、缺点

1. 不支持多线程并发处理任务；
2. 类修饰为 __final__，不能通过继承重写父类实现；
3. 任务等待队列不能自定义初始化大小；
4. 不支持设置 __LayoutInflater.Factory__ 或 __LayoutInflater.Factory2__；