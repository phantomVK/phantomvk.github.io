---
layout:     post
title:      "EventBus源码剖析(2) -- EventBusBuilder"
date:       2018-12-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

上期文章 [EventBus源码剖析(1) -- 注册与注销订阅](/2018/11/14/EventBus_1_Register/) 介绍了订阅者向 __EventBus__ 进行注册和注销的操作，并涉及部分 __EventBus__ 初始化逻辑。

在初始化逻辑中，很多功能都由本文将介绍的 __EventBusBuilder__ 提供。一般使用默认 __EventBusBuilder__ 对象，但 __EventBus__ 贴心地提供了定制的能力，以便满足不同需求。

## 一、类签名

通过自定义参数构造 __EventBus__ 实例，并能装载自定义的默认 __EventBus__ 实例。前文 [EventBus源码剖析(1) -- 注册与注销订阅_3.2 基础构造](/2018/11/14/EventBus_1_Register/#32-基础构造) 使用此实例，通过 __EventBus.builder()__ 创建新 __builder__。

```java
public class EventBusBuilder
```

## 二、数据成员

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

## 三、构造方法

默认构造方法

```java
EventBusBuilder() {
}
```

## 四、成员方法

设置 __logSubscriberExceptions__ 的参数值，默认为 true。表示是否记录订阅者抛出异常的日志。

```java
public EventBusBuilder logSubscriberExceptions(boolean logSubscriberExceptions) {
    this.logSubscriberExceptions = logSubscriberExceptions;
    return this;
}
```

设置 __logNoSubscriberMessages__ 的参数值，默认为true

```java
public EventBusBuilder logNoSubscriberMessages(boolean logNoSubscriberMessages) {
    this.logNoSubscriberMessages = logNoSubscriberMessages;
    return this;
}
```

设置 __sendSubscriberExceptionEvent__ 的参数值，默认为true

```java
public EventBusBuilder sendSubscriberExceptionEvent(boolean sendSubscriberExceptionEvent) {
    this.sendSubscriberExceptionEvent = sendSubscriberExceptionEvent;
    return this;
}
```

设置 __sendNoSubscriberEvent__ 的参数值，默认为true

```java
public EventBusBuilder sendNoSubscriberEvent(boolean sendNoSubscriberEvent) {
    this.sendNoSubscriberEvent = sendNoSubscriberEvent;
    return this;
}
```

设置 __throwSubscriberException__ 的参数值，当订阅者出现异常时不继续运行，默认为false。建议和 __BuildConfig.DEBUG__ 一起使用，开发期间不会错过任何抛出的异常。

```java
public EventBusBuilder throwSubscriberException(boolean throwSubscriberException) {
    this.throwSubscriberException = throwSubscriberException;
    return this;
}
```

设置 __eventInheritance__ 的参数值，默认为true。[EventBus源码剖析(1) -- 注册与注销订阅_4.2 subscribe](/2018/11/14/EventBus_1_Register/#42-subscribe) 小节后半部分代码注释，里面包含 __eventInheritance__ 的具体作用。默认情况下，__EventBus__ 考虑事件类的层级结构(父类的订阅者将收到通知)。

关闭此特性能提升事件投递性能，对一个直接继承 __Object__ 的简单事件类来说，事件投递性能有20%的提升。对更复杂的事件层级，提升幅度超过20%。

不过需记住的是，事件投送一般只消耗应用很小部分的处理器时间片，除非发送事件的频率每秒数以千计。

```java
public EventBusBuilder eventInheritance(boolean eventInheritance) {
    this.eventInheritance = eventInheritance;
    return this;
}
```

自定义线程池替换默认线程池实现，为事件分发服务提供异步后台线程池。一般没有必要修改。

```java
public EventBusBuilder executorService(ExecutorService executorService) {
    this.executorService = executorService;
    return this;
}
```

方法名验证，验证以onEvent开头的方法避免拼写错误。使用此方法能排除指定订阅者类的检查，并关闭对使用修饰符 __public__、__not static__、__nor abstract__ 方法的检查。

```java
public EventBusBuilder skipMethodVerificationFor(Class<?> clazz) {
    if (skipMethodVerificationForClasses == null) {
        skipMethodVerificationForClasses = new ArrayList<>();
    }
    // 存入的类不会受到检查
    skipMethodVerificationForClasses.add(clazz);
    return this;
}
```

设置 __ignoreGeneratedIndex__ 的参数值，默认为false。即使已经存在已生成的索引，也强制使用反射。

```java
public EventBusBuilder ignoreGeneratedIndex(boolean ignoreGeneratedIndex) {
    this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    return this;
}
```

设置 __strictMethodVerification__ 的参数值，默认为false。启用严格的方法验证。

```java
public EventBusBuilder strictMethodVerification(boolean strictMethodVerification) {
    this.strictMethodVerification = strictMethodVerification;
    return this;
}
```

添加由 __EventBus__ 注解处理器建立的索引

```java
public EventBusBuilder addIndex(SubscriberInfoIndex index) {
    if (subscriberInfoIndexes == null) {
        subscriberInfoIndexes = new ArrayList<>();
    }
    subscriberInfoIndexes.add(index);
    return this;
}
```

为 __EventBus__ 所有日志事件指定日志处理器，默认所有日志由 __android.util.Log__ 处理。

```java
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
        // 获取Android Looper
        Object looperOrNull = getAndroidMainLooperOrNull();
        return looperOrNull == null ? null :
                new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
    } else {
        return null;
    }
}
```

尝试获取Android主线程上的 __Looper__，__Looper用法的实现参考__：[Android源码系列(5) -- Looper](/2016/12/03/Android_Looper/)
```java
Object getAndroidMainLooperOrNull() {
    try {
        return Looper.getMainLooper();
    } catch (RuntimeException e) {
        // 不是Android应用或应用功能不正常，导致找不到MainLooper，返回null
        return null;
    }
}
```

装载由 __EventBus.getDefault()__ 返回的默认 __EventBus__ 实例。

```java
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

## 五、MainThreadSupport

获取“主”线程的接口。对 __Android__ 来说是UI线程，其他类型的工程则根据需要设定。

```java
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    // AndroidHandlerMainThreadSupport是内部类，实现外部接口MainThreadSupport
    class AndroidHandlerMainThreadSupport implements MainThreadSupport {

        private final Looper looper;
        
        // 指定Looper
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

