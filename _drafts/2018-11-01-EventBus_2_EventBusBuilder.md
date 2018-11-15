---
layout:     post
title:      "EventBus源码剖析(2) -- EventBusBuilder"
date:       2018-11-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

上篇文章 [EventBus源码剖析(1) — 注册与注销订阅](/2018/11/14/EventBus_1_Register/) 详细介绍了订阅者如何向 __EventBus__ 进行注册和注销的操作，并设计到 __EventBus__ 部分初始化的逻辑。在初始化逻辑中，很多功能都是本文将介绍的 __EventBusBuilder__ 提供。一般来使用的就是默认的 __EventBusBuilder__ 对象，但 __EventBus__ 贴心提供了自定制的能力，以满足不同需求。

## 类签名

通过自定义参数构造 __EventBus__ 实例，并允许装载自定义的默认 __EventBus__ 实例。前文 [EventBus源码剖析(1) — 注册与注销订阅 - 3.2 基础构造](/2018/11/14/EventBus_1_Register/#32-基础构造) 使用此实例。

```java
/**
 * Creates EventBus instances with custom parameters and also allows to install a custom default EventBus instance.
 * Create a new builder using {@link EventBus#builder()}.
 */
public class EventBusBuilder
```

## 数据成员

```java
// 构造线程池为newCachedThreadPool，所有实例共享同一个静态线程池
private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

boolean logSubscriberExceptions = true;
boolean logNoSubscriberMessages = true;
boolean sendSubscriberExceptionEvent = true;
boolean sendNoSubscriberEvent = true;
boolean throwSubscriberException;
boolean eventInheritance = true;
boolean ignoreGeneratedIndex;
boolean strictMethodVerification;

// 获取静态实例的线程池
ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
List<Class<?>> skipMethodVerificationForClasses;
List<SubscriberInfoIndex> subscriberInfoIndexes;
Logger logger;
MainThreadSupport mainThreadSupport;
```

## 构造方法

默认构造方法

```java
EventBusBuilder() {
}
```

## 成员方法

设置 __logSubscriberExceptions__ 的参数值，默认为 true

```java
/** Default: true */
public EventBusBuilder logSubscriberExceptions(boolean logSubscriberExceptions) {
    this.logSubscriberExceptions = logSubscriberExceptions;
    return this;
}
```

设置 __logNoSubscriberMessages__ 的参数值，默认为true

```java
/** Default: true */
public EventBusBuilder logNoSubscriberMessages(boolean logNoSubscriberMessages) {
    this.logNoSubscriberMessages = logNoSubscriberMessages;
    return this;
}
```

设置 __sendSubscriberExceptionEvent__ 的参数值，默认为true

```java
/** Default: true */
public EventBusBuilder sendSubscriberExceptionEvent(boolean sendSubscriberExceptionEvent) {
    this.sendSubscriberExceptionEvent = sendSubscriberExceptionEvent;
    return this;
}
```

设置 __sendNoSubscriberEvent__ 的参数值，默认为true

```java
/** Default: true */
public EventBusBuilder sendNoSubscriberEvent(boolean sendNoSubscriberEvent) {
    this.sendNoSubscriberEvent = sendNoSubscriberEvent;
    return this;
}
```

设置 __throwSubscriberException__ 的参数值，默认为false。建议和 __BuildConfig.DEBUG__ 一起使用，以便在开发过程中通过抛出的异常了解具体错误。

```java
/**
 * Fails if an subscriber throws an exception (default: false).
 * <p/>
 * Tip: Use this with BuildConfig.DEBUG to let the app crash in DEBUG mode (only). This way, you won't miss
 * exceptions during development.
 */
public EventBusBuilder throwSubscriberException(boolean throwSubscriberException) {
    this.throwSubscriberException = throwSubscriberException;
    return this;
}
```

设置 __eventInheritance__ 的参数值，默认为true。[EventBus源码剖析(1) — 注册与注销订阅_4.2 subscribe](/2018/11/14/EventBus_1_Register/#42-subscribe)小结后半部分代码注释，其中包含 __eventInheritance__ 的具体作用。

```java
/**
 * By default, EventBus considers the event class hierarchy (subscribers to super classes will be notified).
 * Switching this feature off will improve posting of events. For simple event classes extending Object directly,
 * we measured a speed up of 20% for event posting. For more complex event hierarchies, the speed up should be
 * >20%.
 * <p/>
 * However, keep in mind that event posting usually consumes just a small proportion of CPU time inside an app,
 * unless it is posting at high rates, e.g. hundreds/thousands of events per second.
 */
public EventBusBuilder eventInheritance(boolean eventInheritance) {
    this.eventInheritance = eventInheritance;
    return this;
}

```

自定义线程池替换默认线程池实现。

```java
/**
 * Provide a custom thread pool to EventBus used for async and background event delivery. This is an advanced
 * setting to that can break things: ensure the given ExecutorService won't get stuck to avoid undefined behavior.
 */
public EventBusBuilder executorService(ExecutorService executorService) {
    this.executorService = executorService;
    return this;
}
```

```java
/**
 * Method name verification is done for methods starting with onEvent to avoid typos; using this method you can
 * exclude subscriber classes from this check. Also disables checks for method modifiers (public, not static nor
 * abstract).
 */
public EventBusBuilder skipMethodVerificationFor(Class<?> clazz) {
    if (skipMethodVerificationForClasses == null) {
        skipMethodVerificationForClasses = new ArrayList<>();
    }
    skipMethodVerificationForClasses.add(clazz);
    return this;
}
```

设置 __ignoreGeneratedIndex__ 的参数值，默认为false

```java
/** Forces the use of reflection even if there's a generated index (default: false). */
public EventBusBuilder ignoreGeneratedIndex(boolean ignoreGeneratedIndex) {
    this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    return this;
}
```

设置 __strictMethodVerification__ 的参数值，默认为false

```java
/** Enables strict method verification (default: false). */
public EventBusBuilder strictMethodVerification(boolean strictMethodVerification) {
    this.strictMethodVerification = strictMethodVerification;
    return this;
}
```

```java
/** Adds an index generated by EventBus' annotation preprocessor. */
public EventBusBuilder addIndex(SubscriberInfoIndex index) {
    if (subscriberInfoIndexes == null) {
        subscriberInfoIndexes = new ArrayList<>();
    }
    subscriberInfoIndexes.add(index);
    return this;
}
```

为 __EventBus__ 所有日志事件指定日志处理器

```java
/**
 * Set a specific log handler for all EventBus logging.
 * <p/>
 * By default all logging is via {@link android.util.Log} but if you want to use EventBus
 * outside the Android environment then you will need to provide another log target.
 */
public EventBusBuilder logger(Logger logger) {
    this.logger = logger;
    return this;
}
```

获取 __Logger__

```java
Logger getLogger() {
    if (logger != null) {
        return logger;
    } else {
        // also check main looper to see if we have "good" Android classes (not Stubs etc.)
        return Logger.AndroidLogger.isAndroidLogAvailable() && getAndroidMainLooperOrNull() != null
                ? new Logger.AndroidLogger("EventBus") :
                new Logger.SystemOutLogger();
    }
}
```

获取 __MainThreadSupport__

```java
MainThreadSupport getMainThreadSupport() {
    if (mainThreadSupport != null) {
        return mainThreadSupport;
    } else if (Logger.AndroidLogger.isAndroidLogAvailable()) {
        Object looperOrNull = getAndroidMainLooperOrNull();
        return looperOrNull == null ? null :
                new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
    } else {
        return null;
    }
}
```

尝试获取Android主线程上的Loopep，传送门：[Android源码系列(5) -- Looper](/2016/12/03/Android_Looper/)
```java
Object getAndroidMainLooperOrNull() {
    try {
        return Looper.getMainLooper();
    } catch (RuntimeException e) {
        // 不是Android应用或应用功能不正常，导致找不到MainLooper
        return null;
    }
}
```

装载默认 __EventBus__ 实例

```java
/**
 * Installs the default EventBus returned by {@link EventBus#getDefault()} using this builders' values. Must be
 * done only once before the first usage of the default EventBus.
 *
 * @throws EventBusException if there's already a default EventBus instance in place
 */
public EventBus installDefaultEventBus() {
    synchronized (EventBus.class) {
        if (EventBus.defaultInstance != null) {
            throw new EventBusException("Default instance already exists." +
                    " It may be only set once before it's used the first time to ensure consistent behavior.");
        }
        EventBus.defaultInstance = build();
        return EventBus.defaultInstance;
    }
}
```

根据当前定义配置构建 __EventBus__ 实例

```java
public EventBus build() {
    return new EventBus(this);
}
```

## MainThreadSupport

```java
/**
 * Interface to the "main" thread, which can be whatever you like. Typically on Android, Android's main thread is used.
 */
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    // AndroidHandlerMainThreadSupport是内部类，实现外部接口MainThreadSupport
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

