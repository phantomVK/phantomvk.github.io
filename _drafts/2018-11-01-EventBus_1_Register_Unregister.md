---
layout:     post
title:      "EventBus源码解析 -- 一、注册订阅与注销订阅"
date:       2017-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus源码
---

## 一、简介

#### 1.1 特性

__EventBus__ 是为 __Android__ 而设的中心化 __publish/subscribe (发布者/订阅者)__ 事件系统。消息通过 __post(Object)__ 把消息提交到总线，总线把消息分发给订阅者，而该订阅者需拥有匹配消息类型的处理方法。

![EventBus-Publish-Subscribe](/img/android/EventBus/EventBus-Publish-Subscribe.png)

为能接收消息，订阅者需要通过 __register(Object)__ 把自己注册到总线上。一旦成功注册，订阅者将一直接收对应消息，直到订阅者通过 __unregister(Object)__ 注销监听。处理消息的方法一定需要标注为 __Subscribe__ 注解，且被注解方法为 __Subscribe__、方法返回值为 __void__，且必须只含有一个变量，用于接收来自总线的消息。


```java
/**
 * EventBus is a central publish/subscribe event system for Android. Events are posted ({@link #post(Object)}) to the
 * bus, which delivers it to subscribers that have a matching handler method for the event type. To receive events,
 * subscribers must register themselves to the bus using {@link #register(Object)}. Once registered, subscribers
 * receive events until {@link #unregister(Object)} is called. Event handling methods must be annotated by
 * {@link Subscribe}, must be c, return nothing (void), and have exactly one parameter
 * (the event).
 *
 * @author Markus Junginger, greenrobot
 */
```

#### 1.2 优点

* 简化不同组件间的通讯
   *  对事件发送者和接收者两者进行解耦
   *  与Activities、 Fragments、和 background threads 运行得很好
   *  避免复杂、易错的依赖和生命周期问题
* 令实现代码更简洁
* 运行速度快
* 库体积小 (约50KB)
* 已经过累计 100,000,000+ 安装量的应用验证
* 有消息分发线程、订阅者优先级等的高级特性

## 二、用法

#### 2.1 订阅者

订阅者需要在合适的生命周期把自己注册到消息总线上以便接收关心的事件。同时，由于事件的基础接收单位是方法，所以需要把接收事件的方法添加注解，以便 __EventBus__ 把事件发送到该方法上。

接收者方法需要遵循以下规则：

-  使用 __EventBus__ 的注册修饰方法；
- 方法不能为 __private__，才能让  __EventBus__ 获取该方法；
- 方法必须只有一个参数，且参数类型就是所关心事件的类型；

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (EventBus.getDefault().isRegistered(this)) {
            EventBus.getDefault().register(this)
        }
    }
    
    @Subscribe(threadMode = ThreadMode.MAIN)
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

除了把实例注册到 __EventBus__ ，如果接收者类对事件不再关心，也需要在合适时间点移除注册。每个接收者类只需向 __EventBus__ 注册一次。为避免多次注册，可以向上述代码一样，在注册前检查是否已注册。

#### 2.2 发布者

对事件发布者来说，工作就不叫简单了。只需要构建目标事件，把数据或负载内容构建到事件中发出即可。如果事件只是为了发出通知，事件实现类可以不带任何数据成员。

```java
fun postEvent() {
    val user =UserEvent("Mike", 24)
    EventBus.getDefault().post(user)
}
```

#### 2.3 事件消息体

这是示例的消息体，消息体重包含用户的名字和年龄。

```java
class UserEvent(val name: String, val age: Int)
```

如果消息只是为了发出简单通知，事件消息体可以不含任何数据成员。例如在 __Kotlin__ 中：

```java
class Notification
```

## 三、初始化

#### 3.1 单例

```java
public class EventBus {

    static volatile EventBus defaultInstance;

    /** Convenience singleton for apps using a process-wide EventBus instance. */
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

#### 3.2 基础构造

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();

/**
 * Creates a new EventBus instance; each instance is a separate scope in which events are delivered. To use a
 * central bus, consider {@link #getDefault()}.
 */
public EventBus() {
    this(DEFAULT_BUILDER);
}
```

#### 3.3 深入构造

构造过程对以下数据成员进行赋值

```java
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
private final Map<Object, List<Class<?>>> typesBySubscriber;
private final Map<Class<?>, Object> stickyEvents;

// @Nullable
private final MainThreadSupport mainThreadSupport;
// @Nullable
private final Poster mainThreadPoster;
private final BackgroundPoster backgroundPoster;
private final AsyncPoster asyncPoster;
private final SubscriberMethodFinder subscriberMethodFinder;
private final ExecutorService executorService;

private final boolean throwSubscriberException;
private final boolean logSubscriberExceptions;
private final boolean logNoSubscriberMessages;
private final boolean sendSubscriberExceptionEvent;
private final boolean sendNoSubscriberEvent;
private final boolean eventInheritance;

private final int indexCount;
private final Logger logger;
```

```java
EventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    subscriptionsByEventType = new HashMap<>();
    typesBySubscriber = new HashMap<>();
    stickyEvents = new ConcurrentHashMap<>();
    mainThreadSupport = builder.getMainThreadSupport();
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);
    indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
            builder.strictMethodVerification, builder.ignoreGeneratedIndex);
    logSubscriberExceptions = builder.logSubscriberExceptions;
    logNoSubscriberMessages = builder.logNoSubscriberMessages;
    sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
    sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
    throwSubscriberException = builder.throwSubscriberException;
    eventInheritance = builder.eventInheritance;
    executorService = builder.executorService;
}
```

#### 3.4 PostingThreadState

```java
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};

/** For ThreadLocal, much faster to set (and get multiple values). */
final static class PostingThreadState {
    final List<Object> eventQueue = new ArrayList<>();
    boolean isPosting;
    boolean isMainThread;
    Subscription subscription;
    Object event;
    boolean canceled;
}
```

## 四、注册事件

#### 4.1 register

```java
/**
 * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
 * are no longer interested in receiving events.
 * <p/>
 * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
 * The {@link Subscribe} annotation also allows configuration like {@link
 * ThreadMode} and priority.
 */
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

#### 4.2 subscribe

```java
// 此方法必须在同步块中调用
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);

    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

#### 4.3 checkPostStickyEventToSubscription

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
        // --> Strange corner case, which we don't take care of here.
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

#### 4.4 postToSubscription

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
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
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

#### 4.5 invokeSubscriber

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

#### 4.6 handleSubscriberException

```java
private void handleSubscriberException(Subscription subscription, Object event, Throwable cause) {
    if (event instanceof SubscriberExceptionEvent) {
        if (logSubscriberExceptions) {
            // Don't send another SubscriberExceptionEvent to avoid infinite event recursion, just log
            logger.log(Level.SEVERE, "SubscriberExceptionEvent subscriber " + subscription.subscriber.getClass()
                    + " threw an exception", cause);
            SubscriberExceptionEvent exEvent = (SubscriberExceptionEvent) event;
            logger.log(Level.SEVERE, "Initial event " + exEvent.causingEvent + " caused exception in "
                    + exEvent.causingSubscriber, exEvent.throwable);
        }
    } else {
        if (throwSubscriberException) {
            throw new EventBusException("Invoking subscriber failed", cause);
        }
        if (logSubscriberExceptions) {
            logger.log(Level.SEVERE, "Could not dispatch event: " + event.getClass() + " to subscribing class "
                    + subscription.subscriber.getClass(), cause);
        }
        if (sendSubscriberExceptionEvent) {
            SubscriberExceptionEvent exEvent = new SubscriberExceptionEvent(this, cause, event,
                    subscription.subscriber);
            post(exEvent);
        }
    }
}
```

## 五、注销订阅

#### 5.1 unregister

```java
/** Unregisters the given subscriber from all event classes. */
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

#### 5.2 unsubscribeByEventType

```java
/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

