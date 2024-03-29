---
layout:     post
title:      "EventBus源码剖析(3) -- 线程模式"
date:       2018-12-03
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

__EventBus__ 消息支持通过不同线程模式发送并调起目标方法，以满足 __Android__ 不同应用场景。通过订阅者的[订阅方法](/2018/11/14/EventBus_1_Register/#21-订阅者)可知，线程模式通过注解参数进行配置。

## 二、用法

注册类必须包含至少一个接收事件的方法，如果不满足此条件，订阅类注册到 __EventBus__ 时会抛出异常。若没有设置 __threadMode__ ，订阅方法默认在主线程调用。

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
fun onEventReceived(event: UserEvent) {
    val name = event.name
    val age = event.age
    Log.i(TAG, "Event received, Name: $name, Age: $age.")
}
```

## 三、类介绍

__Subscribe__ 接口是用于修饰订阅方法的注解。

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

__EventBus__ 支持的线程模式都保存在枚举类中。每个订阅者方法含有默认线程模式，该模式决定如何调用订阅者方法。

```java
public enum ThreadMode
```

## 四、线程模式

__ThreadMode__ 枚举类包含5种线程模式：

![EventBus-Publish-Subscribe](/img/android/EventBus/EventBus_ThreadMode.png)

#### 4.1 POSTING

订阅方法在发布线程上调起，这是 __EventBus__ 的默认线程模式。因为能完全避免线程切换，意味着此线程模式的事件分发能减少性能开销。所以可在短时间、主线程完成的任务，本模式是首要推荐的。就是说此模式处理事件时，方法需要迅速执行完毕，并返回方法避免阻塞(主线程)发布线程。

#### 4.2 MAIN

如果 __EventBus__ 在 __Android__ 中使用，订阅者会在主线程中进行调用。如果消息发布线程就是主线程，则订阅方法直接调用，并阻塞发布线程。否则，事件在消息队列等待主线程(非阻塞性的)分派。

由于接收者方法执行在主线程上，所以方法逻辑应轻巧，或在内部开启子线程执行重量级操作，以此避免阻塞主线程运行。推理可知，如果订阅方法后续还有更多事件需要在主线程上发出，那么这些事件也会受到该次阻塞的影响。

如果 __EventBus__ 所在环境不是 __Android__ ，则本模式的执行行为和 __POSTING__ 模式相同。

#### 4.3 MAIN_ORDERED

在 __Android__ 中订阅者方法也是在主线程中被调用。但和 __MAIN__ 模式不同的，是本模式的消息进入属于主线程的消息队列等待执行，而不是阻塞主线程直接运行。

#### 4.4 BACKGROUND

在 __Android__ 中，此模式的订阅者会在后台线程调用。如果发布消息的线程不是主线程，则订阅者方法直接在发布线程上调用。如果消息发布线程是主线程，__EventBus__ 会使用单个后台线程，该线程会顺序分派所有事件。如果订阅者使用此种模式，就应该快速从方法中返回，避免阻塞这个单一后台线程的运行。在非 __Android__ 直接使用一个后台线程，不区分主线程。

#### 4.5 ASYNC

订阅者会在分离的线程上调用。此线程独立于事件发布线程和主线程。使用这种模式时，发送事件不会等待订阅者方法。可使用这个模式处理订阅者方法的耗时事件，例如：进行网络访问。为了避免同时触发大量、长期运行的订阅者方法，__EventBus__ 使用了线程池，从已完成的异步订阅者通知复用线程。

## 实现

上文提到实现方法 [EventBus.postToSubscription()](/2018/11/14/EventBus_1_Register/#44-posttosubscription) 的逻辑就不难理解了。

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

        // 传入未知threadMode引起异常
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

在消息发送者所在线程上调用目标方法。

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        // subscription.subscriber：订阅方法所在实例
        // event：调用时传递给方法的参数，即方法接收的消息
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        // 捕获订阅者方法运行时异常，避免EventBus崩溃
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```
