---
layout:     post
title:      "Android源码系列(23) -- LocalBroadcastManager"
date:       2019-03-15
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

用于同进程本地对象注册或发送广播的帮助类。如果广播信息只需要在同进程内收发，则无需发送全局广播，仅发送本地广播即可。同一应用开启多进程，不能在不同进程间用本地广播进行通讯，因为不同进程持有不同 __LocalBroadcastManager__ 实例，每个实例间没有关联。

```java
public final class LocalBroadcastManager
```

相比全局广播，本地广播有以下优点：

- 正在广播的数据不会离开进程发送到外部，无需担心泄漏隐私数据；
- 其他应用无法发送他们的广播到你的应用中，因此不存在安全漏洞问题；
- 相比全局广播效率更高，因为本地广播无需通过IPC和其他进程交互；

## 二、记录

注册广播接收者

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

发送广播信息

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

## 三、数据成员

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

// Handler单例
private final Handler mHandler;

// 同步锁
private static final Object mLock = new Object();

// 保存同步锁单例的静态变量
private static LocalBroadcastManager mInstance;
```

## 四、实例

同一个进程中所有线程共享同一个 __LocalBroadcastManager__ 实例。而 __LocalBroadcastManager__ 初始化时持有 __ApplicationContext__，显然其生命周期和整个进程相同。

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

## 五、构造方法

构造方法内构造了 __Handler__ 实例，负责触发分派到的广播

```java
private LocalBroadcastManager(Context context) {
    mAppContext = context;

    // 构造Handler，在主线程回调
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

## 六、成员方法

#### 6.1 注册

注册一个匹配指定 __IntentFilter__ 时触发的 __BroadcastReceiver__。

```java
public void registerReceiver(@NonNull BroadcastReceiver receiver,
        @NonNull IntentFilter filter) {
    // 先获取同步锁
    synchronized (mReceivers) {
        // 用传入的变量构造ReceiverRecord
        ReceiverRecord entry = new ReceiverRecord(filter, receiver);
        // 从mReceivers查找该receiver是否已经存在记录
        ArrayList<ReceiverRecord> filters = mReceivers.get(receiver);
        // 记录为空，则创建新的filters
        if (filters == null) {
            filters = new ArrayList<>(1);
            mReceivers.put(receiver, filters);
        }
        // key为receiver，value为ReceiverRecord
        filters.add(entry);

        // 按照Action记录对应的ReceiverRecord
        for (int i=0; i<filter.countActions(); i++) {
            // 从filter逐个获取Action
            String action = filter.getAction(i);
            ArrayList<ReceiverRecord> entries = mActions.get(action);
            // 这个Action没有记录过，则创建新记录
            if (entries == null) {
                entries = new ArrayList<ReceiverRecord>(1);
                mActions.put(action, entries);
            }
            // key为Action，value为ReceiverRecord
            entries.add(entry);
        }
    }
}
```

#### 6.2 注销

注销已经注册的 __BroadcastReceiver__，所有该 __BroadcastReceiver__ 的 __filters__ 也会被移除。

```java
public void unregisterReceiver(@NonNull BroadcastReceiver receiver) {
    synchronized (mReceivers) {
        final ArrayList<ReceiverRecord> filters = mReceivers.remove(receiver);
        // 该Receiver没有注册过，结束方法的执行
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
                            // 标记该接收器已经失效
                            rec.dead = true;
                            // 从列表移除接收器
                            receivers.remove(k);
                        }
                    }
                  
                    // actions没有接收器，就直接把整个actions记录移除
                    if (receivers.size() <= 0) {
                        mActions.remove(action);
                    }
                }
            }
        }
    }
}
```

#### 6.3 发送广播

通过 __Intent__ 发送广播。此方法的调用是异步的，方法执行后会马上返回，而接收器也会同时执行。

```java
public boolean sendBroadcast(@NonNull Intent intent) {
    synchronized (mReceivers) {
        final String action = intent.getAction();
        final String type = intent.resolveTypeIfNeeded(
                mAppContext.getContentResolver());
        final Uri data = intent.getData();
        final String scheme = intent.getScheme();
        final Set<String> categories = intent.getCategories();

        // 根据发送的action，找出已经注册的接收者记录
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

发送同步广播，但如果发送的 __Intent__ 有接收者，则此方法会阻塞线程并直接分发广播。相比 __sendBroadcast(Intent)__ 方法把广播放入消息队列等候派发，这个方法会马上占用线程派发，直到派发工作完成。

```java
public void sendBroadcastSync(@NonNull Intent intent) {
    if (sendBroadcast(intent)) {
        executePendingBroadcasts();
    }
}
```

存在有效的广播时触发这个方法，向满足条件的接收者派发广播记录。整个派发过程都在主线程上进行，如果接收器处理逻辑耗时，会阻塞主线程。

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

## 七、参考链接

- [LocalBroadcastManager原理分析](https://gityuan.com/2017/04/23/local_broadcast_manager/)