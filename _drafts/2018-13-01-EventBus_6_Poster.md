---
layout:     post
title:      "EventBus源码剖析(6) -- Poster"
date:       2018-11-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、Poster

这是所有 __Poster__ 实现的共同接口，包含一个抽象方法。子类实心该抽象方法后，会接收到订阅记录 __Subscription__ 和事件 __Object__。实现类需要根据自身的特性，把事件按照既定模式发送给订阅者的接收方法。

```java
interface Poster {

    /**
     * Enqueue an event to be posted for a particular subscription.
     *
     * @param subscription Subscription which will receive the event.
     * @param event        Event that will be posted to subscribers.
     */
    void enqueue(Subscription subscription, Object event);
}
```

## 二、AsyncPoster

在后台投递事件。

```java
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }

}
```

## 三、BackgroundPoster

#### 3.1 BackgroundPoster

__BackgroundPoster__ 实现在后台投递事件。__BackgroundPoster__ 还同时实现了 __Runnable__ ，这样就可以把类实例直接送到线程池中执行。

从前文介绍可知，这里使用的线程池实现是 __Executors.newCachedThreadPool()__。

```java
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        // 从PendingPost缓存池中获取缓存对象，用于保存subscription和event
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        // 把设置完毕的PendingPost实例存入队列中
        synchronized (this) {
            queue.enqueue(pendingPost);
            // 激活运行
            if (!executorRunning) {
                executorRunning = true;
                // 实现了Runnable接口，所以可以直接在线程池中执行
                eventBus.getExecutorService().execute(this);
            }
        }
    }
    
    // 把本类的实例放入线程池之后，有线程池调度执行
    @Override
    public void run() {
        try {
            try {
                // 循环执行，知道完成PendingPostQueue队列里所有任务
                while (true) {
                    // 从PendingPostQueue队列中获取一个PendingPost实例
                    PendingPost pendingPost = queue.poll(1000);
                    // PendingPost实例为空
                    if (pendingPost == null) {
                        synchronized (this) {
                            // 加锁后再到队列获取PendingPost实例
                            pendingPost = queue.poll();
                            // 队列中已经没有任务，退出执行
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    // 在线程池的线程中执行事件派发
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                // 本方法在线程池执行过程中被终端，捕获InterruptedException
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            // 结束运行，把executorRunning设置为false
            executorRunning = false;
        }
    }
}
```

#### 3.2 PendingPost

再看看存放订阅信息和事件 __PendingPost__ 类的内部构造。

```java
final class PendingPost {
    // 所有PendingPost共享相同ArrayList<PendingPost>缓存池，最大容量为10000
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

    // 将要处理的事件
    Object event;
    
    // 接收事件的订阅者信息
    Subscription subscription;
    
    // 指向下一个PendingPost实例的引用，在PendingPostQueue中使用
    PendingPost next;

    // 构造方法，可知构造新实例时next为null
    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }
    
    // 从缓存池中获取缓存实例，并把订阅信息和事件放入取得的实例中
    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            // 缓存池非空
            if (size > 0) {
                // 从缓存池中获取最后一个实例
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                // 放入通知事件
                pendingPost.event = event;
                // 放入订阅信息
                pendingPost.subscription = subscription;
                // 初始化next为null
                pendingPost.next = null;
                return pendingPost;
            }
        }
        // 缓存池没有已缓存的实例，直接创建新实例
        return new PendingPost(event, subscription);
    }

    // PendingPost负载的事件已经发送给订阅方法，所以可以回收PendingPost到缓存池
    static void releasePendingPost(PendingPost pendingPost) {
        // 相关数据成员置空
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // 限制缓存池最大容量为10000，避免缓存池无限扩容
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }
}
```

#### 3.3 PendingPostQueue

```java
final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

    synchronized void enqueue(PendingPost pendingPost) {
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        if (tail != null) {
            tail.next = pendingPost;
            tail = pendingPost;
        } else if (head == null) {
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        notifyAll();
    }

    synchronized PendingPost poll() {
        PendingPost pendingPost = head;
        if (head != null) {
            head = head.next;
            if (head == null) {
                tail = null;
            }
        }
        return pendingPost;
    }

    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
        if (head == null) {
            wait(maxMillisToWait);
        }
        return poll();
    }
}
```

## 四、HandlerPoster

```java
public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```

## 五、MainThreadSupport

在主线程投递事件，在Android中是UI线程。如果在其他系统中使用，可以自行指定特定线程为"主线程"。

```java
/**
 * Interface to the "main" thread, which can be whatever you like. Typically on Android, Android's main thread is used.
 */
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    class AndroidHandlerMainThreadSupport implements MainThreadSupport {

        private final Looper looper;

        public AndroidHandlerMainThreadSupport(Looper looper) {
            this.looper = looper;
        }

        @Override
        public boolean isMainThread() {
            return looper == Looper.myLooper();
        }

        @Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
    }

}
```

