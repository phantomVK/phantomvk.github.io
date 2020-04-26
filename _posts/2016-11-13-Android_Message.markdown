---
layout:     post
title:      "Android源码系列(3) -- Message"
date:       2016-11-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

[Handler](/2016/12/01/Android_Handler/)是Android中一种处理线程消息循环的机制，而[Message](https://developer.android.com/reference/android/os/Message.html)是Handler放消息的包装。

```java
public final class Message implements Parcelable
```

Android常用序列化有Serializable和[Parcelable](https://developer.android.com/reference/android/os/Parcelable.html)两种。前者历史悠久且范围更广，但序列化过程产生大量小对象。后者性能好，但需要手动实现序列化实现方法且只用在Android。而本类实现后者。


# 一、成员变量

此标志用于区分不同消息的身份，不同Handler使用相同值的消息不会弄混。因为日志输出时该值以16进制形式显示，所以设置时建议用16进制。

```java
public int what; // 0x01
```

`arg1`和`arg2`是类中可选变量，用于存放两个整形值，无需访问`obj`对象即可使用。

```java
public int arg1; 
public int arg2;
```

`obj`用来保存对象(负载)，接受消息后取出获得传送的对象。

```java
// 用来保存对象
public Object obj;

// 回复跨进程的Messenger
public Messenger replyTo;

// Messenger发送时使用
public int sendingUid = -1;
```
相关标志位：

```java
// 正在使用标志值
static final int FLAG_IN_USE = 1 << 0;

// 异步标志值
static final int FLAG_ASYNCHRONOUS = 1 << 1;

static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

// 上面三个常量标志位用在这里
int flags;

// 消息分发的目标时间戳
long when;
```
其他数据成员：
```java
// 存放Bundle
Bundle data;

// 存放所属Handler实例
Handler target;

// 消息回调操作
Runnable callback;

// 消息池用链表的方式存储
Message next;

// 消息池同步锁对象
private static final Object sPoolSync = new Object();

// 消息池
private static Message sPool;

// 已缓存消息数量
private static int sPoolSize = 0;

// 消息池最大容量
private static final int MAX_POOL_SIZE = 50;

// 该版本系统是否支持回收标志位
private static boolean gCheckRecycle = true;
```

# 二、消息体获取

从消息池中获得复用消息对象。方法体以`sPoolSync`作为同步代码块的锁标志，避免不同线程获得相同空消息体导致使用紊乱。没有可复用对象则直接创建新对象。

当然也可以手动创建新消息对象，但是最好的方式还是从`obtain()`中获取缓存好的空消息体。

```java
public static Message obtain() {
    // 上锁
    synchronized (sPoolSync) {
        if (sPool != null) {
            // 取链头的缓存对象m
            Message m = sPool;
            // 把sPool当头指针使用，指向m之后有效的缓存对象
            sPool = m.next;
            // 置空缓存对象m的next引用
            m.next = null;
            // 移除使用中标志
            m.flags = 0;
            // 缓存池缓存对象数量自减1
            sPoolSize--;
            return m;
        }
    }

    // 缓存池为空，返回新构建的Message
    return new Message();
}
```

从消息池中取可用消息体后赋值实参

```java
public static Message obtain(Message orig) {
    Message m = obtain();
    m.what = orig.what;
    m.arg1 = orig.arg1;
    m.arg2 = orig.arg2;
    m.obj = orig.obj;
    m.replyTo = orig.replyTo;
    m.sendingUid = orig.sendingUid;
    // 深拷贝
    if (orig.data != null) {
        m.data = new Bundle(orig.data);
    }
    m.target = orig.target;
    m.callback = orig.callback;

    return m;
}
```

返回一个消息，这个消息已经设置发送目标

```java
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h;

    return m;
}
```

下面是获取消息体然后设置消息体参数的方法，均为`obtain()`的重载。

```java
public static Message obtain(Handler h, Runnable callback) {
    Message m = obtain();
    m.target = h;
    m.callback = callback;

    return m;
}

public static Message obtain(Handler h, int what) {
    Message m = obtain();
    m.target = h;
    m.what = what;

    return m;
}

public static Message obtain(Handler h, int what, Object obj) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.obj = obj;

    return m;
}

public static Message obtain(Handler h, int what, int arg1, int arg2) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.arg1 = arg1;
    m.arg2 = arg2;

    return m;
}

public static Message obtain(Handler h, int what, int arg1, int arg2, Object obj) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.arg1 = arg1;
    m.arg2 = arg2;
    m.obj = obj;

    return m;
}
```

# 三、消息回收

检查当前系统版本是否支持消息对象循环回收。当消息已经保存在缓存池中时会标记为`IN_USE`，不能再次回收。

```java
public static void updateCheckRecycle(int targetSdkVersion) {
    if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
        gCheckRecycle = false;
    }
}
```

对象使用完毕最终调用`recycleUnchecked()`

```java
public void recycle() {
    if (isInUse()) {
        if (gCheckRecycle) {
            throw new IllegalStateException("This message cannot be recycled because it "
                    + "is still in use.");
        }
        return;
    }

    recycleUnchecked();
}
```

消息体清空回收

```java
void recycleUnchecked() {
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    
    // 把消息加入到缓存池，最多缓存50个空消息体
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

# 四、copyFrom

从一个消息体复制参数到另一个消息体。

```java
public void copyFrom(Message o) {
    this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
    this.what = o.what;
    this.arg1 = o.arg1;
    this.arg2 = o.arg2;
    this.obj = o.obj;
    this.replyTo = o.replyTo;
    this.sendingUid = o.sendingUid;

    if (o.data != null) {
        this.data = (Bundle) o.data.clone();
    } else {
        this.data = null;
    }
}
```

# 五、Getter & Setter

```java
public void setTarget(Handler target) {
    this.target = target;
}

public Handler getTarget() {
    return target;
}

public long getWhen() {
    return when;
}

public Runnable getCallback() {
    return callback;
}

public Bundle getData() {
    if (data == null) {
        data = new Bundle();
    }
    
    return data;
}

// peekData()对比getData()
public Bundle peekData() {
    return data;
}

public void setData(Bundle data) {
    this.data = data;
}

// 把消息发送到Handler，配合obtain()使用令target不为空
public void sendToTarget() {
    target.sendMessage(this);
}

// async为true将不受MessageQueue中SyncBarrier的影响
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

# 六、标志操作

```java
public boolean isAsynchronous() {
    return (flags & FLAG_ASYNCHRONOUS) != 0;
}

boolean isInUse() {
    return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
}

void markInUse() {
    flags |= FLAG_IN_USE;
}
```
