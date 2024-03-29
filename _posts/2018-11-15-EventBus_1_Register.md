---
layout:     post
title:      "EventBus源码剖析(1) -- 订阅注册与注销"
date:       2018-11-15
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus源码剖析
---

文章列表：
- [EventBus源码剖析(1) -- 订阅注册与注销](/2018/11/15/EventBus_1_Register/)
- [EventBus源码剖析(2) -- EventBusBuilder](/2018/12/01/EventBus_2_EventBusBuilder/)
- [EventBus源码剖析(3) -- 线程模式](/2018/12/03/EventBus_3_ThreadMode/)
- [EventBus源码剖析(4) -- 订阅记录](/2018/12/06/EventBus_4_Subscription/)
- [EventBus源码剖析(5) -- Poster](/2018/12/10/EventBus_5_Poster/)

## 一、简介

#### 1.1 特性

__EventBus__ 是为 __Android__ 而设的 __publish/subscribe (发布/订阅)__ 消息系统。事件通过 __post(Object)__ 提交到总线，总线把事件投递给订阅者。而该订阅者，需包含匹配类型消息的处理方法。

![EventBus-Publish-Subscribe](/img/android/EventBus/EventBus-Publish-Subscribe.png)

为了能接收事件，订阅者需通过 __register(Object)__ 把自己注册到事件总线上。一旦成功注册，订阅者可以持续接收关心的事件，直至通过 __unregister(Object)__ 结束订阅。

处理事件的方法需满足以下条件：

- 用 __@Subscribe__ 进行注解；
- 方法可见性为 __public__；
- 方法返回值为 __void__；
- 仅含有一个参数，且形参为事件类；

由于把 __Subscription__ 翻译为名词性的“订阅”后，和字面动词性“订阅”没法区分。所以本系列文章把该词翻译为更贴切的名词：“订阅记录”，这个词会将在后续文章继续沿用。

#### 1.2 优点

* 简化不同组件间通讯
   *  对事件发送者和接收者进行解耦
   *  与Activities、 Fragments、和后台线程运行良好
   *  避免复杂、易错的依赖和生命周期问题
* 令实现代码更简洁
* 运行速度快
* 库体积小 (约50KB)
* 已经过累计 100,000,000+ 安装量的应用验证
* 包含消息分发线程、订阅者优先级等高级特性

官方文档没有提及，__EventBus__ 在进程(VM)创建单例，所有消息送到同进程的消息中心，由消息中心分发给同进程的实例。运行在不同进程的页面，没法收到来自其他进程的消息。要实现跨进程消息分发，必须使用 __Socket__、__Binder__ 等方法。

#### 1.3 版本

__EventBus__ 自17年年底开始，基本停止代码提交，可认为 __EventBus__ 功能稳定无变动。处于这种状况的开源库非常合适进行源码剖析，而本篇文章基于当时最新正式版 __3.1.1__ 开展。

## 二、用法

#### 2.1 订阅者

订阅者要在适当的生命周期时，把自己注册到消息总线。由于事件的基本接收单位是方法，所以需要给接收事件的方法添加注解，以便 __EventBus__ 通过注解发现该方法。

接收者方法需要遵循以下规则：

-  使用 __EventBus__ 的注解修饰方法；
-  方法不能为 __private__，才能让 __EventBus__ 获取该方法；
-  方法必须只有一个参数，且参数类型就是所关心事件的类型；

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        EventBus.getDefault().register(this)
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

每个接收者类实例只需向 __EventBus__ 注册一次。当接收者类对事件不再关心时，也需要在合适时间点注销订阅。

#### 2.2 发布者

对事件发布者来说事情就简单多了，构建目标事件，把数据或负载内容封装在事件内发出即可。

```kotlin
fun postEvent() {
    val user = UserEvent("Mike", 24)
    EventBus.getDefault().post(user)
}
```

#### 2.3 事件消息体

这是示例的消息体，消息体包含用户的名字和年龄

```kotlin
class UserEvent(val name: String, val age: Int)
```

如果消息只是简单通知，事件类甚至可以不含任何数据成员。例如：

```kotlin
class Notification
```

## 三、初始化

#### 3.1 单例

整个 __EventBus__ 通过以下方法创建单例。所有在此单例发送的事件，只对注册在单例里的订阅者有效。为保证所有事件能在同一个 __EventBus__ 内流动，一定要从此方法获取 __EventBus__ 实例。

```java
public class EventBus {
    // 变量使用volatile修饰
    static volatile EventBus defaultInstance;

    // 同一进程内有效，双重检验锁实现单例
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

调用 __getDefault()__ 方法时，同类两个常量也获得初始化。

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

事件类型缓存，实例为 __HashMap<Class\<?>, List<Class<?\>>>__

```java
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();
```

#### 3.2 基础构造

单例的初始化调用此构造方法，然后方法内又调用另一个构造方法：

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

    // 注解处理器所生成订阅者信息的数量
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;

    // 初始化订阅者方法查找器初始化
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

__PostingThreadState__ 是变量的封装，放在 __ThreadLocal__ 中，以便快速设置、获取当前线程状态的多个变量。

```java
final static class PostingThreadState {
    // 当前线程的事件队列
    final List<Object> eventQueue = new ArrayList<>();

    // 消息是否在投递中
    boolean isPosting;

    // 是否为主线程
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

所有订阅者通过此方法向 __EventBus__ 注册，以便在注册后收取所关心的事件。注销订阅则通过方法 __unregister(Object)__，这样观察者可在不关心事件的时候取消订阅。

从下面的实现大概可知，如果订阅者在生命周期结束后不移除订阅者，页面会出现内存泄漏。

```java
public void register(Object subscriber) {
    // 获取订阅者Class
    Class<?> subscriberClass = subscriber.getClass();

    // 根据订阅者类型获取接收事件的方法
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

此方法在同步块中调用，把订阅方法按照订阅事件分类。过程会检查订阅者是否出现重复注册问题，后根据设置决定是否分发粘性事件。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 订阅者方法的事件类型，即订阅者方法唯一参数
    Class<?> eventType = subscriberMethod.eventType;

    // 构建新记录
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

    // 用事件类型获取所有订阅记录列表
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);

    if (subscriptions == null) {
        // 订阅记录列表为空，进行初始化
        subscriptions = new CopyOnWriteArrayList<>();
        // 保存事件类型和对应订阅记录
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        // 检查该订阅者是否出现重复注册
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    // 获取同一事件的订阅记录数量
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        // 根据方法注解设置的priority值降序排序，并插入新记录事件记录
        // priority值越大越优先接收消息，默认值为0
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    // 通过订阅者类型获取，该订阅者已订阅事件类型
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    // 如果订阅者首次注册则该列表为空，并存入刚订阅的事件
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        // 同一订阅者订阅的事件
        typesBySubscriber.put(subscriber, subscribedEvents);
    }

    // 向订阅者订阅事件列表增加新事件类型
    subscribedEvents.add(eventType);

    // 订阅方法的sticky为true，把历史粘性事件发送给新订阅方法
    if (subscriberMethod.sticky) {
        // eventInheritance为true，表示订阅subscriberMethod子类消息的订阅者也接收粘性事件
        if (eventInheritance) {
            // 消息类型所有子类的已存在粘性事件，都需要受到关注
            // 有非常多粘性事件时进行事件遍历是低效的，数据结构需在遍历上变得更高效
            // 例如：使用额外的map存储父类的子类：Class -> List<Class>
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                // 候选事件类型
                Class<?> candidateEventType = entry.getKey();
                // 检查eventType是否为candidateEventType的父类或同类
                if (eventType.isAssignableFrom(candidateEventType)) {
                    // 把子类事件发送给注册为父类事件的订阅者
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            // 仅发送类型完全匹配的订阅事件
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

#### 4.3 checkPostStickyEventToSubscription

投递粘性事件给订阅者。如果订阅者尝试终止该事件会失败，因为事件没法通过投递状态进行跟踪。

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    // 检查粘性事件是否为null
    if (stickyEvent != null) {
        // 把粘性事件作为普通事件发送给订阅者
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
                // 当前线程是主线程，把事件放入后台线程处理队列
                backgroundPoster.enqueue(subscription, event);
            } else {
                // 现在是后台线程，直接调用订阅者方法
                invokeSubscriber(subscription, event);
            }
            break;

        case ASYNC:
            // 放入异步队列处理
            asyncPoster.enqueue(subscription, event);
            break;

        default:
            // 传入未知threadMode
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

上述注册流程简化为如下流程图：

![register](/img/android/EventBus/EventBus_register.png)

## 五、注销订阅

#### 5.1 unregister

把指定订阅者从消息总线中注销

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

只更新 __subscriptionsByEventType__，而不是 __typesBySubscriber__，因为 __typesBySubscriber__ 由调用者更新。

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

上述注销流程简化为如下流程图：

![register](/img/android/EventBus/EventBus_unregister.png)

下一章将继续介绍 __EventBus__ 初始化过程中，__EventBusBuilder__ 的作用。