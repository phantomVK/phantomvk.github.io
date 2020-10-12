---
layout:     post
title:      "Android源码系列(16) -- MessageQueue"
date:       2018-11-02
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

# 一、类签名

__MessageQueue__ 是个低层次类，持有需要分发的消息。而消息并不是直接存入 __MessageQueue__，而是通过 __Looper__ 相对应 __Handler__ 加入。通过方法 __Looper.myQueue()__ 可以获取当前线程 __MessageQueue__。

```java
public final class MessageQueue
```

__MessageQueue__、__Looper__ 和 __Thread__ 的关系图解：

![MessageQueue](/img/android/images/MessageQueue.png)

学习 __MessageQueue__ 源码前，建议先学习 [Android源码系列(4) -- Handler](/2016/12/01/Android_Handler/) 和 [Android源码系列(5) -- Looper](/2016/12/03/Android_Looper)，有助于了解 __Message__ 如何在 __Handler__ 、__Looper__、__MessageQueue__ 三者间流动。

由于 __MessageQueue__ 是个低层次类，本次源码阅读只针对Java源码进行分析，没有涉及 __Android Framework__ 源码，以后会补全这部分知识。源码来自Android 28

## 二、数据成员

消息队列允许退出时为true

```java
private final boolean mQuitAllowed;
```

由C语言使用的变量（估计是指示消息队列在底层的ID）

```java
@SuppressWarnings("unused")
private long mPtr;
```

消息队列头消息，通过链表形成队列

```java
Message mMessages;
```

__IdleHandler__ 列表

```java
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
```

__FileDescriptorRecord__ 的稀疏阵列

```java
private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
```

__IdleHandler__ 数组

```java
private IdleHandler[] mPendingIdleHandlers;
```

消息队列是否正在退出

```java
private boolean mQuitting;
```

指示next()是否阻塞在非零超时值参数的pollOnce()调用上

```java
private boolean mBlocked;
```

下一个 __SyncBarrier__ 的token。Barriers就是target为空、arg1变量值为token的消息

```java
private int mNextBarrierToken;
```

## 三、原生方法

原生方法由C语言实现，源码在 __Android Framework__

```java
// 消息队列初始化
private native static long nativeInit();

// 销毁消息队列
private native static void nativeDestroy(long ptr);

// 从消息队列阻塞等待并获取一条消息
private native void nativePollOnce(long ptr, int timeoutMillis);

// 唤醒消息队列
private native static void nativeWake(long ptr);

// 检查消息队列是否在轮询
private native static boolean nativeIsPolling(long ptr);

// 注册文件描述符事件
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

## 四、构造方法

__mQuitAllowed__ 指示本消息队列是否允许退出，主线程的消息队列此值为 __false__。其他线程的消息队列一般可以退出。

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit(); // 初始化
}
```

## 五、成员方法

#### 5.1 finalize

```java
@Override
protected void finalize() throws Throwable {
    try {
        dispose(); // 调用下面方法
    } finally {
        super.finalize();
    }
}
```

销毁消息队列，仅能被looper所在线程或方法 __finalizer__ 调用。方法内调用原生方法销毁消息队列。

```java
private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);  // 调用原生方法
        mPtr = 0;
    }
}
```

#### 5.2 idle

当 __Looper__ 空闲时返回true，方法可在任何线程调用。

```java
public boolean isIdle() {
    synchronized (this) {
        // 当前时间戳
        final long now = SystemClock.uptimeMillis();
        return mMessages == null || now < mMessages.when;
    }
}
```

当方法 __IdleHandler.queueIdle()__ 被调用且返回 __false__ 时，__IdleHandler__ 会被自动移除

```java
public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}
```

从消息队列中移除之前已经添加的 __IdleHandler__，若指定 __IdleHandler__ 不存在则不进行操作。方法由 __synchronized__ 保护，可在任意线程调用

```java
public void removeIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
```

#### 5.3 polling

返回此looper的线程是否在等待获得更多工作。此方法同时表明loop依然存活。

```java
public boolean isPolling() {
    synchronized (this) {
        return isPollingLocked();
    }
}
```

在退出的消息队列一定不是空闲的。__mQuitting__ 为 false 时能假设 mPtr != 0

```java
private boolean isPollingLocked() {
    return !mQuitting && nativeIsPolling(mPtr);
}
```

#### 5.4 FileDescriptor

添加文件描述符监听器，并接受文件描述符相关事件发生的通知。

如果该文件描述符已经注册，则指定的事件和监听器会把旧的给替换掉，所以不可能给每个文件描述符添加多个监听器。文件描述符监听器不再使用时，需注销该监听器。

参数events是 __EVENT_INPUT__ 、__EVENT_OUTPUT__、 __EVENT_ERROR__ 值的掩码。如果events为0，则传入的监听器用于注销，而不是注册。

```java
public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
        @OnFileDescriptorEventListener.Events int events,
        @NonNull OnFileDescriptorEventListener listener) {

    // fd和listener均不能为空
    if (fd == null) {
        throw new IllegalArgumentException("fd must not be null");
    }

    if (listener == null) {
        throw new IllegalArgumentException("listener must not be null");
    }

    synchronized (this) {
        updateOnFileDescriptorEventListenerLocked(fd, events, listener);
    }
}
```

移除文件描述符监听器，指定对象不能为空

```java
public void removeOnFileDescriptorEventListener(@NonNull FileDescriptor fd) {
    
    // fd不能为空
    if (fd == null) {
        throw new IllegalArgumentException("fd must not be null");
    }

    synchronized (this) {
        // 传入events为0表示移除此事件上监听器
        updateOnFileDescriptorEventListenerLocked(fd, 0, null);
    }
}
```

注册文件描述符监听器

```java
private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
        OnFileDescriptorEventListener listener) {
    final int fdNum = fd.getInt$();

    int index = -1;
    FileDescriptorRecord record = null;
    if (mFileDescriptorRecords != null) {
        index = mFileDescriptorRecords.indexOfKey(fdNum);
        if (index >= 0) {
            record = mFileDescriptorRecords.valueAt(index);
            if (record != null && record.mEvents == events) {
                return;
            }
        }
    }

    if (events != 0) {
        events |= OnFileDescriptorEventListener.EVENT_ERROR;
        if (record == null) {
            if (mFileDescriptorRecords == null) {
                mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
            }
            record = new FileDescriptorRecord(fd, events, listener);
            mFileDescriptorRecords.put(fdNum, record);
        } else {
            record.mListener = listener;
            record.mEvents = events;
            record.mSeq += 1;
        }
        nativeSetFileDescriptorEvents(mPtr, fdNum, events);
    } else if (record != null) {
        record.mEvents = 0;
        mFileDescriptorRecords.removeAt(index);
        nativeSetFileDescriptorEvents(mPtr, fdNum, 0);
    }
}
```

此方法由原生代码代码调用

```java
private int dispatchEvents(int fd, int events) {
    // Get the file descriptor record and any state that might change.
    final FileDescriptorRecord record;
    final int oldWatchedEvents;
    final OnFileDescriptorEventListener listener;
    final int seq;
    synchronized (this) {
        record = mFileDescriptorRecords.get(fd);
        if (record == null) {
            return 0; // spurious, no listener registered
        }

        oldWatchedEvents = record.mEvents;
        events &= oldWatchedEvents; // filter events based on current watched set
        if (events == 0) {
            return oldWatchedEvents; // spurious, watched events changed
        }

        listener = record.mListener;
        seq = record.mSeq;
    }

    // 在锁之外调用监听器
    int newWatchedEvents = listener.onFileDescriptorEvents(
            record.mDescriptor, events);
    if (newWatchedEvents != 0) {
        newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
    }

    // Update the file descriptor record if the listener changed the set of
    // events to watch and the listener itself hasn't been updated since.
    if (newWatchedEvents != oldWatchedEvents) {
        synchronized (this) {
            int index = mFileDescriptorRecords.indexOfKey(fd);
            if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                    && record.mSeq == seq) {
                record.mEvents = newWatchedEvents;
                if (newWatchedEvents == 0) {
                    mFileDescriptorRecords.removeAt(index);
                }
            }
        }
    }

    // Return the new set of events to watch for native code to take care of.
    return newWatchedEvents;
}
```

#### 5.5 next

此方法为轮询消息队列上的消息。

**Looper** 会循环调用 **MessageQueue\.next()** 获取 **Message**。如果 **MessageQueue\.next()** 暂时空闲，则 **MessageQueue\.next()** 会触发一次 **IdleHandler** 回调，然后继续尝试直到返回有效 **Message** 给 **Looper** 并退出循环。

```java
Message next() {
    // 如果消息队列已经退出或销毁，在此处返回返回null
    // 这可能发生在应用尝试重启一个已经退出的looper
    final long ptr = mPtr;
    if (ptr == 0) {
        // 为0表示队列已失效，looper收到null
        return null;
    }

    // 待处理IdleHandler计数，在首次迭代中为-1
    int pendingIdleHandlerCount = -1;
    
    // 阻塞时间:
    // -1：一直阻塞等待消息； 
    // +0：不阻塞，立即返回；
    // >0：阻塞时间具体时长，如果有程序唤醒则立即返回；
    int nextPollTimeoutMillis = 0;
    
    // 循环
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        
        // 消息在nextPollTimeoutMillis后就绪，在此等待
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 尝试取下一条消息，并在成功后返回该消息
            // 获取当前时间戳，长度为手机已启动时间
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;  // 上一个消息
            Message msg = mMessages;

            // 消息体不为空，但是消息的Handler为空
            if (msg != null && msg.target == null) {
                // 该消息是SyncBarrier，则查找下一条队列中的异步消息
                // 找到的消息是异步消息，并执行
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }

            // 获取的消息不为空
            if (msg != null) {
                if (now < msg.when) {
                    // 下一条消息尚未就绪，设置超时并在消息就绪时唤醒
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 可获取消息，状态改为非阻塞
                    mBlocked = false;
                    // msg从队列中解除链接并出队
                    if (prevMsg != null) {
                        // msg是中间节点
                        prevMsg.next = msg.next;
                    } else {
                        // msg是头节点，msg下一个节点作为队头消息
                        mMessages = msg.next;
                    }
                    // 解除对下一个消息的引用
                    msg.next = null;
                    // 消息标记为FLAG_IN_USE
                    msg.markInUse();
                    // 返回该消息，交给Looper
                    return msg;
                    // 进入下一次轮询
                }
            } else {
                // 没有更多消息
                nextPollTimeoutMillis = -1;
            }

            // 队列已经标记为退出中，则调用队列的销毁方法
            if (mQuitting) {
                dispose();
                return null;
            }

            // 如果是首次闲置，获取闲置者数量
            // IdleHandlers仅在消息队列为空，或队列头消息在将来某个时间点才处理时运行
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                // 更新IdleHandler数量
                pendingIdleHandlerCount = mIdleHandlers.size();
            }

            if (pendingIdleHandlerCount <= 0) {
                // 没有可运行IdleHandler，进入下一次循环
                mBlocked = true;
                continue;
            }

            // pendingIdleHandlerCount > 0 且 mPendingIdleHandlers == null
            if (mPendingIdleHandlers == null) {
                // 构建数组，长度最小为4
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }

            // 从mIdleHandlers获取对象，放入mPendingIdleHandlers数组
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 执行IdleHandler，只在第一次迭代期间到达此代码块
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            // 获取下标对应IdleHandler
            final IdleHandler idler = mPendingIdleHandlers[i];
            // 释放数组对handler的引用
            mPendingIdleHandlers[i] = null;

            boolean keep = false;
            try {
                // 执行IdleHandler的queueIdle()得到执行值keep，该值由子类重写并返回
                // 由于子类重写queueIdle()，可在该方法内获得空闲状态的通知
                keep = idler.queueIdle();
            } catch (Throwable t) {
                // 捕获IdleHandler内异常，避免终止MessageQueue运行
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            // 检查是否需要移除该IdleHandler
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // 重置数量，以便不再运行它们
        pendingIdleHandlerCount = 0;

        // 处理IdleHandler时可能有新消息分发，所以下一轮直接获取消息不做阻塞等待
        nextPollTimeoutMillis = 0;
    }
}
```

如果 __next()__ 已给 __Looper__ 返回有效消息，__Looper__ 在消费 __Message__ 后会重新调用 __next()__，如下：

```java
public static void loop() {
    final Looper me = myLooper();

    // Looper没有通过prepare()方法初始化
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }

    // 从Looper中获取其MessageQueue
    final MessageQueue queue = me.mQueue;
    Binder.clearCallingIdentity(); // 确保线程就是本地线程，并实时跟踪线程身份
    final long ident = Binder.clearCallingIdentity();
    
    // 循环遍历，从消息队列取消息
    for (;;) {
        // 如果队列没有消息，queue.next()内部会阻塞等待
        Message msg = queue.next();

        // 队列返回null表明消息队列已经关闭，退出loop()的死循环
        if (msg == null) {
            return; // 消息队列关闭，Looper退出
        }

        // 通过对应的Handler进行回调
        msg.target.dispatchMessage(msg);

        // 确保消息在分发的时候线程没有改变
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        // 消息体回收
        msg.recycleUnchecked();
    }
}
```

 虽然 __MessageQueue__ 的 __next()__ 方法内 __IdleHandlers__ 只触发一次，但是由于 __Looper__ 循环调用 __MessageQueue.next()__，所以 __IdleHandlers__ 在空闲时间能被持续通知。

#### 5.6 quit

退出消息队列。如果消息队列设置为不允许退出，调用此方法会抛出 __IllegalStateException__ 异常。典型实现是主线程不允许退出，因为退出会导致消息不能进入主线程的消息队列。

若 __safe__ 值为true，消息队列会把所有到达执行时间点的消息全部处理完毕再退出，抛弃没有到达时间点的消息。否则所有消息直接抛弃，不管是否需要执行。

```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        // 如果消息队列正在退出，则无需执行第二遍退出
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            // 可能会有消息需要继续分发
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

#### 5.7 SyncBarrier

通过此方法向消息队列提交 __SyncBarrier__。除非遇到已经提交的 __SyncBarrier__，否则消息像往常一样正常处理。遇到 __SyncBarrier__ 时，在 __SyncBarrier__ 后面的同步消息会被阻塞并无法执行。直到调用方法 __removeSyncBarrier__，即用 __token__ 把匹配的 __SyncBarrier__ 移除，则同步消息继续分发。

此方法用于立即阻止后续同步消息执行，直到消息队列收到释放 __SyncBarrier__ 的条件。而非同步消息(异步消息)不会受到 __SyncBarrier__ 影响，保持正常执行。

每次 __postSyncBarrier()__ 会返回对应的、独一无二的 __token__，必要时需用该token调用 __removeSyncBarrier__ 移除已插入 __SyncBarrier__ 以保证消息队列重新运转，不然会导致消息队列持续挂起。

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
```

新 __SyncBarrier__ 进入队列，且不需唤醒队列，因为 __SyncBarrier__ 就是让队列停下。

```java
private int postSyncBarrier(long when) {
    synchronized (this) {
        // 获取下一个SyncBarrier的token
        final int token = mNextBarrierToken++;
        // 从缓存池获取空Message
        final Message msg = Message.obtain();
        // 初始化参数，且Message.target为空
        msg.markInUse();
        // 阻塞when时间戳之后的同步事件，小于when的同步事件在msg之前不受阻塞
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;

        // 查找正确队列位置插入
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }

        // when不为0，令prev != null，所以消息插入到消息队列中间；
        // 排在SyncBarrier前的同步Message不受影响，仅后方同步Message受影响；
        // Message异步消息永远不受SyncBarrier影响；
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            // when为0，令prev == null，所以直接插入到消息队列头部
            msg.next = p;
            mMessages = msg;
        }
        // 返回插入成功SyncBarrier的token
        return token;
    }
}
```

根据 __token__ 移除 __SyncBarrier__。当找不到对应 __SyncBarrier__ 时抛出 __IllegalStateException__ 异常。

```java
public void removeSyncBarrier(int token) {
    // 如果队列不受SyncBarrier阻塞，并重新唤醒队列
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;

        // 查找该目标SyncBarrier
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        
        // 找不到目标SyncBarrier，抛出IllegalStateException异常
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        
        // 找到目标SyncBarrier，needWake决定是否需要唤醒消息队列
        final boolean needWake;
        if (prev != null) {
            // p从消息队列中解除链接
            prev.next = p.next;
            // SyncBarrier前有消息可取，不需唤醒队列
            needWake = false;
        } else {
            mMessages = p.next;
            // 下一个消息为空，或消息的target不为空，需要唤醒队列
            // 注：消息的target不为空，表示下一个消息不是SyncBarrier
            needWake = mMessages == null || mMessages.target != null;
        }
        // 回收Message
        p.recycleUnchecked();

        // 循环在退出时消息队列已经唤醒，则取消唤醒
        if (needWake && !mQuitting) {
            // mQuitting 为 false，可以假设mPtr != 0
            nativeWake(mPtr);
        }
    }
}
```

#### 5.8 Messages

消息在从 __Looper__ 送入消息队列

```java
boolean enqueueMessage(Message msg, long when) {
    // Message.target为空，即Message的Handler为空，抛出IllegalArgumentException异常
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    // 消息已在使用，抛出IllegalStateException异常
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        // 消息队列在退出时有新消息进入队列，抛弃该消息并打出日志
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            // 回收该消息
            msg.recycle();
            return false;
        }

        // 消息正在使用
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;

        // p == null: 消息队列为空
        // when == 0: 新插入的消息需要马上执行
        // when < p.when: 插入消息的时间戳小于队列头部消息的时间戳
        // 上述三种情况，都需要把新消息插入到消息队列的首部
        if (p == null || when == 0 || when < p.when) {
            // 头插法插入新消息，并唤醒事件队列
            msg.next = p;
            mMessages = msg;
            // 若队列已阻塞，尝试唤醒
            needWake = mBlocked;
        } else {
            // 消息插入对队列中间而不是头部；
            // 一般是不需要唤醒消息队列的；
            // 除非队列头部有SyncBarrier(即 p.target == null)
            // 且msg是消息队列中最早的异步消息(即 msg.isAsynchronous())
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                // 查找合适的插入位置
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            // 消息插入到p之前，prev之后
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        
        // 执行到这里mQuitting肯定为false，所以能假设mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

检查是否有指定消息

```java
boolean hasMessages(Handler h, int what, Object object) {
    if (h == null) {
        return false;
    }

    synchronized (this) {
        Message p = mMessages;
        while (p != null) {
            if (p.target == h && p.what == what && (object == null || p.obj == object)) {
                return true;
            }
            p = p.next;
        }
        return false;
    }
}
```

检查是否有指定消息

```java
boolean hasMessages(Handler h, Runnable r, Object object) {
    if (h == null) {
        return false;
    }

    synchronized (this) {
        Message p = mMessages;
        while (p != null) {
            // 匹配Message.target、Message.callback、Message.obj三个变量
            if (p.target == h && p.callback == r && (object == null || p.obj == object)) {
                return true;
            }
            // 检查下一个消息
            p = p.next;
        }
        // 没有找到匹配消息
        return false;
    }
}
```

查找是否有目标为指定 __Handler__ 的消息

```java
boolean hasMessages(Handler h) {
    if (h == null) {
        return false;
    }

    synchronized (this) {
        Message p = mMessages;
        while (p != null) {
            // 匹配Message.target是否为执行Handler
            if (p.target == h) {
                return true;
            }
            // 检查下一个消息
            p = p.next;
        }
        return false;
    }
}
```

移除指定消息

```java
void removeMessages(Handler h, int what, Object object) {
    // Handler为空则直接退出
    if (h == null) {
        return;
    }

    synchronized (this) {
        Message p = mMessages;

        // 从消息队列头结点开始匹配，如果队列头连续几个消息匹配并移除，需修改mMessages的引用
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n; // 这里处理消息头引用
            p.recycleUnchecked(); // 消息直接回收到Message缓存池
            p = n;
        }

        // 匹配消息队列后面节点
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

移除指定消息

```java
void removeMessages(Handler h, Runnable r, Object object) {
    if (h == null || r == null) {
        return;
    }

    synchronized (this) {
        Message p = mMessages;

        // 从消息队列头结点开始匹配，如果队列头连续几个消息匹配并移除，需修改mMessages的引用
        while (p != null && p.target == h && p.callback == r
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n; // 这里处理消息头引用
            // 匹配消息p从队列中移除
            p.recycleUnchecked();
            p = n;
        }

        // 匹配消息队列后面节点
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.callback == r
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    // 匹配消息n从队列中移除
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

通过消息的 __Handler__ 和 __Object__ 移除指定消息

```java
void removeCallbacksAndMessages(Handler h, Object object) {
    if (h == null) {
        return;
    }

    synchronized (this) {
        Message p = mMessages;

        // 从消息队列头结点开始匹配，如果队列头连续几个消息匹配并移除，需修改mMessages的引用
        while (p != null && p.target == h
                && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n; // 这里处理消息头引用
            p.recycleUnchecked();
            p = n;
        }

        // 匹配消息队列后面节点
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

在此方法调用前已经对mMessages上锁，然后清理队列所有消息

```java
private void removeAllMessagesLocked() {
    Message p = mMessages;
    // 循环处理并回收Message
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
```

移除所有到达时间点的消息分发执行，剩余消息全部回收

```java
private void removeAllFutureMessagesLocked() {
    // 获取当前时间戳
    final long now = SystemClock.uptimeMillis();

    // 队列头消息
    Message p = mMessages;

    if (p != null) {
        // 队列头部消息没到执行时间点，则移除所有消息
        // 因为队列后面的消息一定不会比头消息更早执行
        if (p.when > now) {
            removeAllMessagesLocked();
        } else {
            // 队列中有消息已经到达执行时间点
            Message n;
            // 遍历消息队列中的消息
            for (;;) {
                n = p.next;
                // n为空表示队列没有更多消息
                if (n == null) {
                    return;
                }
                // 到达执行时间点的消息不移除队列
                if (n.when > now) {
                    break;
                }
                p = n;
            }
            p.next = null;

            // 不到处理时间点的消息全部回收
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}
```

## 六、IdleHandler

用于发现线程在等待消息而阻塞的回调接口，当消息队列没有消息可以分发时调用本接口。

抽象方法 __queueIdle()__ 返回true，本 __IdleHandler__ 持续有效；返回false，抽象方法调用后 __IdleHandler__ 被移除。本方法可能会发生在尚有消息在队列中等待的情况，不过这些消息都安排在未来时间点分发

```java
public static interface IdleHandler {
    boolean queueIdle();
}
```

## 七、OnFileDescriptorEventListener

当文件描述符相关事件发生时，事件监听器被回调

```java
public interface OnFileDescriptorEventListener {
    // 输入事件：表示文件描述符已准备好进行输入操作，例如读取；
    // 监听器要从文件描述符读取所有有效数据，并返回true令监听器保持存活，或false移除监听器；
    // 对Socket来说，此事件表示监听器至少有一个等待接入的连接；
    // 此事件只会在已添加监听器，且指定此值情况下产生；
    public static final int EVENT_INPUT = 1 << 0;

    // 输出事件：表示文件描述符应准备好输入操作，例如写入；
    // 监听器需写入所有有效数据。如果没法一次性完成写入，返回true令监听器保持存活
    // 或返回false移除监听器，在需要写入数据时再次注册监听器；
    // 此事件只会在监听器已添加，且此值已经制定的情况下产生
    public static final int EVENT_OUTPUT = 1 << 1;

    // 错误时间：表明文件描述符遇到致命错误；
    // 文件描述符遇到错误有很多原因。一般是Socket的远端或管道关闭了连接
    // 即使监听器没有设置，本错误事件也会产生
    public static final int EVENT_ERROR = 1 << 2;

    @Retention(RetentionPolicy.SOURCE)
    @IntDef(flag = true, prefix = { "EVENT_" }, value = {
            EVENT_INPUT,
            EVENT_OUTPUT,
            EVENT_ERROR
    })
    public @interface Events {}

    // 当文件描述符收到事件时调用
    // @param fd 文件描述符
    // @param events 发生事件的掩码
    // 返回true表示继续观察事件，返回false表示注销此监听器
    @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
}
```

## 八、FileDescriptorRecord

```java
private static final class FileDescriptorRecord {
    public final FileDescriptor mDescriptor;
    public int mEvents;
    public OnFileDescriptorEventListener mListener;
    public int mSeq;

    public FileDescriptorRecord(FileDescriptor descriptor,
            int events, OnFileDescriptorEventListener listener) {
        mDescriptor = descriptor;
        mEvents = events;
        mListener = listener;
    }
}
```