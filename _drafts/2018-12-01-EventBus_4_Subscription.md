---
layout:     post
title:      "EventBus源码剖析(4) -- 订阅记录"
date:       2018-11-19
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

## 一、Subscription

当订阅者向 __EventBus__ 注册时， __EventBus__ 会扫描整个订阅者类，获取具体接收事件的方法，并构造出以下实例。每个订阅者可能有多个方法接收订阅事件，每个方法均会生成各自的 __Subscription__，作为接受事件的凭证。

由于把 __Subscription__ 翻译为名词性的“订阅”中文后，和字面上动词性的“订阅”没法区分。所以本系列文章，把该词翻译为更具贴切的名词“订阅记录”。这个词会将在后续文章继续沿用。

订阅记录内主要包括3个成员变量。`subscriber`表示订阅者类的实例，`subscriberMethod`表示订阅者内部接受事件的方法，和表示订阅记录是否存活的`active`。

调用 __EventBus#unregister(Object)__ 注销订阅者后，`active`立即改为 __false__。该值被负责队列事件投递的 __EventBus#invokeSubscriber(PendingPost)__ 检查以避免 __race conditions__。

```java
final class Subscription {
    // 订阅者类的实例
    final Object subscriber;
    
    // 订阅事件的方法
    final SubscriberMethod subscriberMethod;

    // 订阅记录是否依然生效标志位，volatile修饰控制多线程可见性
    volatile boolean active;

    // 构造方法，构造之后默认接受事件
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

## 二、SubscriberMethod

具体到 __Subscription__ 内部实现，里面还包含了 __SubscriberMethod__。

前文已经提到，订阅者类通过 __EventBus__ 的注解修饰并因此能被 __EventBus__ 发现。__EventBus__ 通过注解处理器分析注解，和获取的订阅者方法信息构造成这个 __SubscriberMethod__ 类，成为订阅者信息的索引。

```java
public class SubscriberMethod {
    // 接收事件的方法实例
    final Method method;
    
    // 线程模式
    final ThreadMode threadMode;
    
    // 事件类型
    final Class<?> eventType;
    
    // 事件优先级
    final int priority;
    
    // 是否粘性事件
    final boolean sticky;
    
    // 此字符串用于提高比较效率
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }
    
    // 比较两个订阅者方法
    @Override
    public boolean equals(Object other) {
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString has more overhead, just take relevant parts of the method
            StringBuilder builder = new StringBuilder(64);
            builder.append(method.getDeclaringClass().getName());
            builder.append('#').append(method.getName());
            builder.append('(').append(eventType.getName());
            methodString = builder.toString();
        }
    }

    @Override
    public int hashCode() {
        return method.hashCode();
    }
}
```

## 三、Subscribe注解

这个就是 __EventBus__ 注解。从注解类可以看到 __threadMode__、__sticky__、__priority__ 均能和 __SubscriberMethod__ 类的数据成员匹配上。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;

    /**
     * If true, delivers the most recent sticky event (posted with
     * {@link EventBus#postSticky(Object)}) to this subscriber (if event available).
     */
    // 是否是粘性事件
    boolean sticky() default false;

    /** Subscriber priority to influence the order of event delivery.
     * Within the same delivery thread ({@link ThreadMode}), higher priority subscribers will receive events before
     * others with a lower priority. The default priority is 0. Note: the priority does *NOT* affect the order of
     * delivery among subscribers with different {@link ThreadMode}s! */
    // 方法接收事件的优先级
    int priority() default 0;
}
```

实例：

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
fun onEventReceived(event: UserEvent) {
    val name = event.name
    val age = event.age
    Log.i(TAG, "Event received, Name: $name, Age: $age.")
}
```

通过特意构造的实例可知：如果实例没有可执行方法，注册到 __EventBus__ 过程的类扫描会抛出异常：

```
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.phantomvk.playground/com.phantomvk.playground.MainActivity}: org.greenrobot.eventbus.EventBusException: Subscriber class com.phantomvk.playground.MainActivity and its super classes have no public methods with the @Subscribe annotation
```



## 四、SubscriberMethodFinder

前文铺垫 __Subscription__、__SubscriberMethod__、__Subscribe__ 注解，全是都是为了减低 __SubscriberMethodFinder__ 类的理解。因为通过此类扫描订阅者方法的 __Subscribe__ 注解，为每个订阅方法生成 __SubscriberMethod__，构造出订阅记录 __Subscription__。所有事件根据  __Subscription__ 派到对应订阅者的订阅方法。


#### 4.1 类签名

```java
class SubscriberMethodFinder 
```

#### 4.2 常量

```java
/*
 * In newer class files, compilers may add methods. Those are called bridge or synthetic methods.
 * EventBus must ignore both. There modifiers are not public but defined in the Java class file format:
 * http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1
 */
private static final int BRIDGE = 0x40;
private static final int SYNTHETIC = 0x1000;

// 忽视抽象方法、静态方法、桥接方法、自动生成方法
private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;

// 扫描订阅者和订阅方法后的缓存
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

private static final int POOL_SIZE = 4;
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
```

#### 4.3 数据成员

```java
private List<SubscriberInfoIndex> subscriberInfoIndexes;
private final boolean strictMethodVerification;
private final boolean ignoreGeneratedIndex;
```

#### 4.4 构造方法

```java
SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                       boolean ignoreGeneratedIndex) {
    this.subscriberInfoIndexes = subscriberInfoIndexes;
    this.strictMethodVerification = strictMethodVerification;
    this.ignoreGeneratedIndex = ignoreGeneratedIndex;
}
```

#### 4.5 findSubscriberMethods

在订阅者类内扫描订阅者方法，如果订阅者类没有目标方法直接抛出异常

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 根据订阅者类从缓存中获取订阅者方法
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }

    // ignoreGeneratedIndex在EventBusBuilder.ignoreGeneratedIndex为false
    if (ignoreGeneratedIndex) {
        // 通过反射的方式查找订阅者方法
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        // 查找订阅者方法
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    
    // 在订阅者中没有找到使用注解标注的公开方法
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 结果放入缓存中
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        // 返回订阅者方法
        return subscriberMethods;
    }
}
```

####  4.6 

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

#### 4.6 findUsingInfo

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

#### 4.7 getMethodsAndRelease

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle();
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```

#### 4.8 prepareFindState

```java
private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```

#### 4.10 getSubscriberInfo

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}
```

#### 4.12 findUsingReflectionInSingleClass

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

```java
static void clearCaches() {
    METHOD_CACHE.clear();
}
```

## 五、FindState

```java
static class FindState {
    final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
    final Map<Class, Object> anyMethodByEventType = new HashMap<>();
    final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
    final StringBuilder methodKeyBuilder = new StringBuilder(128);

    Class<?> subscriberClass;
    Class<?> clazz;
    boolean skipSuperClasses;
    SubscriberInfo subscriberInfo;

    void initForSubscriber(Class<?> subscriberClass) {
        this.subscriberClass = clazz = subscriberClass;
        skipSuperClasses = false;
        subscriberInfo = null;
    }

    void recycle() {
        subscriberMethods.clear();
        anyMethodByEventType.clear();
        subscriberClassByMethodKey.clear();
        methodKeyBuilder.setLength(0);
        subscriberClass = null;
        clazz = null;
        skipSuperClasses = false;
        subscriberInfo = null;
    }

    boolean checkAdd(Method method, Class<?> eventType) {
        // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
        // Usually a subscriber doesn't have methods listening to the same event type.
        Object existing = anyMethodByEventType.put(eventType, method);
        if (existing == null) {
            return true;
        } else {
            if (existing instanceof Method) {
                if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                    // Paranoia check
                    throw new IllegalStateException();
                }
                // Put any non-Method object to "consume" the existing Method
                anyMethodByEventType.put(eventType, this);
            }
            return checkAddWithMethodSignature(method, eventType);
        }
    }

    private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
        methodKeyBuilder.setLength(0);
        methodKeyBuilder.append(method.getName());
        methodKeyBuilder.append('>').append(eventType.getName());

        String methodKey = methodKeyBuilder.toString();
        Class<?> methodClass = method.getDeclaringClass();
        Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
        if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
            // Only add if not already found in a sub class
            return true;
        } else {
            // Revert the put, old class is further down the class hierarchy
            subscriberClassByMethodKey.put(methodKey, methodClassOld);
            return false;
        }
    }

    void moveToSuperclass() {
        if (skipSuperClasses) {
            clazz = null;
        } else {
            clazz = clazz.getSuperclass();
            String clazzName = clazz.getName();
            /** Skip system classes, this just degrades performance. */
            if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                clazz = null;
            }
        }
    }
}
```

## 六、事件发送给订阅者

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