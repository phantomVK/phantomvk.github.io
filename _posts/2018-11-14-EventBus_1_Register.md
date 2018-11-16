---
layout:     post
title:      "EventBus源码剖析(1) -- 注册与注销订阅"
date:       2018-11-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、简介

#### 1.1 特性

__EventBus__ 是为 __Android__ 而设的中心化 __publish/subscribe (发布/订阅)__ 事件系统。事件通过 __post(Object)__ 提交到总线，总线把事件投递给订阅者。当然，该订阅者需拥有匹配类型消息的处理方法。

![EventBus-Publish-Subscribe](/img/android/EventBus/EventBus-Publish-Subscribe.png)

为了能接收事件，订阅者需要通过 __register(Object)__ 把自己注册到总线上。一旦成功注册，订阅者将可以持续接收关心的事件，直到通过 __unregister(Object)__ 注销监听。

处理事件的方法需满足以下条件：

- 用 __Subscribe__ 进行注解；
- 方法可见性为 __public__；
- 方法返回值为 __void__；
- 仅含有一个参数，参数类型为接收的事件类；

#### 1.2 优点

* 简化不同组件间通讯
   *  对事件发送者和接收者进行解耦
   *  与Activities、 Fragments、和 background threads 运行得很好
   *  避免复杂、易错的依赖和生命周期问题
* 令实现代码更简洁
* 运行速度快
* 库体积小 (约50KB)
* 已经过累计 100,000,000+ 安装量的应用验证
* 有消息分发线程、订阅者优先级等的高级特性

## 二、用法

#### 2.1 订阅者

订阅者需要在界面合适的生命周期，把自己注册到消息总线开始接收事件。同时，由于事件的基本接收单位是方法，所以需要给接收事件的方法添加注解，以便 __EventBus__ 把事件发送到该方法上。

接收者方法需要遵循以下规则：

-  使用 __EventBus__ 的注解修饰方法；
-  方法不能为 __private__，才能让  __EventBus__ 获取该方法；
-  方法必须只有一个参数，且参数类型就是所关心事件的类型；

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (EventBus.getDefault().isRegistered(this)) {
            EventBus.getDefault().register(this)
        }
    }
    
    // 注册类必须包含至少一个接收事件的方法
    @Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
    fun onEventReceived(event: UserEvent) {
        val name = event.name
        val age = event.age
        Log.i(TAG, "Event received, Name: $name, Age: $age.")
    }
    
    override fun onDestroy() {
        super.onDestroy()
        EventBus.getDefault().unregister(this)
    }
}
```

把实例注册到 __EventBus__ 后，接收者类对事件不再关心时，也需要在合适时间点注销订阅。每个接收者类只需向 __EventBus__ 注册一次。为避免多次注册，可以像上述代码一样，在注册前检查是否已注册。

#### 2.2 发布者

对事件发布者来说，工作就比较简单了。只需要构建目标事件，把数据或负载内容构建到事件中发出即可。

```java
fun postEvent() {
    val user = UserEvent("Mike", 24)
    EventBus.getDefault().post(user)
}
```

#### 2.3 事件消息体

这是示例的消息体，消息体重包含用户的名字和年龄

```java
class UserEvent(val name: String, val age: Int)
```

如果消息只为发出简单通知，事件消息体甚至可以不含任何数据成员。例如同 __Kotlin__ 实现的类：

```java
class Notification
```

## 三、初始化

#### 3.1 单例

整个 __EventBus__ 通过以下方法创建单例。所有在此单例发送的事件，只对注册在单例里的订阅者有效。为了所有事件能在同一个 __EventBus__ 内流动，一定要从此方法获取 __EventBus__ 实例。

```java
public class EventBus {
    // 变量使用volatile修饰
    static volatile EventBus defaultInstance;

    // 同一进程内有效，传统的双重检验锁
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
}
```

调用 __getDefault()__ 方法时，以下两个常量也获得初始化。

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

事件类型缓存，实例为HashMap<>

```java
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();
```

#### 3.2 基础构造

单例的初始化调用此构造方法，然后方法内又调用自身的另一个构造方法：

```java
public EventBus() {
    this(DEFAULT_BUILDER);
}
```

#### 3.3 深入构造

构造过程对以下数据成员进行赋值

```java
// 按照事件类型分类订阅，事件类型与对应的订阅记录
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

// 按照订阅者类型分类
private final Map<Object, List<Class<?>>> typesBySubscriber;

// 粘性事件
private final Map<Class<?>, Object> stickyEvents;

// 用于提供访问主线程的能力
private final MainThreadSupport mainThreadSupport;

// 主线程消息分发器
private final Poster mainThreadPoster;

// 后台线程消息分发器
private final BackgroundPoster backgroundPoster;

// 异步消息发布器
private final AsyncPoster asyncPoster;

// 订阅者方法查找器
private final SubscriberMethodFinder subscriberMethodFinder;

// ExecutorService
private final ExecutorService executorService;

private final boolean throwSubscriberException;
private final boolean logSubscriberExceptions;
private final boolean logNoSubscriberMessages;
private final boolean sendSubscriberExceptionEvent;
private final boolean sendNoSubscriberEvent;
private final boolean eventInheritance;

// 索引总计
private final int indexCount;

// 日志类
private final Logger logger;
```

构造方法：

```java
EventBus(EventBusBuilder builder) {
    // 从EventBusBuilder获取Logger
    logger = builder.getLogger();
    
    // 初始化容器
    subscriptionsByEventType = new HashMap<>();
    typesBySubscriber = new HashMap<>();
    stickyEvents = new ConcurrentHashMap<>();
    
    // 从EventBusBuilder获取MainThreadSupport
    mainThreadSupport = builder.getMainThreadSupport();
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;

    // 构建BackgroundPoster
    backgroundPoster = new BackgroundPoster(this);

    // 构建AsyncPoster
    asyncPoster = new AsyncPoster(this);

    // 获取数量
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;

    // 初始化订阅者方法查找器
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
            builder.strictMethodVerification, builder.ignoreGeneratedIndex);

    // 初始化各种标志位
    logSubscriberExceptions = builder.logSubscriberExceptions;
    logNoSubscriberMessages = builder.logNoSubscriberMessages;
    sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
    sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
    throwSubscriberException = builder.throwSubscriberException;
    eventInheritance = builder.eventInheritance;
    
    // ExecutorService
    executorService = builder.executorService;
}
```

#### 3.4 PostingThreadState

单例初始化过程还初始化了 __ThreadLocal__ 实例

```java
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};
```

__PostingThreadState__ 是变量的封装，用在 __ThreadLocal__ 中，以便快速设置、获取多个变量。

```java
final static class PostingThreadState {
    // 事件队列
    final List<Object> eventQueue = new ArrayList<>();

    // 消息是否在投递中
    boolean isPosting;

    // 是否在主线程
    boolean isMainThread;

    // 订阅记录
    Subscription subscription;

    // 事件
    Object event;

    // 是否已被取消投递
    boolean canceled;
}
```

## 四、注册事件

#### 4.1 register

所有订阅者通过此方法向 __EventBus__ 注册，以便在注册后收取所关心的事件。注销订阅则通过方法 __unregister(Object)__，这样观察者就能在不再关心事件的时候取消订阅。

```java
public void register(Object subscriber) {
    // 获取订阅者的Class
    Class<?> subscriberClass = subscriber.getClass();

    // 从订阅者方法中找出该类接收事件的方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);

    synchronized (this) {
        // 逐个注册订阅者中的订阅方法
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

#### 4.2 subscribe

此方法必须在同步块中调用，主要是把订阅方法按照订阅事件分类

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 订阅者方法的事件类型，即订阅者方法唯一参数
    Class<?> eventType = subscriberMethod.eventType;

    // 构建新记录
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

    // 用事件类型获取所有订阅记录
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);

    if (subscriptions == null) {
        // 该事件未曾有订阅者订阅，进行初始化
        subscriptions = new CopyOnWriteArrayList<>();
        // 把事件类型或对应订阅记录放入，形成<eventType, <subscriber, subscriberMethod>>
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            // 多次register(this)抛出此异常
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    // 获取同一事件上订阅记录的数量
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        // 根据方法注解设置的priority值，降序插入新记录事件记录
        // priority值越大越优先接收消息，默认值为0
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    // 通过订阅者身份获取其订阅的事件
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        // 同一个订阅类订阅的事件
        typesBySubscriber.put(subscriber, subscribedEvents);
    }

    // 向订阅者订阅的事件列表增加新事件类型
    subscribedEvents.add(eventType);

    // 订阅者方法的sticky为true，把历史事件发送给此订阅者
    if (subscriberMethod.sticky) {
        // eventInheritance为true，表示订阅subscriberMethod子类消息的订阅者也接收粘性事件
        if (eventInheritance) {
            // 消息类型所有子类的已存在粘性事件，都需要受到关注
            // 有非常多粘性事件时进行事件遍历是低效的，数据结构需在遍历上更高效
            // 例如：使用额外的map存储父类的子类：Class -> List<Class>
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                // 候选事件类型
                Class<?> candidateEventType = entry.getKey();
                // 检查eventType是否为candidateEventType的父类或同类
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            // 仅发送指定类型订阅事件
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

#### 4.3 checkPostStickyEventToSubscription

如果订阅者尝试终止该事件会失败，因为事件没有通过投递状态进行跟踪

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

#### 4.4 postToSubscription

__ThreadMode__ 中几种类别的主要含义在后续文章详解

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    // 获取订阅者方法指定线程的类别
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;

        case MAIN:
            if (isMainThread) {
                // 处于主线程就直接触发订阅者
                invokeSubscriber(subscription, event);
            } else {
                // 处在其他线程，向主线程Handler发送消息
                mainThreadPoster.enqueue(subscription, event);
            }
            break;

        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                // 优先放入主线程的消息队列等待处理
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;

        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;

        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;

        // 传入未知threadMode
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```


## 五、注销订阅

#### 5.1 unregister

从所有事件类中注销指定订阅者

```java
public synchronized void unregister(Object subscriber) {
    // 订阅者订阅的事件类型
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        // 逐个注销订阅者订阅的事件
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }

        // 从typesBySubscriber移除subscriber
        typesBySubscriber.remove(subscriber);
    } else {
        // 此订阅者之前未曾注册过，却进行注销操作
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

#### 5.2 unsubscribeByEventType

只更新 __subscriptionsByEventType__，而不是 __typesBySubscriber__，__typesBySubscriber__ 由调用者更新。

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    // 根据事件类型获取订阅记录
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        // 订阅记录数量
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            // 查找并移除所有相关订阅记录
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                // subscription设置为不活动
                subscription.active = false;
                // 从列表中移除
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

