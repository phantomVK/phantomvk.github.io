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

## Poster

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

## AsyncPoster

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

## BackgroundPoster

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

再看看存放订阅信息和事件 __PendingPost__ 类的内部构造

```java
final class PendingPost {
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

    Object event;
    Subscription subscription;
    PendingPost next;

    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }

    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        return new PendingPost(event, subscription);
    }

    static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // Don't let the pool grow indefinitely
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }
}
```



## HandlerPoster

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

## MainThreadSupport

在主线程投递事件，

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

