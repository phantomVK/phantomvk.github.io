---
layout:     post
title:      "EventBus源码剖析 -- 订阅记录"
subtitle:   ""
date:       2017-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---



from 1_postToSubscription



```java
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Subscription) {
            Subscription otherSubscription = (Subscription) other;
            return subscriber == otherSubscription.subscriber
                    && subscriberMethod.equals(otherSubscription.subscriberMethod);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode() {
        return subscriber.hashCode() + subscriberMethod.methodString.hashCode();
    }
}
```

#### invokeSubscriber

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

#### handleSubscriberException

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

上述代码传递给订阅者方法参数的数量只有一个，所以订阅者方法也只能用一个参数，就像以下实现：

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
fun onEventReceived(event: UserEvent) {
    val name = event.name
    val age = event.age
    Log.i(TAG, "Event received, Name: $name, Age: $age.")
}
```

如果实例中没有可执行方法，注册到 __EventBus__ 过程的类扫描会抛出异常：

```
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.phantomvk.playground/com.phantomvk.playground.MainActivity}: org.greenrobot.eventbus.EventBusException: Subscriber class com.phantomvk.playground.MainActivity and its super classes have no public methods with the @Subscribe annotation
```