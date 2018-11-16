---
layout:     post
title:      "EventBus源码剖析(3) -- ThreadMode"
date:       2018-11-15
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、简介
__EventBus__ 支持消息通过不同的线程模式发送，以此满足 __Android__ 不同应用场景的要求。

通过以下订阅者内的[订阅方法](/2018/11/14/EventBus_1_Register/#21-订阅者)可知，线程模式通过注解参数进行配置

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

__EventBus__ 支持的线程模式都保存在以下枚举类中。每个订阅者方法都定义一种线程模式，该模式决定 __EventBus__ 通过哪种线程方法调用订阅者方法。

```java
/**
 * Each subscriber method has a thread mode, which determines in which thread the method is to be called by EventBus.
 * EventBus takes care of threading independently from the posting thread.
 */
public enum ThreadMode
```

## 四、线程模式

枚举类中默认包含5种线程模式：

#### 4.1 POSTING

订阅者会直接在于发布线程相同的线程调用。本模式也是 __EventBus__ 默认线程模式。事件分发意味着减少性能开销，因为能完全避免线程切换。因此，对短时间就能完成的单个任务来说，如果不会影响主线程，则本模式就是首要推荐的模式。就会是说，使用此模式处理时间的时候需要在方法上迅速执行完毕并返回，避免阻塞发布线程，也就是一般所用的主线程。

```java
/**
 * Subscriber will be called directly in the same thread, which is posting the event. This is the default. Event delivery
 * implies the least overhead because it avoids thread switching completely. Thus this is the recommended mode for
 * simple tasks that are known to complete in a very short time without requiring the main thread. Event handlers
 * using this mode must return quickly to avoid blocking the posting thread, which may be the main thread.
 */
POSTING,
```

#### 4.2 MAIN

如果 __EventBus__ 正在 __Android__ 中使用，订阅者会在主线程中进行调用。如果消息发布者所在线程是主线程，则订阅者方法会直接调用，并阻塞发布线程。否则，事件会排队等待(非阻塞性的)分派。

由于接收者方法执行在主线程上，所以方法逻辑应轻巧，或在内部通过子线程执行重量级操作，避免阻塞主线程的运行。同理可知，如果接受方法后续还有更多事件需要在主线程上发出，那么这些事件也会受到该次阻塞的影响。

如果 __EventBus__ 所在环境不是 __Android__ ，则本模式的执行行为和 __POSTING__ 模式相同。

```java
/**
 * On Android, subscriber will be called in Android's main thread (UI thread). If the posting thread is
 * the main thread, subscriber methods will be called directly, blocking the posting thread. Otherwise the event
 * is queued for delivery (non-blocking). Subscribers using this mode must return quickly to avoid blocking the main thread.
 * If not on Android, behaves the same as {@link #POSTING}.
 */
MAIN,
```

#### 4.3 MAIN_ORDERED

在 __Android__ 中，订阅者方法也是在主线程中被调用。但和 __MAIN__ 不同的是，本模式的消息会先进入属于主线程的消息队列中排队等待执行，而不是直接阻塞主线程。

```java
/**
 * On Android, subscriber will be called in Android's main thread (UI thread). Different from {@link #MAIN},
 * the event will always be queued for delivery. This ensures that the post call is non-blocking.
 */
MAIN_ORDERED,
```

#### 4.4 BACKGROUND

在 __Android__ 中，订阅者会在后台线程调用。如果发布消息的线程不是主线程，则订阅者方法会直接在发布线程上调用。如果消息发布线程是主线程，__EventBus__ 会使用一个单一的后台线程，该线程会顺序分派所有事件。如果订阅则使用此种模式，就应该快速从方法中返回，避免执行阻塞此单一后台线程。

在非 __Android__ 直接使用一个后台线程，不区分主线程。

```java
/**
 * On Android, subscriber will be called in a background thread. If posting thread is not the main thread, subscriber methods
 * will be called directly in the posting thread. If the posting thread is the main thread, EventBus uses a single
 * background thread, that will deliver all its events sequentially. Subscribers using this mode should try to
 * return quickly to avoid blocking the background thread. If not on Android, always uses a background thread.
 */
BACKGROUND,
```

#### 4.5 ASYNC

订阅者会在一个单独的线程上调用。此线程独立于事件发布线程和主线程。使用这种模式事，发送事件不会等待订阅者方法。如果订阅者方法执行耗费事件，则可使用这个模式，例如进行网络访问。为了同时避免触发大量的长期运行的订阅者方法，__EventBus__ 使用了线程池，从已完成的异步订阅者通知中复用线程。

```java
/**
 * Subscriber will be called in a separate thread. This is always independent from the posting thread and the
 * main thread. Posting events never wait for subscriber methods using this mode. Subscriber methods should
 * use this mode if their execution might take some time, e.g. for network access. Avoid triggering a large number
 * of long running asynchronous subscriber methods at the same time to limit the number of concurrent threads. EventBus
 * uses a thread pool to efficiently reuse threads from completed asynchronous subscriber notifications.
 */
ASYNC
```

## 实现

结合具体的实现方法[EventBus.postToSubscription()](/2018/11/14/EventBus_1_Register/#44-posttosubscription)，具体逻辑就不难理解了。

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

