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

Handler是Android中一种处理线程消息循环的机制，而 [Message](https://developer.android.com/reference/android/os/Message.html) 是Handler放消息的包装。

```java
public final class Message implements Parcelable
```

Android常用序列化有 Serializable 和 [Parcelable](https://developer.android.com/reference/android/os/Parcelable.html) 两种，该类支持后者的序列化。前者用的时间比较长且范围更广，但是序列化过程中产生大量小对象；后者性能好，但是需要手动实现序列化实现方法，只有Android中可以使用。


# 一、成员变量

用一个标志来区分不同的消息的身份，不同的Handler使用相同值的消息不会弄混。一般用十六进制形式表示，阅读起来比较容易。

```java
public int what; // 0x01
```

`arg1`和`arg2`都是类中可选变量，可以用来存放两个数值，不用访问`obj`对象就能读取变量。

```java
public int arg1; 
public int arg2;
```

`obj`用来保存对象，接受消息后取出获得传送的对象。

```java
public Object obj; // 用来保存对象
public Messenger replyTo; // 回复跨进程的Messenger
public int sendingUid = -1; // Messenger发送时使用
```

```java
static final int FLAG_IN_USE = 1 << 0; // 正在使用标志值
static final int FLAG_ASYNCHRONOUS = 1 << 1; // 异步标志值
static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

int flags; // 消息标志，上面三个常量 FLAG_* 用在这里
long when; // 存时间戳
```

```java
Bundle data;    // 存放Bundle
Handler target; // 存放Handler实例
Runnable callback; // 消息的回调操作
Message next;   // 消息池用链表的方式存储

private static final Object sPoolSync = new Object(); // 消息池同步锁对象
private static Message sPool; // 消息池
private static int sPoolSize = 0; // 消息池已缓存数量
private static final int MAX_POOL_SIZE = 50; // 消息池最大容量
private static boolean gCheckRecycle = true; // 该版本系统是否支持回收标志位
```

# 二、消息体获取

从消息池中获得可复用消息对象。方法体有一个同步代码块，对象`sPoolSync`作为锁标志，避免不同线程取同一个空消息体导致使用紊乱。如果没有可复用的对象，会就地创建新的Message对象。

当然我们可以手动创建一个消息对象，但是最好的方式还是从`obtain()`中获取缓存好的空消息体，避免创建多余对象。

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            // 取链头的缓存对象m
            Message m = sPool;
            // 把sPool当头指针使用，指向m之后有效的缓存对象
            sPool = m.next;
            // 置空缓存对象m的next指针
            m.next = null;
            // 移除使用中标志
            m.flags = 0;
            // 缓存池缓存对象数量自减1
            sPoolSize--;
            return m;
        }
    }
    // 如果缓存池是空的，就立即创建一个新的Message
    return new Message();
}
```

从消息池中取可用的消息体，然后把实参全部复制进去

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

返回一个消息，这个消息已经设置好发送的目标

```java
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h;

    return m;
}
```

下面都是获取消息体然后设置消息体参数的方法，就是`obtain()`的重载。

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

检查当前系统版本是否支持消息对象循环回收。当消息已经保存在缓存池中时，就会标记为`IN_USE`，不能再次回收。

```java
public static void updateCheckRecycle(int targetSdkVersion) {
    // 低于Android 5.0不支持对象缓存
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
    flags = FLAG_IN_USE; // 添加正在使用标志位，其他情况就清除掉
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
    
    // 最多缓存49个空消息体
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

# 七、Parcelable实现
 
实现Parcelable接口的方法`describeContents()`、`writeToParcel()`、`readFromParcel()`

```java
public static final Parcelable.Creator<Message> CREATOR
        = new Parcelable.Creator<Message>() {
    // 通过Parcelable构造一个Message
    public Message createFromParcel(Parcel source) {
        Message msg = Message.obtain();
        msg.readFromParcel(source);
        return msg;
    }
    
    public Message[] newArray(int size) {
        return new Message[size];
    }
};
    
public int describeContents() {
    return 0;
}

public void writeToParcel(Parcel dest, int flags) {
    if (callback != null) {
        throw new RuntimeException(
            "Can't marshal callbacks across processes.");
    }
    dest.writeInt(what);
    dest.writeInt(arg1);
    dest.writeInt(arg2);
    // obj对象一定要实现了Parcelable，否则无法支持跨进程通讯
    if (obj != null) {
        try {
            Parcelable p = (Parcelable)obj;
            dest.writeInt(1);
            dest.writeParcelable(p, flags);
        } catch (ClassCastException e) {
            throw new RuntimeException(
                "Can't marshal non-Parcelable objects across processes.");
        }
    } else {
        dest.writeInt(0);
    }
    dest.writeLong(when);
    dest.writeBundle(data);
    Messenger.writeMessengerOrNullToParcel(replyTo, dest);
    dest.writeInt(sendingUid);
}

private void readFromParcel(Parcel source) {
    what = source.readInt();
    arg1 = source.readInt();
    arg2 = source.readInt();
    if (source.readInt() != 0) {
        obj = source.readParcelable(getClass().getClassLoader());
    }
    when = source.readLong();
    data = source.readBundle();
    replyTo = Messenger.readMessengerOrNullFromParcel(source);
    sendingUid = source.readInt();
}
```

