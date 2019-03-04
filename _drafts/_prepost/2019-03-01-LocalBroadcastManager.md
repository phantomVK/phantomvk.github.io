---
layout:     post
title:      "LocalBroadcastManager"
date:       2019-03-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 类签名

同于在同进程内本地对象注册或发送广播的帮助类。相比全局广播有较多优点：

- 你能了解到正在广播的数据不会离开应用，无需担心泄漏隐私数据；
- 其他应用无法大宋他们的广播到你的应用中，因此不存在安全漏洞问题；
- 相比全局广播效能更高，因为本地广播无需通过IPC发送到其他进程中；

```java
/**
 * Helper to register for and send broadcasts of Intents to local objects
 * within your process.  This has a number of advantages over sending
 * global broadcasts with {@link android.content.Context#sendBroadcast}:
 * <ul>
 * <li> You know that the data you are broadcasting won't leave your app, so
 * don't need to worry about leaking private data.
 * <li> It is not possible for other applications to send these broadcasts to
 * your app, so you don't need to worry about having security holes they can
 * exploit.
 * <li> It is more efficient than sending a global broadcast through the
 * system.
 * </ul>
 */
public final class LocalBroadcastManager
```

## 记录

```java
private static final class ReceiverRecord {
    // 用于筛选目标广播的IntentFilter
    final IntentFilter filter;
    final BroadcastReceiver receiver;
    boolean broadcasting;
    boolean dead;

    ReceiverRecord(IntentFilter _filter, BroadcastReceiver _receiver) {
        filter = _filter;
        receiver = _receiver;
    }
}
```

```java
private static final class BroadcastRecord {
    // 发送广播中携带的数据
    final Intent intent;
    // 广播接收者
    final ArrayList<ReceiverRecord> receivers;

    BroadcastRecord(Intent _intent, ArrayList<ReceiverRecord> _receivers) {
        intent = _intent;
        receivers = _receivers;
    }
}
```

## 数据成员

```java
// ApplicationContext
private final Context mAppContext;

private final HashMap<BroadcastReceiver, ArrayList<ReceiverRecord>> mReceivers
        = new HashMap<>();
private final HashMap<String, ArrayList<ReceiverRecord>> mActions = new HashMap<>();

// 等待处理的广播的列表
private final ArrayList<BroadcastRecord> mPendingBroadcasts = new ArrayList<>();

// 有待处理广播的标志位
static final int MSG_EXEC_PENDING_BROADCASTS = 1;

private final Handler mHandler;

// 同步锁
private static final Object mLock = new Object();
// 保存同步锁单例的静态变量
private static LocalBroadcastManager mInstance;
```

## 单例

同一个进程中，所有线程共享同一个 __LocalBroadcastManager__ 实例。而 __LocalBroadcastManager__ 初始化时持有 __ApplicationContext__，显然其生命周期和整个进程相同。

```java
@NonNull
public static LocalBroadcastManager getInstance(@NonNull Context context) {
    synchronized (mLock) {
        if (mInstance == null) {
            // 持有ApplicationContext
            mInstance = new LocalBroadcastManager(context.getApplicationContext());
        }
        return mInstance;
    }
}
```

## 构造方法

构造方法内构造了 __Handler__ 实例，负责触发广播逻辑

```java
private LocalBroadcastManager(Context context) {
    mAppContext = context;
    // 在主线程回调
    mHandler = new Handler(context.getMainLooper()) {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_EXEC_PENDING_BROADCASTS:
                    // 执行队列中的广播
                    executePendingBroadcasts();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };
}
```

## 成员方法

#### 注册

```java
/**
 * Register a receive for any local broadcasts that match the given IntentFilter.
 *
 * @param receiver The BroadcastReceiver to handle the broadcast.
 * @param filter Selects the Intent broadcasts to be received.
 *
 * @see #unregisterReceiver
 */
public void registerReceiver(@NonNull BroadcastReceiver receiver,
        @NonNull IntentFilter filter) {
    // 先获取同步锁
    synchronized (mReceivers) {
        // 用传入的变量构造ReceiverRecord
        ReceiverRecord entry = new ReceiverRecord(filter, receiver);
        ArrayList<ReceiverRecord> filters = mReceivers.get(receiver);
        if (filters == null) {
            filters = new ArrayList<>(1);
            mReceivers.put(receiver, filters);
        }
        filters.add(entry);
        for (int i=0; i<filter.countActions(); i++) {
            String action = filter.getAction(i);
            ArrayList<ReceiverRecord> entries = mActions.get(action);
            if (entries == null) {
                entries = new ArrayList<ReceiverRecord>(1);
                mActions.put(action, entries);
            }
            entries.add(entry);
        }
    }
}
```

#### 注销

```java
/**
 * Unregister a previously registered BroadcastReceiver.  <em>All</em>
 * filters that have been registered for this BroadcastReceiver will be
 * removed.
 *
 * @param receiver The BroadcastReceiver to unregister.
 *
 * @see #registerReceiver
 */
public void unregisterReceiver(@NonNull BroadcastReceiver receiver) {
    synchronized (mReceivers) {
        final ArrayList<ReceiverRecord> filters = mReceivers.remove(receiver);
        if (filters == null) {
            return;
        }
        for (int i=filters.size()-1; i>=0; i--) {
            final ReceiverRecord filter = filters.get(i);
            filter.dead = true;
            for (int j=0; j<filter.filter.countActions(); j++) {
                final String action = filter.filter.getAction(j);
                final ArrayList<ReceiverRecord> receivers = mActions.get(action);
                if (receivers != null) {
                    for (int k=receivers.size()-1; k>=0; k--) {
                        final ReceiverRecord rec = receivers.get(k);
                        if (rec.receiver == receiver) {
                            rec.dead = true;
                            receivers.remove(k);
                        }
                    }
                    if (receivers.size() <= 0) {
                        mActions.remove(action);
                    }
                }
            }
        }
    }
}
```

#### 发送广播

通过 __Intent__ 发送广播

```java
/**
 * Broadcast the given intent to all interested BroadcastReceivers.  This
 * call is asynchronous; it returns immediately, and you will continue
 * executing while the receivers are run.
 *
 * @param intent The Intent to broadcast; all receivers matching this
 *     Intent will receive the broadcast.
 *
 * @see #registerReceiver
 *
 * @return Returns true if the intent has been scheduled for delivery to one or more
 * broadcast receivers.  (Note tha delivery may not ultimately take place if one of those
 * receivers is unregistered before it is dispatched.)
 */
public boolean sendBroadcast(@NonNull Intent intent) {
    synchronized (mReceivers) {
        final String action = intent.getAction();
        final String type = intent.resolveTypeIfNeeded(
                mAppContext.getContentResolver());
        final Uri data = intent.getData();
        final String scheme = intent.getScheme();
        final Set<String> categories = intent.getCategories();

        ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
        if (entries != null) {
            ArrayList<ReceiverRecord> receivers = null;
            for (int i=0; i<entries.size(); i++) {
                ReceiverRecord receiver = entries.get(i);

                if (receiver.broadcasting) {
                    continue;
                }

                int match = receiver.filter.match(action, type, scheme, data,
                        categories, "LocalBroadcastManager");
                if (match >= 0) {
                    if (receivers == null) {
                        receivers = new ArrayList<ReceiverRecord>();
                    }
                    receivers.add(receiver);
                    receiver.broadcasting = true;
                }
            }

            if (receivers != null) {
                for (int i=0; i<receivers.size(); i++) {
                    receivers.get(i).broadcasting = false;
                }
                mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                // 向消息队列放入消息，表示有广播可以分发
                if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                    mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                }
                return true;
            }
        }
    }
    return false;
}
```

发送同步广播，但如果 __Intent__ 有的接收者，则此方法会阻塞线程并直接分发广播。__sendBroadcast(Intent)__ 方法会把广播放入消息队列等候派发，这个方法会马上占用线程派发。

```java
/**
 * Like {@link #sendBroadcast(Intent)}, but if there are any receivers for
 * the Intent this function will block and immediately dispatch them before
 * returning.
 */
public void sendBroadcastSync(@NonNull Intent intent) {
    if (sendBroadcast(intent)) {
        executePendingBroadcasts();
    }
}
```

存在有效的广播时触发这个方法，向满足条件的接收者派发广播记录

```java
private void executePendingBroadcasts() {
    while (true) {
        final BroadcastRecord[] brs;
        synchronized (mReceivers) {
            // 获取待处理广播的数量
            final int N = mPendingBroadcasts.size();
            if (N <= 0) {
                return;
            }
            // 创建与待处理广播数量相同的数组
            brs = new BroadcastRecord[N];
            // 把所有待处理的广播赋值到新创建的数组中
            mPendingBroadcasts.toArray(brs);
            // 清除待处理广播的列表
            mPendingBroadcasts.clear();
        }
        // 遍历刚创建数组的广播事件
        for (int i=0; i<brs.length; i++) {
            // 逐个取出广播
            final BroadcastRecord br = brs[i];
            // 获取广播接收者的数量
            final int nbr = br.receivers.size();
            for (int j=0; j<nbr; j++) {
                final ReceiverRecord rec = br.receivers.get(j);
                // 把广播记录派发给所有事件接收者
                if (!rec.dead) {
                    rec.receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
}
```

## 参考链接

- [LocalBroadcastManager原理分析](https://gityuan.com/2017/04/23/local_broadcast_manager/)