---
layout:     post
title:      "Android源码系列 -- Message"
date:       2016-11-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

Handler是Android中一种处理线程消息循环的机制，而 [Message](https://developer.android.com/reference/android/os/Message.html) 是Handler用来放消息的包装。总得来说，Message作为一个用于封装消息的对象，逻辑并不复杂。

```java
public final class Message implements Parcelable {
    // Source Code here
}
```

Android常用序列化有 Serializable 和 [Parcelable](https://developer.android.com/reference/android/os/Parcelable.html) 两种，该类继承Parcelable接口表明支持序列化。简单说，前者用的时间比较长且范围更广，但是序列化过程中产生大量小对象；后者性能好，但是需要手动实现4个必须方法，范围也没有前者广。


# 一、成员变量

用一个标志来区分不同的消息的身份。不同的Handler里使用相同`what`值的不同消息不会弄混。一般用十六进制形式表示，阅读起来比较容易。

```java
public int what; // 0x01
```

`arg1`和`arg2`都是类中可选的变量存储位置，可以方便地用来存放两个数值，这样就不用访问`obj`对象就能读取变量。

```java
public int arg1; 
public int arg2;
```

`obj`用来保存一个需要的对象，在处理的时候取出来使用。

```java
public Object obj; // 用来保存对象
public Messenger replyTo; // 回复跨进程的Messenger
public int sendingUid = -1; // Messenger发送时使用

static final int FLAG_IN_USE = 1 << 0; // 正在使用标志值
static final int FLAG_ASYNCHRONOUS = 1 << 1; // 异步标志值
static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

int flags; // 消息标志，上面三个常量 FLAG_* 用在这里
long when; // 估计和arg1、arg2性质一样，存时间戳

Bundle data;    // 存放Bundle
Handler target; // 存放Handler实例
Runnable callback; // 消息的回调操作
Message next;   // 消息池用链表的方式存储

private static final Object sPoolSync = new Object(); // 消息池同步公用标志
private static Message sPool; // 消息池
private static int sPoolSize = 0; // 消息池已缓存数量
private static final int MAX_POOL_SIZE = 50; // 消息池最大容量
private static boolean gCheckRecycle = true; // 该版本系统是否支持回收标志位
```

# 二、消息体获取

从消息池里取可以复用的消息对象。方法体有一个同步代码块，对象`sPoolSync`作为锁标志，避免不同线程取到同一个消息体导致消息紊乱。如果没有可复用的对象，就新建一个消息体返回。

我们可以手动创建一个消息对象，但是最好从`obtain()`中获取缓存好的消息体，避免造成多余对象创建。

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // 移除使用中标志
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

从消息池中取可用的消息体，然后把形参全部复制进去。

```java
public static Message obtain(Message orig) {
    Message m = obtain();
    m.what = orig.what;
    m.arg1 = orig.arg1;
    m.arg2 = orig.arg2;
    m.obj = orig.obj;
    m.replyTo = orig.replyTo;
    m.sendingUid = orig.sendingUid;
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

下面都是获取消息体然后设置消息体参数的方法，就是`public static Message obtain()`的重载。

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

检查系统是否支持消息对象循环回收

```
public static void updateCheckRecycle(int targetSdkVersion) {
    if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
        gCheckRecycle = false;
    }
}
```

对象使用完毕就会调用`recycleUnchecked()`

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
    // 当消息还在回收池中，添加正在使用标志位，其他情况就清除掉
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
    // 最多缓存50个消息体，多余的GC
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
// 把消息发送到Handler，配合obtain()使用，这样target不为空。
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

# 六、标志位

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
 
实现Parcelable接口的四个固定方法 `CREATOR`、`describeContents()`、`writeToParcel()`、`readFromParcel()`

```
public static final Parcelable.Creator<Message> CREATOR
        = new Parcelable.Creator<Message>() {
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

