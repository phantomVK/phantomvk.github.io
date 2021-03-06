---
layout:     post
title:      "Android源码系列(12) -- SharedPreferences"
date:       2018-09-14
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

# 一、概述

首先获取 __SharedPreferences__ 实例，调用 __sp.edit()__ 获得可编辑实例，写入数据并提交。

```java
SharedPreferences sp = getSharedPreferences("PrefsName", MODE_PRIVATE);

SharedPreferences.Editor editor = sp.edit();
editor.putInt("Int", 1024);
editor.putLong("Long", 47526348576L);
editor.putFloat("Float", 3.1415926F);
editor.putBoolean("Boolean", true);
editor.putString("String", "SharedPreferences String");
editor.apply();
```

__Android Studio__ 右下角有 __Device File Explorer__，按照以下路径找出保存的 `<PrefsName>.xml` 。文件名为 __getSharedPreferences("PrefsName", MODE_PRIVATE)__ 的实参值。

`/data/data/<Application Package Name>/shared_prefs/<PrefsName>.xml`

实际写入文件内容:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="Int" value="1024" />
    <float name="Float" value="3.1415926" />
    <boolean name="Boolean" value="true" />
    <long name="Long" value="47526348576" />
    <string name="String">SharedPreferences String</string>
</map>
```

# 二、源码剖析

## 2.1 SharedPreferences接口

#### 2.1.1 SharedPreferences成员

__SharedPreferences__ 是接口，源码在 __/frameworks/base/core/java/android/content__

读写由 __getSharedPreferences__ 返回的数据。修改操作需通过 __Editor__ 对象，以保证数据一致性并控制回写时机。不建议作为IPC使用，但相同进程的不同线程调用时线程安全。

```java
public interface SharedPreferences {

    // SharedPreference变化事件监听器，后面解释
    public interface OnSharedPreferenceChangeListener { ... }

    // 编辑SharedPreference的Editor
    public interface Editor { ... }

    // 从preferences获得所有值，且不得修改这个Map的数据，否则会导致已存储数据不一致
    Map<String, ?> getAll();

    // 获取key对应的String，若key不存在则返回defValue
    // 如果key存在但不是String类型，则直接抛出ClassCastException
    @Nullable
    String getString(String key, @Nullable String defValue);
    
    // 通过key获取字符串集合，且不得修改返回的数据，否则会导致数据不一致的问题
    @Nullable
    Set<String> getStringSet(String key, @Nullable Set<String> defValues);
    
    // 获取key对应的int，若key不存在则返回defValue
    // 如果key存在但不是int类型，则直接抛出ClassCastException
    int getInt(String key, int defValue);
    
    // 获取key对应的long，若key不存在则返回defValue
    // 如果key存在但不是long类型，则直接抛出ClassCastException
    long getLong(String key, long defValue);
    
    // 获取key对应的float，若key不存在则返回defValue
    // 如果key存在但不是float类型，则直接抛出ClassCastException
    float getFloat(String key, float defValue);

    // 获取key对应的boolean，若key不存在则返回defValue
    // 如果key存在但不是boolean类型，则直接抛出ClassCastException
    boolean getBoolean(String key, boolean defValue);

    // 检查是否存在此键代表的preference
    boolean contains(String key);

    // 给preferences创建新Editor
    Editor edit();
    
    // 注册事件监听器
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    
    // 移除事件监听器
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```

#### 2.1.2 OnSharedPreferenceChangeListener接口

__OnSharedPreferenceChangeListener__ 是 __SharedPreferences__ 的内部接口，当 __SharedPreference__ 发生变化时回调的监听器，方法执行在主线程

```java
public interface OnSharedPreferenceChangeListener {
    // @param sharedPreferences 发生变化的SharedPreferences
    // @param key 发生修改、新增、移除的key
    void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
}
```

#### 2.1.3 Editor接口

__Editor__ 是 __SharedPreferences__ 的内部接口，用于修改 __SharedPreferences__ 的值。所有操作均为批处理，只在调用 __commit()__ 或 __apply()__ 后才回写到磁盘

```java
public interface Editor {
    // 向编辑器设置String类型键值对，并在commit或apply方法调用时进行回写
    Editor putString(String key, @Nullable String value);

    // 向编辑器设置String类型键值对集合，并在commit或apply方法调用时进行回写
    Editor putStringSet(String key, @Nullable Set<String> values);

    // 向编辑器设置int类型键值对，并在commit或apply方法调用时进行回写
    Editor putInt(String key, int value);

    // 向编辑器设置long类型键值对，并在commit或apply方法调用时进行回写
    Editor putLong(String key, long value);

    // 向编辑器设置float类型键值对，并在commit或apply方法调用时进行回写
    Editor putFloat(String key, float value);

    // 向编辑器设置boolean类型键值对，并在commit或apply方法调用时进行回写
    Editor putBoolean(String key, boolean value);

    // 移除编辑器中指定的key，并在commit或apply方法调用时进行回写
    // 移除操作在其他操作之前，即不管移除操作是否在添加之后调用，都会优先回写
    Editor remove(String key);

    // 清除所有值，并在commit或apply方法调用时进行回写
    // 移除操作在其他操作之前，即不管移除操作是否在添加之后调用，都会优先回写
    Editor clear();

    // 执行所有preferences修改，当两个editor同时修改preferences，最后被调用的总能被执行
    // 如果不关心执行的结果值，且在主线程使用，建议通过apply提交修改
    // true提交并修改成功，false表示失败
    boolean commit();

    // 执行所有preferences修改，当有两个editors同时修改preferences，最后一个被调用的总能被执行
    // apply所有提交保存在内存中，并异步回写，回写失败不会提示
    // 当apply还没完成，其他editor的commit操作会阻塞并等待异步回写完成
    // SharedPreferences进程内为单例且线程安全，不关心返回值时可用apply代替commit
    // apply的写入安全由Android Framework保证，不受组件生命周期或交互影响
    void apply();
}
```

## 2.2 SharedPreferencesImpl实现类

#### 2.2.1 类签名

类文件在 __/frameworks/base/core/java/android/app__ 中，是 __SharedPreferences__ 的实现类

```java
final class SharedPreferencesImpl implements SharedPreferences
```

#### 2.2.2 常量

```java
private static final Object CONTENT = new Object();
```

全同步时间超过此值(ms)时发出警告

```java
private static final long MAX_FSYNC_DURATION_MILLIS = 256;
```

#### 2.2.3 数据成员

锁顺序规则：

- 在 __EditorImpl.mLock__ 前先获取 __SharedPreferencesImpl.mLock__
- 在 __EditorImpl.mLock__ 前先获取 __mWritingToDiskLock__

```java
// 当前操作文件
private final File mFile;

// 已备份文件
private final File mBackupFile;

// 模式
private final int mMode;

// 锁
private final Object mLock = new Object();

// 磁盘写入锁
private final Object mWritingToDiskLock = new Object();

@GuardedBy("mLock")
private Map<String, Object> mMap;

@GuardedBy("mLock")
private int mDiskWritesInFlight = 0;

// 磁盘数据已加载标志位
@GuardedBy("mLock")
private boolean mLoaded = false;

@GuardedBy("mLock")
private StructTimespec mStatTimestamp;

@GuardedBy("mLock")
private long mStatSize;

// 监听器列表，通过WeakHashMap持有监听器
@GuardedBy("mLock")
private final WeakHashMap<OnSharedPreferenceChangeListener, Object> mListeners =
        new WeakHashMap<OnSharedPreferenceChangeListener, Object>();

// 当前内存状态，持续递增
@GuardedBy("this")
private long mCurrentMemoryStateGeneration;

// 提交到磁盘的最新内存状态
@GuardedBy("mWritingToDiskLock")
private long mDiskStateGeneration;

// 系统文件同步请求的次数
@GuardedBy("mWritingToDiskLock")
private final ExponentiallyBucketedHistogram mSyncTimes = new ExponentiallyBucketedHistogram(16);
private int mNumSync = 0;
```

#### 2.2.4 构造方法

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    startLoadFromDisk();
}
```

#### 2.2.5 成员方法

开始从磁盘加载

```java
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }

    // 从子线程加载数据到内存
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

从子线程调用方法 __loadFromDisk()__ 

```java
private void loadFromDisk() {
    synchronized (mLock) {
        // 如果别的线程已经完成加载则直接跳出
        if (mLoaded) {
            return;
        }

        // 备份文件存在，删除原文件且把备份文件作为原文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    Map<String, Object> map = null;
    StructStat stat = null;
    try {
        stat = Os.stat(mFile.getPath());
        // 从文件读出数据
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16*1024);
                // 把xml转换为对应键值对
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                // 关闭流
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        /* ignore */
    }

    synchronized (mLock) {
        mLoaded = true;
        // 读取数据不为空，数据设置到mMap
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtim;
            mStatSize = stat.st_size;
        } else {
            // 磁盘没有已保存数据，创建新HashMap
            mMap = new HashMap<>();
        }
        // 释放锁，通知其他操作
        mLock.notifyAll();
    }
}
```

创建备份文件

```java
static File makeBackupFile(File prefsFile) {
    return new File(prefsFile.getPath() + ".bak");
}
```

出现意外时开始重新加载

```java
void startReloadIfChangedUnexpectedly() {
    synchronized (mLock) {
        if (!hasFileChangedUnexpectedly()) {
            return;
        }
        // 从磁盘重新读取数据
        startLoadFromDisk();
    }
}

// 文件被外部操作修改时执行的方法
private boolean hasFileChangedUnexpectedly() {
    synchronized (mLock) {
        if (mDiskWritesInFlight > 0) {
            // 已知是本程序主动引起的，不需要处理
            return false;
        }
    }

    final StructStat stat;
    try {
        // Metadata operations don't usually count as a block guard violation
        // but we explicitly want this one.
        BlockGuard.getThreadPolicy().onReadFromDisk();
        // 读取文件状态
        stat = Os.stat(mFile.getPath());
    } catch (ErrnoException e) {
        return true;
    }

    synchronized (mLock) {
        // 内存中记录磁盘文件的状态，和实际磁盘文件的状态进行对比
        return !stat.st_mtim.equals(mStatTimestamp) || mStatSize != stat.st_size;
    }
}
```

注册和移除监听器

```java
@Override
public void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
    synchronized(mLock) {
        // 向列表中加入监听器
        mListeners.put(listener, CONTENT);
    }
}

@Override
public void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
    synchronized(mLock) {
        // 从列表中移除监听器
        mListeners.remove(listener);
    }
}
```

阻塞等待文件内容加载完成

```java
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    
    // 阻塞等待数据加载完毕
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
}
```

获取所有键值对

```java
@Override
public Map<String, ?> getAll() {
    synchronized (mLock) {
        awaitLoadedLocked();
        return new HashMap<String, Object>(mMap);
    }
}
```

值获取

```java
// 获取字符串
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

// 获取字符串集合
@Override
@Nullable
public Set<String> getStringSet(String key, @Nullable Set<String> defValues) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Set<String> v = (Set<String>) mMap.get(key);
        return v != null ? v : defValues;
    }
}

// 获取int
@Override
public int getInt(String key, int defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Integer v = (Integer)mMap.get(key);
        return v != null ? v : defValue;
    }
}

// 获取long
@Override
public long getLong(String key, long defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Long v = (Long)mMap.get(key);
        return v != null ? v : defValue;
    }
}

// 获取float
@Override
public float getFloat(String key, float defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Float v = (Float)mMap.get(key);
        return v != null ? v : defValue;
    }
}

// 获取boolean
@Override
public boolean getBoolean(String key, boolean defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Boolean v = (Boolean)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

检查是否包含指定key

```java
@Override
public boolean contains(String key) {
    synchronized (mLock) {
        awaitLoadedLocked();
        return mMap.containsKey(key);
    }
}
```

阻塞等待加载完成，并返回编辑对象

```java
@Override
public Editor edit() {
    synchronized (mLock) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```

从内存写入磁盘任务的队列，顺序是FIFO依次执行：

- 参数postWriteRunnable非空执行apply()，writeToDiskRunnable完成后回调；
- 参数postWriteRunnable为空执行commit()，并允许数据在主线程写入磁盘；
- commit()可减少内存申请和后台回写线程，可通过StrictMode报告优化commit()为apply()；

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {

    // postWriteRunnable为空，同步提交数据到磁盘
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    // 创建磁盘写入Runnable
    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                // 写入文件，isFromSyncCommit是否同步写入
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }

                // 写入任务数量递减，因为本任务已经完成
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }

                // 仅apply()时执行，触发回调告知写入已完成
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // 更少内存申请的commit()方式，在当前线程写入
    // 因为子线程也占用内存，同线程写入能节约一个线程所需内存开销
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            // 值为1表示只有当前任务在执行，没有其他任务
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            // 直接执行完就退出
            // writeToDiskRunnable是Runnable，调动run()相当于调用普通方法
            writeToDiskRunnable.run();
            return;
        }
    }

    // 否则把任务放入工作队列按序执行
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

创建文件的输出流，即用于保存内容的流

```java
private static FileOutputStream createFileOutputStream(File file) {
    FileOutputStream str = null;
    try {
        // 创建文件输出流
        str = new FileOutputStream(file);
    } catch (FileNotFoundException e) {
        File parent = file.getParentFile();
        if (!parent.mkdir()) {
            Log.e(TAG, "Couldn't create directory for SharedPreferences file " + file);
            return null;
        }
        FileUtils.setPermissions(
            parent.getPath(),
            FileUtils.S_IRWXU|FileUtils.S_IRWXG|FileUtils.S_IXOTH,
            -1, -1);
        try {
            str = new FileOutputStream(file);
        } catch (FileNotFoundException e2) {
            Log.e(TAG, "Couldn't create SharedPreferences file " + file, e2);
        }
    }
    return str;
}
```

写入文件

```java
@GuardedBy("mWritingToDiskLock")
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;

    // 原文件是否存在
    boolean fileExists = mFile.exists();

    // 重命名当前文件，作为写入异常后还原备份
    if (fileExists) {
        boolean needsWrite = false;

        // 仅在磁盘状态比这次提交旧时写入
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            // commit()同步写入
            if (isFromSyncCommit) {
                // 需要写入
                needsWrite = true;
            } else {
                // apply()异步写入
                synchronized (mLock) {
                    // 中间状态不需要持久化，仅持久化最新状态
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }
        
        // 此次内存变更不需要进行磁盘回写
        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }

        // 备份文件是否存在
        boolean backupFileExists = mBackupFile.exists();

        // 磁盘文件备份不存在
        if (!backupFileExists) {
            // 把当前文件改名为备份文件
            if (!mFile.renameTo(mBackupFile)) {
                // 文件备份失败，新变更内容不得写入磁盘
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            // 备份文件已存在，删除源文件
            mFile.delete();
        }
    }

    // 尝试写入文件、删除备份和返回true时，尽可能做到原子性
    // 如果出现任何异常则删除新文件，下一次从备份文件中恢复
    try {
        // 创建文件输出流失败
        FileOutputStream str = createFileOutputStream(mFile);
        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }

        // 把内存中需要写入的数据按照xml格式写入到str流
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

        // 同步流
        FileUtils.sync(str);

        // 关闭文件写入流
        str.close();

        // 修改文件权限
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        // 获取物理文件状态并记录到内存中
        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
        }

        // 删除备份文件
        mBackupFile.delete();

        // 磁盘状态代值更新
        mDiskStateGeneration = mcr.memoryStateGeneration;

        mcr.setDiskWriteResult(true, true);

        // 同步次数递增
        mNumSync++;

        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }

    // 磁盘写文件操作出现异常，删除已写入文件
    // 下次读取内容发现mFile不存在，会检查备份文件
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```

#### 2.2.6 MemoryCommitResult

__MemoryCommitResult__ 是 __SharedPreferencesImpl__ 的内部类

```java
// 从EditorImpl.commitToMemory()返回值
private static class MemoryCommitResult {
    // 内存状态代
    final long memoryStateGeneration;
    
    // 被修改的keys列表
    @Nullable final List<String> keysModified;
    
    // 修改事件监听器集合
    @Nullable final Set<OnSharedPreferenceChangeListener> listeners;
    
    // 需要写磁盘的Map
    final Map<String, Object> mapToWriteToDisk;
    
    final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

    // 写磁盘操作是否成功
    @GuardedBy("mWritingToDiskLock")
    volatile boolean writeToDiskResult = false;
    
    // 是否有实际内容写入磁盘
    boolean wasWritten = false;

    private MemoryCommitResult(long memoryStateGeneration,
            @Nullable List<String> keysModified,
            @Nullable Set<OnSharedPreferenceChangeListener> listeners,
            Map<String, Object> mapToWriteToDisk) {
        this.memoryStateGeneration = memoryStateGeneration;
        this.keysModified = keysModified;
        this.listeners = listeners;
        this.mapToWriteToDisk = mapToWriteToDisk;
    }

    // wasWritten：已写入
    // result：结果
    void setDiskWriteResult(boolean wasWritten, boolean result) {
        this.wasWritten = wasWritten;
        writeToDiskResult = result;
        writtenToDiskLatch.countDown();
    }
}
```

#### 2.2.7 EditorImpl

__Editor__ 接口实现类，是 __SharedPreferencesImpl__ 的内部类

```java
public final class EditorImpl implements Editor {
    // 编辑器锁
    private final Object mEditorLock = new Object();

    // 已修改的记录，记录在内存中
    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();

    // 是否是干净的
    @GuardedBy("mEditorLock")
    private boolean mClear = false;

    // 存入String
    @Override
    public Editor putString(String key, @Nullable String value) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, value);
            return this;
        }
    }

    // 存入StringSet
    @Override
    public Editor putStringSet(String key, @Nullable Set<String> values) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key,
                    (values == null) ? null : new HashSet<String>(values));
            return this;
        }
    }

    // 存入int
    @Override
    public Editor putInt(String key, int value) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, value);
            return this;
        }
    }

    // 存入long
    @Override
    public Editor putLong(String key, long value) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, value);
            return this;
        }
    }

    // 存入float
    @Override
    public Editor putFloat(String key, float value) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, value);
            return this;
        }
    }

    // 存入boolean
    @Override
    public Editor putBoolean(String key, boolean value) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, value);
            return this;
        }
    }

    // 移除key
    @Override
    public Editor remove(String key) {
        synchronized (mEditorLock) {
            // 向HashMap记录最新修改
            mModified.put(key, this);
            return this;
        }
    }

    // 清空所有key
    @Override
    public Editor clear() {
        synchronized (mEditorLock) {
            // 设置清除标志位
            mClear = true;
            return this;
        }
    }

    @Override
    public void apply() {
        // 获取内存提交结果
        final MemoryCommitResult mcr = commitToMemory();
        final Runnable awaitCommit = new Runnable() {
                @Override
                public void run() {
                    try {
                        mcr.writtenToDiskLatch.await();
                    } catch (InterruptedException ignored) {
                    }
                }
            };

        QueuedWork.addFinisher(awaitCommit);

        Runnable postWriteRunnable = new Runnable() {
                @Override
                public void run() {
                    // 任务完成后移除Finisher
                    awaitCommit.run();
                    QueuedWork.removeFinisher(awaitCommit);
                }
            };
            
        // 提交到磁盘写入队列
        SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

        notifyListeners(mcr);
        // 这里不像commit()，不需要返回值
    }
    
    // 把mMap暂存的修改提交到mapToWriteToDisk，mapToWriteToDisk提交到磁盘
    private MemoryCommitResult commitToMemory() {
        long memoryStateGeneration;
        List<String> keysModified = null;
        Set<OnSharedPreferenceChangeListener> listeners = null;
        Map<String, Object> mapToWriteToDisk;

        synchronized (SharedPreferencesImpl.this.mLock) {
            // 一般不对mMap进行深拷贝，除非磁盘在回写时又有新内存变更提交
            if (mDiskWritesInFlight > 0) {
                // 磁盘写入正使用原始mMap，内存修改需要克隆获得新拷贝，再修改拷贝的内容
                mMap = new HashMap<String, Object>(mMap);
            }
            mapToWriteToDisk = mMap;
            // 磁盘待写入任务数递增
            mDiskWritesInFlight++;

            // 监听器数量
            boolean hasListeners = mListeners.size() > 0;
            if (hasListeners) {
                // 获取所有被修改的keys
                keysModified = new ArrayList<String>();
                // 获取所有注册的修改事件监听器
                listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
            }

            synchronized (mEditorLock) {
                // 有数据需要修改，则changesMade需设为true
                boolean changesMade = false;

                // 如果调用过clear()，则mClear为true
                if (mClear) {
                    // 需写入磁盘的数据不为空
                    if (!mapToWriteToDisk.isEmpty()) {
                        changesMade = true;
                        // 把需写入的数据全部置空
                        mapToWriteToDisk.clear();
                    }
                    mClear = false;
                }

                for (Map.Entry<String, Object> e : mModified.entrySet()) {
                    String k = e.getKey();
                    Object v = e.getValue();
                    // "this" is the magic value for a removal mutation. 
                    // v为空等同于remove(v)
                    if (v == this || v == null) {
                        // 检查需要写磁盘的map中是否包含这个key，没有则检查下一个key
                        if (!mapToWriteToDisk.containsKey(k)) {
                            continue;
                        }
                        mapToWriteToDisk.remove(k);
                    } else {
                        // 检查已修改的key是否需要加入到mapToWriteToDisk
                        if (mapToWriteToDisk.containsKey(k)) {
                            Object existingValue = mapToWriteToDisk.get(k);
                            // 新旧值相同，不需要更新到磁盘
                            if (existingValue != null && existingValue.equals(v)) {
                                continue;
                            }
                        }
                        mapToWriteToDisk.put(k, v);
                    }

                    changesMade = true;
                    if (hasListeners) {
                        keysModified.add(k);
                    }
                }

                // 已修改的列表置空，所有修改过的k-v均已加入mapToWriteToDisk
                // 而mapToWriteToDisk则准备被回写到磁盘中
                mModified.clear();
                
                // 内存状态版本递增
                if (changesMade) {
                    mCurrentMemoryStateGeneration++;
                }

                memoryStateGeneration = mCurrentMemoryStateGeneration;
            }
        }
        
        // 给apply()和commit()返回mcr
        return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
                mapToWriteToDisk);
    }

    // 如果SharedPreferences没有磁盘写入任务在执行，commit()直接执行在调用者线程
    // 否则commit()的任务会添加到任务队列等待执行，同时调用者线程阻塞等待任务完成
    @Override
    public boolean commit() {
        // 获取提交到内存的结果
        MemoryCommitResult mcr = commitToMemory();

        // 数据写入到磁盘，并在当前线程进行磁盘写入操作
        SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);

        try {
            // 等待所有磁盘写入任务完成，直接阻塞在调用者线程上
            mcr.writtenToDiskLatch.await();
        } catch (InterruptedException e) {
            return false;
        } finally {
        }

        // 发送通知
        notifyListeners(mcr);

        // 返回写入磁盘的结果
        return mcr.writeToDiskResult;
    }

    private void notifyListeners(final MemoryCommitResult mcr) {
        // 没有需要通知的监听器或没有被修改的key，则直接退出
        if (mcr.listeners == null || mcr.keysModified == null ||
            mcr.keysModified.size() == 0) {
            return;
        }

        // 已在主线程，直接通知所有监听器
        if (Looper.myLooper() == Looper.getMainLooper()) {
            for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
                // 逐个获取已修改的key
                final String key = mcr.keysModified.get(i);
                // 遍历所有监听器
                for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                    if (listener != null) {
                        // listener不为空，则发出通知
                        listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                    }
                }
            }
        } else {
            // 切换到主线程在进行通知
            ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
        }
    }
}
```

## 三、总结

由于 __SharedPreferences__ 写入是对文件的操作，即使修改其中一个值，整个记录文件都会重新写入。因此记录的内容越多，读写的时间也越长。虽然提供异步写入，但后台线程依然有可能因为其他IO操作的阻塞而执行较长时间，包括系统和其他正在运行应用所发起的IO操作。

读取指定键值对时，其他键值对会全部加载到内存中，保存所用的 __HashMap__ 也意味着更高的内存开销。考虑到 __SharedPreferences__ 由全局锁控制线程安全，获取 __SharedPreferencesImpl__ 的操作都是串行的。__Application__ 启动时最好少用它来读写值，避免减低应用冷启动的速度。