---
layout:     post
title:      "EventBus源码剖析(5) -- Poster"
date:       2018-12-10
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、Poster

这是所有 __Poster__ 实现的共同接口，包含一个抽象方法。子类实现该抽象方法后，可接收到订阅记录 __Subscription__ 和事件 __Object__。实现类需要根据自身特性，把事件按照既定模式发送给订阅者的接收方法。

```java
interface Poster {

    // 把需要发送给指定Subscription的事件加到队列中
    // @param subscription 事件记录Subscription
    // @param event 发送给订阅者的事件
    void enqueue(Subscription subscription, Object event);
}
```

## 二、AsyncPoster

在后台异步投递事件，每个使用 __AsyncPoster__ 的 __Runnable__ 都有自己的线程，适合耗时但不占用处理器时间片的io操作。任务运行完毕后线程归还给线程池。__AsyncPoster__ 和 __BackgroundPoster__ 共享同一个 __EventBus__ 线程池，该线程池类型为 __Executors.newCachedThreadPool()__。

```java
class AsyncPoster implements Runnable, Poster {
    // 事件投递队列
    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    // 向队列存入订阅记录和事件，并激活线程池进行事件派发
    public void enqueue(Subscription subscription, Object event) {
        // 创建PendingPost实例，存入订阅记录subscription和事件event
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        // PendingPost实例放入队列等待派发
        queue.enqueue(pendingPost);
        // 任务放入线程池等待执行
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        // 从事件投递队列获取一个任务执行
        PendingPost pendingPost = queue.poll();
        // pendingPost不能为空
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        // 把事件发送给订阅者
        eventBus.invokeSubscriber(pendingPost);
    }
}
```

## 三、BackgroundPoster

#### 3.1 BackgroundPoster

__BackgroundPoster__ 实现后台投递事件。__BackgroundPoster__ 本身同时实现 __Runnable__ 接口，这样就可以把类实例直接送到线程池中执行。线程池执行任务，从事件队列获取需要派发的任务并执行。__BackgroundPoster__ 只有一个运行线程，按任务进入队列的顺序依次执行，适合大量短小的任务。如果队列没有任务，该 __Runnable__ 会退出，线程也会归还给线程池。

从前文介绍可知，这里使用的线程池实现是 __Executors.newCachedThreadPool()__。

```java
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;
    
    // executor是否正在运行
    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    // 订阅记录和其订阅的事件进队
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
    
    // 把本类的实例放入线程池之后，由线程池调度执行
    @Override
    public void run() {
        try {
            try {
                // 循环执行，直到完成PendingPostQueue队列里所有任务
                while (true) {
                    // 从PendingPostQueue队列获取PendingPost实例
                    PendingPost pendingPost = queue.poll(1000);
                    // 获取超时会出现PendingPost实例为空
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
                // 本方法在线程池执行过程中被中断，捕获InterruptedException
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
                // 缓存对象放入缓存队列中
                pendingPostPool.add(pendingPost);
            }
        }
    }
}
```

#### 3.3 PendingPostQueue

__AsyncPoster__ 和 __BackgroundPoster__ 拥有各自的 __PendingPostQueue__，用于存放所有待完成的任务。

```java
final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

    // 存入PendingPost实例
    synchronized void enqueue(PendingPost pendingPost) {
        // 不能向队列存入空任务
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        // 队列已有其他任务，新任务放入队列尾部
        if (tail != null) {
            tail.next = pendingPost;
            tail = pendingPost;
        } else if (head == null) {
            // 队列是空的，新任务直接进队
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        notifyAll();
    }

    // 从队列中获取任任务
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

    // 等待maxMillisToWait毫秒或被notifyAll，再从队列中去任务
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
    // 处理队列
    private final PendingPostQueue queue;
    // 执行消息的超时时间
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    // 构造方法
    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }
    
    // 存入PendingPost实例
    public void enqueue(Subscription subscription, Object event) {
        // 创建PendingPost实例，存入订阅记录和事件
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            // PendingPost实例放入队列等待派发
            queue.enqueue(pendingPost);
            // handler没在运行
            if (!handlerActive) {
                // 向Handler发送一个Message
                handlerActive = true;
                // 发送Message失败
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
                // 从队列中获取任务
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // 在同步下再次检查
                        pendingPost = queue.poll();
                        // 队列中没有任务
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                // 已获得任务，把任务的事件发给对应订阅方法
                eventBus.invokeSubscriber(pendingPost);
                // 计算接收者方法处理事件耗费的时长
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
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    // 内部类实现外部接口
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

