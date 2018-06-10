---
layout:     post
title:      "Android源码系列(4) -- Handler"
date:       2016-12-01
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

# 一、作用

Handler有两个主要用法：

 * 计划在将来某个时间点处理Message和Runnable
 * 在不同线程里将一个动作加入Handler所对应的队列去执行

# 二、成员变量

Handler有4个不可变成员变量：

```java
final MessageQueue mQueue;   // 消息队列
final Looper mLooper;        //消息队列所属Looper
final Callback mCallback;    // 可选Handler回调
final boolean mAsynchronous; // 可选异步标志
```


# 三、构造方法

如果线程已经开启Looper，Handler可以使用下列构造方法。

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback) {
    this(callback, false);
}

public Handler(boolean async) {
    this(null, async);
}

public Handler(Callback callback, boolean async) {
    // 判断是否匿名类、本地类、成员类，判断修饰符是否static，避免内存泄漏
    // 只要是静态内部类，就不会持有外部类引用从而造成内存泄漏
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    
    // 主动获取Handler所在线程的Looper
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

带Looper形参的构造方法。通常和`Looper.getMainLooper()`合用。

``` java
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

# 四、封装

作用是把`r`封装到`msg.callback`，把`token`赋值给`m.obj`。

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}

private static Message getPostMessage(Runnable r, Object token) {
    Message m = Message.obtain();
    m.obj = token;
    m.callback = r;
    return m;
}
```


# 五、消息发送

方法封装形参`Runnable`，方法名组成是 `post()`：

```java
public final boolean post(Runnable r) {
   return sendMessageDelayed(getPostMessage(r), 0); //delayMillis = 0 
}

// 时间单位毫秒，如:delayMillis = 1000
public final boolean postDelayed(Runnable r, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```

方法形参是`msg`或`msg.what`，方法名组成是 `sendMessage()`：

```java
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}

// msg.what用16进制，如:0x01
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
```

以上方法带`Delayed`可设置延迟时间，带`EmptyMessage`为创建空消息。共同点是都调用了`sendMessageDelayed()`，并返回这个调用的结果。

`SystemClock.uptimeMillis()`是从开机到现在的毫秒数，不包括手机睡眠的时间。个人估计使用`SystemClock.uptimeMillis()`是为了避免用户调整系统时间后影响`System.currentTimeMillis()`，导致消息分发时间点异常。

`postAtTime()`重载方法调用了`sendMessageAtTime()`。

```java
public final boolean postAtTime(Runnable r, long uptimeMillis){
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}

public final boolean postAtTime(Runnable r, Object token, long uptimeMillis){
    return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
}
```

`sendEmptyMessage()`调`sendEmptyMessageDelayed()`，`sendEmptyMessageDelayed()`和`sendEmptyMessageAtTime`最终调用`sendMessageAtTime()`。

```java
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageAtTime(msg, uptimeMillis);
}
```

总而言之，上面所有post和send都终结在`sendMessageAtTime()`，而`sendMessageAtTime()`仅负责把消息确定一个具体执行时间点，然后送进消息队列中。

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

消息默认放在消息队列的队尾处，返回`true`代表成功进入队列，不代表消息会被调度。

一般情况下消息队列都会等待所有消息完成才退出。如果手动关闭消息队列，滞留在消息队列的消息不会得到处理且消息被丢弃，这是进入消息队列却不一定能调度的主要原因。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // `this` is a Handler.
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

消息也可以放在消息队列头优先执行，不过这两个方法只能在非常特殊的情况下采取用，因为顺序问题和未知副作用很容易导致队列后方消息发生饥饿。

```java
public final boolean postAtFrontOfQueue(Runnable r){
    return sendMessageAtFrontOfQueue(getPostMessage(r));
}

public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```

# 六、调度和回调

### 6.1 消息调度

当消息到达预定执行时间，消息所在Looper会调用`msg.target.dispatchMessage(msg)`：

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // 由Message.callback消费事件；
        handleCallback(msg);
    } else {
        // 由Handler.mCallback消费事件；
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 事件没被消费，交给Handler.handleMessage(msg).
        handleMessage(msg);
    }
}
```

### 6.2 消息回调

(1) `dispatchMessage(msg)`首先尝试执行消息体的`msg.callback`。由于上面有`EmptyMessage`一类方法的存在，所以`msg.callback`可能为空并跳过。

```java
private static void handleCallback(Message message) {
    message.callback.run();
}
```

(2) `msg.callback`不行就看看Handler自己有没有`mCallback`

```java
public interface Callback {
    public boolean handleMessage(Message msg);
}
```

例子: 创建Handler时可以实现这个回调，支持操作主线程

```java
Handler handler = new Handler(new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        return false;
        Toast.makeText(Activity.this,"handleMessage override",Toast.LENGTH_SHORT).show();
    }
});
```

(3) 如果上两个回调都不存在，最终交给Handler类的handleMessage(Message msg)方法：

```java
@Override
public void handleMessage(Message msg) {
    super.handleMessage(msg);

    int what = msg.what;
    switch(what){
        case START_ACTIVITY:
            Intent intent = new Intent(Activity.this, MainActivity.class);
            Activity.this.startActivity(intent);
            break;

        case TOAST_SHORT_SHOW:
            Toast.makeText(Activity.this, "Toast", Toast.LENGTH_SHORT).show();
            break;
    }
}
```

# 七、移除消息

根据消息身份`what`、消息`Runnable`或`msg.obj`移除队列中对应的消息。

例如发送`msg`，用同一个`msg.what`作为参数。所有方法最终调用`MessageQueue.removeMessages`，具体在`MessageQueue`的源码阅读里面说。

```java
public final void removeCallbacks(Runnable r) {
    mQueue.removeMessages(this, r, null);
}

public final void removeCallbacks(Runnable r, Object token) {
    mQueue.removeMessages(this, r, token);
}

public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}

public final void removeMessages(int what, Object object) {
    mQueue.removeMessages(this, what, object);
}

public final void removeCallbacksAndMessages(Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}
```


# 八、查找消息

查看对应消息是否存在

```java
public final boolean hasMessages(int what) {
    return mQueue.hasMessages(this, what, null);
}

public final boolean hasMessages(int what, Object object) {
    return mQueue.hasMessages(this, what, object);
}

public final boolean hasCallbacks(Runnable r) {
    return mQueue.hasMessages(this, r, null);
}
```

# 九、阻塞非安全执行

如果当前执行线程是Handler的线程，Runnable会被立刻执行。否则把它放在消息队列中一直等待执行完毕或者超时。超时后这个任务还是在队列中，在后面的某个时刻它仍然会执行，很有可能造成死锁，所以尽量不要用它。

这个方法使用场景是Android初始化一个WindowManagerService，因为WindowManagerService不成功，其他组件就不允许继续，所以使用阻塞的方式直到完成。

```java
public final boolean runWithScissors(final Runnable r, long timeout) {
    if (r == null) {
        throw new IllegalArgumentException("runnable must not be null");
    }
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout must be non-negative");
    }

    // Looper相同，立即执行Runnable
    if (Looper.myLooper() == mLooper) {
        r.run();
        return true;
    }
    
    // 一个阻塞的队列
    BlockingRunnable br = new BlockingRunnable(r);
    return br.postAndWait(this, timeout);
}
```

```java
IMessenger mMessenger;  // IPC

private static final class BlockingRunnable implements Runnable {
    private final Runnable mTask;
    private boolean mDone;

    public BlockingRunnable(Runnable task) {
        mTask = task;
    }

    @Override
    public void run() {
        try {
            mTask.run();
        } finally {
            synchronized (this) {
                mDone = true;
                notifyAll();
            }
        }
    }

    public boolean postAndWait(Handler handler, long timeout) {
        // 向Handler提交Runnable任务，提交失败返回false
        if (!handler.post(this)) {
            return false;
        }

        synchronized (this) {
            if (timeout > 0) {
                // 截止时间
                final long expirationTime = SystemClock.uptimeMillis() + timeout;
                while (!mDone) {
                    // 已经超过截止时间，有delay <= 0
                    long delay = expirationTime - SystemClock.uptimeMillis();
                    if (delay <= 0) {
                        return false; // timeout
                    }
                    try {
                        wait(delay);
                    } catch (InterruptedException ex) {
                    }
                }
            } else {
                while (!mDone) {
                    try {
                        wait();
                    } catch (InterruptedException ex) {
                    }
                }
            }
        }
        return true;
    }
}
```

# 十、获取消息名

获取消息里Handler的类名，或消息msg.what的16进制值。如果我们开始就使用16进制设置，这里不用换算就能对应起来。

```java
public String getMessageName(Message message) {
    if (message.callback != null) {
        return message.callback.getClass().getName();
    }
    return "0x" + Integer.toHexString(message.what);
}
```

获取空消息体的重载方法

```java
public final Message obtainMessage() {
    return Message.obtain(this);
}

public final Message obtainMessage(int what) {
    return Message.obtain(this, what);
}

public final Message obtainMessage(int what, Object obj) {
    return Message.obtain(this, what, obj);
}

public final Message obtainMessage(int what, int arg1, int arg2) {
    return Message.obtain(this, what, arg1, arg2);
}

public final Message obtainMessage(int what, int arg1, int arg2, Object obj) {
    return Message.obtain(this, what, arg1, arg2, obj);
}
```

# 十一、其他

剩下这个方法关于跨进程通讯的Messager，在AIDL中使用。

```java
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

