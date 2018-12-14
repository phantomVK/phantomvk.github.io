---
layout:     post
title:      "EventBus源码剖析(3) -- 线程模式"
date:       2018-12-03
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、简介

__EventBus__ 消息支持通过不同线程模式发送，以此满足 __Android__ 不同应用场景的需求。通过订阅者内的[订阅方法](/2018/11/14/EventBus_1_Register/#21-订阅者)可知，线程模式是通过注解参数进行配置

## 二、用法

注册类必须包含至少一个接收事件的方法

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
fun onEventReceived(event: UserEvent) {
    val name = event.name
    val age = event.age
    Log.i(TAG, "Event received, Name: $name, Age: $age.")
}
```

## 三、类介绍

__Subscribe__ 接口为修饰接收者的注解， __threadMode()__ 决定订阅者方法触发时所在的线程。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;
    ....
    ....
}
```

__EventBus__ 支持的线程模式都保存在枚举类中。每个订阅者方法默认都定义一种线程模式，该模式告诉 __EventBus__ 通过哪种线程去调用订阅者方法。

```java
public enum ThreadMode
```

## 四、线程模式

__ThreadMode__ 枚举类中默认包含5种线程模式：

![EventBus-Publish-Subscribe](/img/android/EventBus/EventBus_ThreadMode.png)

#### 4.1 POSTING

订阅者在发布线程相同的线程上被调用，本模式是 __EventBus__ 默认线程模式。因为完全避免线程切换，此线程模式的事件分发意味着能减少性能开销。

因此，对已知能在短时间内、主线程上完成的单个任务，本模式是首要推荐的。就是说此模式处理事件时，方法需要迅速执行完毕，并返回方法避免阻塞(主线程)发布线程。

```java
POSTING,
```

#### 4.2 MAIN

如果 __EventBus__ 正在 __Android__ 中使用，订阅者会在主线程中进行调用。如果消息发布线程就是主线程，则订阅者方法直接调用，并阻塞发布线程。否则，事件在消息队列等待主线程(非阻塞性的)分派。

由于接收者方法执行在主线程上，所以方法逻辑应轻巧，或在内部开启子线程执行重量级操作，以此避免阻塞主线程运行。推理可知，如果订阅者方法后续还有更多事件需要在主线程上发出，那么这些事件也会受到该次阻塞的影响。

如果 __EventBus__ 所在环境不是 __Android__ ，则本模式的执行行为和 __POSTING__ 模式相同。

```java
MAIN,
```

#### 4.3 MAIN_ORDERED

在 __Android__ 中订阅者方法也是在主线程中被调用。但和 __MAIN__ 模式不同的，是本模式的消息会直接进入属于主线程的消息队列中排队等待执行，而不是阻塞主线程直接运行。

```java
MAIN_ORDERED,
```

#### 4.4 BACKGROUND

在 __Android__ 中，此模式的订阅者会在后台线程调用。如果发布消息的线程不是主线程，则订阅者方法直接在发布线程上调用。如果消息发布线程是主线程，__EventBus__ 会使用单个后台线程，该线程会顺序分派所有事件。如果订阅者使用此种模式，就应该快速从方法中返回，避免阻塞这个单一后台线程的运行。在非 __Android__ 直接使用一个后台线程，不区分主线程。

```java
BACKGROUND,
```

#### 4.5 ASYNC

订阅者会在分离的线程上调用。此线程独立于事件发布线程和主线程。使用这种模式时，发送事件不会等待订阅者方法。可使用这个模式处理订阅者方法的耗时事件，例如：进行网络访问。为了避免同时触发大量、长期运行的订阅者方法，__EventBus__ 使用了线程池，从已完成的异步订阅者通知复用线程。

```java
ASYNC
```

## 实现

前文提到实现方法 [EventBus.postToSubscription()](/2018/11/14/EventBus_1_Register/#44-posttosubscription) 的逻辑就不难理解了。

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

        // 传入未知threadMode，引起异常
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

__invokeSubscriber__ 直接在消息发送者所在线程上调用目标方法。

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        // subscription.subscriber：通过实例调用方法
        // event：调用时传递给方法的参数
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        // 捕获订阅者方法运行时异常，避免导致EventBus崩溃
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```
