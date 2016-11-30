---
layout:     post
title:      "Android Message源码阅读"
date:       2016-11-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
    - 源码阅读
---

Handler是Android中一种处理线程消息循环的机制，而Message是在Handler中流动的对象，就像是邮件一样。总得来说，Message只是作为一个用于封装消息的对象，其中并不复杂。

Message继承了一个Parcelable接口表明支持序列化，并且是final类不能继承也不能修改。

Android常用的序列化上有Serializable和Parcelable两种。简单说，前者用的时间比较长且范围更广，但是序列化过程中产生大量小对象对JVM有压力；后者性能好，但是需要手动实现4个必须方法有点麻烦。

```
public final class Message implements Parcelable {
    // Source Code here
}
```

用一个标志来区分不同的消息

```
public int what;
```

`arg1`和`arg2`都是预置的存储变量，可以用来存放零碎整形。这样就不用把变量包装到Obj对象里面，毕竟取出来也麻烦。

```
public int arg1; 
public int arg2;
```

```
// 用来保存对象
public Object obj;

/**
 * Optional Messenger where replies to this message can be sent.  The
 * semantics of exactly how this is used are up to the sender and
 * receiver.
 */
public Messenger replyTo;

/**
 * Optional field indicating the uid that sent the message.  This is
 * only valid for messages posted by a {@link Messenger}; otherwise,
 * it will be -1.
 */
public int sendingUid = -1;

/** If set message is in use.
 * This flag is set when the message is enqueued and remains set while it
 * is delivered and afterwards when it is recycled.  The flag is only cleared
 * when a new message is created or obtained since that is the only time that
 * applications are allowed to modify the contents of the message.
 *
 * It is an error to attempt to enqueue or recycle a message that is already in use.
 */
/*package*/ static final int FLAG_IN_USE = 1 << 0;
```

```
// 异步标志位
/*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;

/** Flags to clear in the copyFrom method */
/*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

/*package*/ int flags;

/*package*/ long when;

/*package*/ Bundle data;

/*package*/ Handler target;

/*package*/ Runnable callback;
```

消息对象复用链表指针

```
Message next;
```

 
同步锁对象

```
private static final Object sPoolSync = new Object();
```

对象池

```
private static Message sPool;
```

记录对象池缓存对象数量

```
private static int sPoolSize = 0;
```

对象池最大容量

```
private static final int MAX_POOL_SIZE = 50;
```

该版本系统是否支持回收标志位

```
private static boolean gCheckRecycle = true;
```

从消息池里取可以复用的消息对象。方法体有一个同步代码块，锁的是一个公共对象`sPoolSync`，避免不同线程取到同一个消息体导致消息紊乱。如果没有可复用的对象，就新建一个消息体返回。

```
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

从消息池中取一个可用的消息体，然后把形参的参数全部复制到这个消息体里面。虽然我们可以手动创建一个消息对象，但是建议从obtain方法中获取缓存好的消息体，避免造成多余对象创建和销毁的处理。

```
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

```
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h;

    return m;
}
```

下面都是获取消息体然后设置消息体参数的方法，就是`public static Message obtain()`的重载。

```
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

public static Message obtain(Handler h, int what, 
        int arg1, int arg2, Object obj) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.arg1 = arg1;
    m.arg2 = arg2;
    m.obj = obj;

    return m;
}
```

检查系统是否支持消息对象循环回收

```
public static void updateCheckRecycle(int targetSdkVersion) {
    if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
        gCheckRecycle = false;
    }
}
```

对象没有使用就会调用`recycleUnchecked`

```
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

```
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
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

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

从一个消息体复制参数到另一个消息体

```
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

获取消息时间，单位是毫秒

``` 

public long getWhen() {
    return when;
}
```
 
 设置消息发送到Handler
 
```
public void setTarget(Handler target) {
    this.target = target;
}

public Handler getTarget() {
    return target;
}
```

JavaBean对应的Getter、Setter

```
public Runnable getCallback() {
    return callback;
}

public Bundle getData() {
    if (data == null) {
        data = new Bundle();
    }
    
    return data;
}

public Bundle peekData() {
    return data;
}

public void setData(Bundle data) {
    this.data = data;
}

把消息发送到Handler，配合obtain()使用，这样target不为空。

public void sendToTarget() {
    target.sendMessage(this);
}

public boolean isAsynchronous() {
    return (flags & FLAG_ASYNCHRONOUS) != 0;
}

public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

标志位状态改变方法

```
/*package*/ boolean isInUse() {
    return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
}

/*package*/ void markInUse() {
    flags |= FLAG_IN_USE;
}
```

显式实现的空构造器

```
public Message() {
}
```
 
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

