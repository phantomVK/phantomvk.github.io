---
layout:     post
title:      "EventBus源码剖析(4) -- 订阅记录"
date:       2018-12-06
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - EventBus
---

文章列表：
- [EventBus源码剖析(1) -- 订阅注册与注销](/2018/11/15/EventBus_1_Register/)
- [EventBus源码剖析(2) -- EventBusBuilder](/2018/12/01/EventBus_2_EventBusBuilder/)
- [EventBus源码剖析(3) -- 线程模式](/2018/12/03/EventBus_3_ThreadMode/)
- [EventBus源码剖析(4) -- 订阅记录](/2018/12/06/EventBus_4_Subscription/)
- [EventBus源码剖析(5) -- Poster](/2018/12/10/EventBus_5_Poster/)

## 一、Subscription

订阅者进行注册时，__EventBus__ 会扫描整个订阅者类，获取接收事件的具体方法，并构造出 __Subscription__ 实例。每个订阅者可有多个方法接收订阅事件，每个方法生成各自的 __Subscription__ 作为事件接收凭证。

订阅记录内主要包括3个成员变量：

- `subscriber`表示订阅类的实例；
- `subscriberMethod`表示订阅者内接受事件的方法；
- 还有表示订阅记录是否有效的`active`；

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
            // 对比使用同一个实例且注册了同一个订阅方法
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

调用 __EventBus#unregister(Object)__ 注销订阅者后，`active`立即改为 __false__。该值被负责队列事件投递的 __EventBus#invokeSubscriber(PendingPost)__ 检查以避免 __race conditions__。

## 二、SubscriberMethod

具体到 __Subscription__ 内部实现，里面还包含了 __SubscriberMethod__。每个 __SubscriberMethod__ 表示一个订阅者类的订阅方法。

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
            // Don't use method.equals because of https://issuetracker.google.com/issues/36916580#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString开销较大，只需要取方法名
            StringBuilder builder = new StringBuilder(64);
            // 三元组：订阅者类、订阅方法、订阅消息类型
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
    // 通过指定线程模式调起订阅者方法
    ThreadMode threadMode() default ThreadMode.POSTING;

    // 若是粘性事件，把最近的粘性事件发送给订阅者
    boolean sticky() default false;

    // 方法接收事件的优先级，默认优先级是0
    // 在相同线程内，高优先级订阅者方法比低优先级方法更早接收事件
    // 此优先级不会影响不同线程模式中不同订阅者事件的派发
    int priority() default 0;
}
```

若注解方法无需修改默认条件，则使用 __@Subscribe__ 时不指定参数即可

```java
// 三个参数的设置
@Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
fun onEventReceived(event: UserEvent) {
    val name = event.name
    val age = event.age
    Log.i(TAG, "Event received, Name: $name, Age: $age.")
}
```

如果实例没有可执行方法，注册到 __EventBus__ 过程的类扫描会抛出异常：

```
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.phantomvk.playground/com.phantomvk.playground.MainActivity}: org.greenrobot.eventbus.EventBusException: Subscriber class com.phantomvk.playground.MainActivity and its super classes have no public methods with the @Subscribe annotation
```

## 四、SubscriberMethodFinder

#### 4.1 类签名

前文铺垫 __Subscription__、__SubscriberMethod__、__Subscribe__ 注解，降低这里 __SubscriberMethodFinder__ 类的理解难度。

```java
class SubscriberMethodFinder 
```

通过扫描订阅者方法的 __Subscribe__ 注解，为每个订阅方法生成 __SubscriberMethod__，构造出订阅记录 __Subscription__。

#### 4.2 常量

较新的类文件中编译器可能会添加额外方法， 这些方法被称为桥梁或合成方法。__EventBus__ 同时忽略这两种方法。这些修饰符都不是 __public__ 的，而是以Java类文件格式定义：http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1

```java
private static final int BRIDGE = 0x40;
private static final int SYNTHETIC = 0x1000;
```

忽略抽象方法、静态方法、桥接方法、合成方法

```java
private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
```

扫描订阅者和订阅方法后生成缓存

```java
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
```

__FindState__ 对象缓存池大小

```java
private static final int POOL_SIZE = 4;
```

__FindState__ 对象缓存池

```java
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

在订阅者类内扫描订阅者方法，如果订阅者类没有目标方法直接抛出异常。从缓存列表中查找缓存减少扫描时间，缓存命中就不需要从订阅者类中进行查找。

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

####  4.6 findUsingReflection

通过反射查找订阅者的订阅方法

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    // 从缓存池获取findState
    FindState findState = prepareFindState();
    // 用订阅者类初始化findState
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

#### 4.7 findUsingInfo

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    // 从缓存池获取findState
    FindState findState = prepareFindState();
    // 用订阅者类初始化findState
    findState.initForSubscriber(subscriberClass);
    // 遇到："java."、"javax."、"android."的类名clazz会置空，并终止这个循环
    while (findState.clazz != null) {
        // 如果订阅者之前没有注册过，则以下返回值为null，需通过反射的方式查找
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            // 上面无差别订阅类中所有方法，下面需要遍历筛选出有效的订阅方法
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        // 扫描目标类转移到父类上
        findState.moveToSuperclass();
    }
    // 返回List<SubscriberMethod>，释放FindState实例
    return getMethodsAndRelease(findState);
}
```

#### 4.8 getMethodsAndRelease

从 __findState__ 获取订阅者的订阅方法， __findState__ 重置后放入缓存，最后返回订阅者方法列表

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    // 从findState获取所有订阅方法
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    // 重置findState
    findState.recycle();
    // 把findState放入缓存池中
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            // 找一个非空位置存放可缓存实例
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    // 返回findState中保存的subscriberMethods
    return subscriberMethods;
}
```

#### 4.9 prepareFindState

重缓存池中获取 __FindState__ 实例，如果缓存池没有缓存的实例，则创建新实例

```java
private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            // 从缓存池取一个可用的实例
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    // 缓存池为空，创建并返回新实例
    return new FindState();
}
```

#### 4.10 getSubscriberInfo

获取订阅者的信息

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
    // 首次注册的订阅者，以下条件判断为空
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    // 首次注册的订阅者，以下条件判断为空
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    // 首次注册的订阅者返回值为null
    return null;
}
```

#### 4.11 findUsingReflectionInSingleClass

__FindState.clazz__ 就是订阅者类。先把订阅者的方法收集成为列表，后逐个分析方法的注解。根据约束条件逐渐筛选出符合条件的方法，放入到 __FindState.subscriberMethods__。

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // 此方法比getMethods速度更快，尤其是订阅者像Activities这种巨型类的时候
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    
    // 遍历订阅者类内所有方法
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        // 方法使用可见性为public，且没有使用MODIFIERS_IGNORE修饰
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            // 从方法获取变量类型
            Class<?>[] parameterTypes = method.getParameterTypes();
            // 变量的数量必须为1个，否则不处理
            if (parameterTypes.length == 1) {
                // 检查方法是否有Subscribe注解修饰
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 从方法变量中确认订阅事件的类型
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        // 从注解获取threadMode信息
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        // 向findState增加新SubscriberMethod
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                // 订阅者方法的形参数量不为1，但又使用Subscribe注解修饰方法，需抛出异常
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            // 方式包含Subscribe注解，但不方法不能同时满足以下条件：公开可见性、非静态、非抽象
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

#### 4.12 clearCaches

清除缓存

```java
static void clearCaches() {
    METHOD_CACHE.clear();
}
```

## 五、FindState

#### 5.1 类信息

__EventBus__ 创建多个缓存的 __FindState__ 实例，当有订阅者需要扫描订阅方法时，从缓存池中取出一个 __FindState__ ，向此实例中放入需被扫描的订阅者类。之后，包含订阅者的  __FindState__ 传递给负责处理工作的方法，处理完毕后的订阅者方法结果也会存回到  __FindState__ 。

```java
static class FindState 
```

因此处理完毕后的 __FindState__ 既包含订阅者类的信息，也保存着订阅者方法的列表。__EventBus__ 从 __FindState__ 获取所有订阅者方法后，该 __FindState__ 会重置并重新放入缓存池中。

根据  __FindState__ 内部结构可知，__FindState__ 实例包含 __ArrayList__、__HashMap__、初始长度为128的 __StringBuilder__，所以 __FindState__ 实例本身就有一定分量。如果实例不断创建并销毁，会加重虚拟机垃圾回收的负担。相反，把用过的   __FindState__ 放入缓存池重用应该是更合理的行为。

从 __SubscriberMethodFinder__ 的 __POOL_SIZE__ 常量可知缓存池大小为4。

```java
private static final int POOL_SIZE = 4;
```

#### 5.2 不可变成员

这个数据成员用于保存订阅者类扫描扫描方法的结果，使用完毕后会简单清空为下次重用。

```java
final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
final Map<Class, Object> anyMethodByEventType = new HashMap<>();
final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
final StringBuilder methodKeyBuilder = new StringBuilder(128);
```

#### 5.3 可变成员

这些成员用于存储订阅者类的信息。每次 __EventBus__ 从缓存池获取 __FindState__ 缓存实例后，都会把订阅者类的基本信息存入以下变量，作为后续操作的参考内容。

```java
// 保存订阅者的类型
Class<?> subscriberClass;
Class<?> clazz;
boolean skipSuperClasses;
// 订阅者的信息，包含订阅者类的方法数组、给指向父类SubscriberInfo的引用
SubscriberInfo subscriberInfo;
```

#### 5.4 initForSubscriber

订阅者类数据通过此方法设置到 __FindState__

```java
void initForSubscriber(Class<?> subscriberClass) {
    this.subscriberClass = clazz = subscriberClass;
    skipSuperClasses = false;
    subscriberInfo = null;
}
```

#### 5.5 recycle

__FindState__ 使用完毕后需调用 __recycle()__ 清空后归还给缓存池。这个方法的处理方式有点像 __Message__ 类的 [recycleUnchecked()](/2016/11/13/Android_Message/#三消息回收) 方法。

```java
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
```

#### 5.6 checkAdd

两重检验：第一层，通过事件类型快速检验；第二层，在需要时通过完整的签名检查。一般来说，订阅者类不会有方法订阅同一个事件。

```java
boolean checkAdd(Method method, Class<?> eventType) {
    // 检查同一个订阅者类内是有多个方法订阅相同事件
    Object existing = anyMethodByEventType.put(eventType, method);
    if (existing == null) {
        // 有多个方法订阅相同事件
        return true;
    } else {
        if (existing instanceof Method) {
            if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                throw new IllegalStateException();
            }
            // Put any non-Method object to "consume" the existing Method
            anyMethodByEventType.put(eventType, this);
        }
        return checkAddWithMethodSignature(method, eventType);
    }
}
```

#### 5.7 checkAddWithMethodSignature

通过方法签名检查方法是否重复注册，检查通过就能加入到列表。

```java
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
    methodKeyBuilder.setLength(0);
    methodKeyBuilder.append(method.getName());
    methodKeyBuilder.append('>').append(eventType.getName());

    // 例如：onEventReceived>UserEvent
    String methodKey = methodKeyBuilder.toString();
    Class<?> methodClass = method.getDeclaringClass();
    // 通过插入新(methodKey, methodClass)，并获取旧value
    Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
    // 旧value为空，或methodClassOld是methodClass的父类或同类
    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
        // 方法可以添加订阅
        return true;
    } else {
        // 撤销此次插入，保留缓存中的旧值
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }
}
```

#### 5.8 moveToSuperclass

```java
void moveToSuperclass() {
    if (skipSuperClasses) {
        clazz = null;
    } else {
        clazz = clazz.getSuperclass();
        // 获取类名
        String clazzName = clazz.getName();

        // 跳过系统类，这只会降低性能。
        if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
            clazz = null;
        }
    }
}
```

## 六、事件发送给订阅者

#### 6.1 invokeSubscriber

根据 __event__ 和 订阅记录 __Subscription__，就能把事件发送给订阅方法。

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        // 用订阅者的实例和订阅的事件调起订阅方法
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        // 订阅者方法接收事件是出现异常
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        // 出现未知异常
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

#### 6.2 handleSubscriberException

检查标志位决定是否处理事件订阅方法抛出的异常

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