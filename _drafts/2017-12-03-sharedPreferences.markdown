---
layout:     post
title:      "SharedPreferences源码剖析"
date:       2017-12-03
author:     "phantomVK"
header-img: "img/main_img.jpg"
tags:
    - Android
---

# 一、概述

## 1.1 简介

## 1.2 用法

首先获取`SharedPreferences`实例，调用`sp.edit()`获得可编辑实例，写入几个类型的数据并提交。

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

在`Android Studio`右下角有`Device File Explorer`，按照以下路径找出保存的`<PrefsName>.xml`。文件名为`getSharedPreferences("PrefsName", MODE_PRIVATE)`中的实参值。

`/data/data/<Application Package Name>/shared_prefs/<PrefsName>.xml`

实际写入的文件内容是这样的:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="Int" value="1024" />
    <float name="Float" value="3.1415925" />
    <boolean name="Boolean" value="true" />
    <long name="Long" value="47526348576" />
    <string name="String">SharedPreferences String</string>
</map>
```

简单看来，除了String类型的其他类型值实际上都保存为字符串。

## 1.3 架构

# 二、源码剖析

## 2.1 SharedPreferences抽象接口

`SharedPreferences`是一个接口，源码路径是`/frameworks/base/core/java/android/content`。


```java
// 读写由getSharedPreferences返回的数据。修改操作需通过Editor对象，以保证数据一致性和控制回写的时机
// 此类不能用在多进程中，可以在相同进程不同线程中同时调用线程安全
public interface SharedPreferences {

    // 当SharedPreference发生变化时回调的监听器
    public interface OnSharedPreferenceChangeListener {
        // 当SharedPreference发生修改、新增、移除时回调的方法，key为此时变化的键，方法回调在主线程
        // @param sharedPreferences 发生变化的SharedPreferences
        // @param key               值发生修改、新增、移除的键
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }

    // 修改SharedPreferences中值的接口，所有操作均为批处理，只在调用commit或apply后才回写到磁盘
    public interface Editor {
        // 向编辑器设置一个String类型的键值对，并在commit或apply方法调用时进行回写
        Editor putString(String key, @Nullable String value);

        // 向编辑器设置一个String类型的键值对集合，并在commit或apply方法调用时进行回写
        Editor putStringSet(String key, @Nullable Set<String> values);
        
        // 向编辑器设置一个int类型的键值对，并在commit或apply方法调用时进行回写
        Editor putInt(String key, int value);
        
        // 向编辑器设置一个long类型的键值对，并在commit或apply方法调用时进行回写
        Editor putLong(String key, long value);
        
        // 向编辑器设置一个float类型的键值对，并在commit或apply方法调用时进行回写
        Editor putFloat(String key, float value);
        
        // 向编辑器设置一个boolean类型的键值对，并在commit或apply方法调用时进行回写
        Editor putBoolean(String key, boolean value);

        // 移除编辑器中指定的key，并在commit或apply方法调用时进行回写
        // 移除操作在其他操作之前，即不管移除操作是否在添加之后调用，都会优先执行
        Editor remove(String key);

        // 清除所有值，并在commit或apply方法调用时进行回写
        // 移除操作在其他操作之前，即不管移除操作是否在添加之后调用，都会优先执行
        Editor clear();

        // 执行所有preferences修改
        // 当有两个editors同时修改preferences，最后一个被调用的总能被执行
        // 如果不关心执行的结果值，且在主线程使用，建议通过apply提交修改
        // true提交并修改成功，false表示失败
        boolean commit();

        // 执行所有preferences修改
        // 当有两个editors同时修改preferences，最后一个被调用的总能被执行
        // apply所有提交保存在内存中，并异步进行回写，即回写失败不会有任何提示
        // 当apply还没完成，其他editor的commit操作会阻塞并等待异步回写完成
        // SharedPreferences进程内线程安全，且在不关心返回值的情况下可用apply代替commit
        // apply的写入安全由Android Framework保证，不受组件生命周期或交互影响
        void apply();
    }

    // 从preferences获得所有值，且一定不能修改这个Map的数据，增删改都不行
    // 否则会出现已存储数据不一致的问题
    Map<String, ?> getAll();

    // 获取key对应的String，若key不存在则返回defValue
    // 如果key存在但不是String类型，则直接抛出ClassCastException
    @Nullable
    String getString(String key, @Nullable String defValue);
    
    // 通过key获取字符串集合，且一直能够不能修改返回的数据，否则会导致数据不一致的问题
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

    // 检查是否已包含此键代表的preference
    boolean contains(String key);

    // 给preferences创建多一个新的Editor，并使用commit提交修改
    Editor edit();
    
    // 新增一个未注册的监听器
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    
    // 移除一个已注册的监听器
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```

## 2.2 SharedPreferencesImpl实现类

类文件在`/frameworks/base/core/java/android/app`中

```java
final class SharedPreferencesImpl implements SharedPreferences {
    private static final String TAG = "SharedPreferencesImpl";
    private static final boolean DEBUG = false;
    private static final Object CONTENT = new Object();

    /** If a fsync takes more than {@value #MAX_FSYNC_DURATION_MILLIS} ms, warn */
    private static final long MAX_FSYNC_DURATION_MILLIS = 256;

    // Lock ordering rules:
    //  - acquire SharedPreferencesImpl.mLock before EditorImpl.mLock
    //  - acquire mWritingToDiskLock before EditorImpl.mLock

    private final File mFile;
    private final File mBackupFile;
    private final int mMode;
    private final Object mLock = new Object();
    private final Object mWritingToDiskLock = new Object();

    @GuardedBy("mLock")
    private Map<String, Object> mMap;

    @GuardedBy("mLock")
    private int mDiskWritesInFlight = 0;

    @GuardedBy("mLock")
    private boolean mLoaded = false;

    @GuardedBy("mLock")
    private StructTimespec mStatTimestamp;

    @GuardedBy("mLock")
    private long mStatSize;

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

    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }

    // 开始从磁盘加载
    private void startLoadFromDisk() {
        synchronized (mLock) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();
            }
        }.start();
    }

    private void loadFromDisk() {
        synchronized (mLock) {
            // 如果已经加载过，跳出
            if (mLoaded) {
                return;
            }
            // 备份文件存在，则把源文件删除，备份文件作为原文件
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
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            /* ignore */
        }

        synchronized (mLock) {
            mLoaded = true;
            // 读取的数据不为空，则把数据设置到mMap
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            } else {
                // 磁盘没有已保存数据，则创建空HashMap给mMap
                mMap = new HashMap<>();
            }
            mLock.notifyAll();
        }
    }
    
    // 创建备份文件
    static File makeBackupFile(File prefsFile) {
        return new File(prefsFile.getPath() + ".bak");
    }
    
    // 出现意外是开始重新加载
    void startReloadIfChangedUnexpectedly() {
        synchronized (mLock) {
            if (!hasFileChangedUnexpectedly()) {
                return;
            }
            startLoadFromDisk();
        }
    }

    // Has the file changed out from under us?  i.e. writes that
    // we didn't instigate.
    // 文件被外部操作修改了？
    private boolean hasFileChangedUnexpectedly() {
        synchronized (mLock) {
            if (mDiskWritesInFlight > 0) {
                // If we know we caused it, it's not unexpected.
                if (DEBUG) Log.d(TAG, "disk write in flight, not unexpected.");
                return false;
            }
        }

        final StructStat stat;
        try {
            /*
             * Metadata operations don't usually count as a block guard
             * violation, but we explicitly want this one.
             */
            BlockGuard.getThreadPolicy().onReadFromDisk();
            stat = Os.stat(mFile.getPath());
        } catch (ErrnoException e) {
            return true;
        }

        synchronized (mLock) {
            return !stat.st_mtim.equals(mStatTimestamp) || mStatSize != stat.st_size;
        }
    }

    @Override
    public void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
        synchronized(mLock) {
            mListeners.put(listener, CONTENT);
        }
    }

    @Override
    public void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
        synchronized(mLock) {
            mListeners.remove(listener);
        }
    }

    private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                mLock.wait();
            } catch (InterruptedException unused) {
            }
        }
    }
    
    // 获取所有
    @Override
    public Map<String, ?> getAll() {
        synchronized (mLock) {
            awaitLoadedLocked();
            //noinspection unchecked
            return new HashMap<String, Object>(mMap);
        }
    }

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
    
    // 获取Int
    @Override
    public int getInt(String key, int defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Integer v = (Integer)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    
    // 获取Long
    @Override
    public long getLong(String key, long defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Long v = (Long)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    
    // 获取Float
    @Override
    public float getFloat(String key, float defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Float v = (Float)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    
    // 获取Boolean
    @Override
    public boolean getBoolean(String key, boolean defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Boolean v = (Boolean)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    
    // 检查时候包含指定key
    @Override
    public boolean contains(String key) {
        synchronized (mLock) {
            awaitLoadedLocked();
            return mMap.containsKey(key);
        }
    }

    @Override
    public Editor edit() {
        // TODO: remove the need to call awaitLoadedLocked() when
        // requesting an editor.  will require some work on the
        // Editor, but then we should be able to do:
        //
        //      context.getSharedPreferences(..).edit().putString(..).apply()
        //
        // ... all without blocking.
        synchronized (mLock) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }

    // Return value from EditorImpl#commitToMemory()
    private static class MemoryCommitResult {
        final long memoryStateGeneration;
        @Nullable final List<String> keysModified;
        @Nullable final Set<OnSharedPreferenceChangeListener> listeners;
        final Map<String, Object> mapToWriteToDisk;
        final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

        @GuardedBy("mWritingToDiskLock")
        volatile boolean writeToDiskResult = false;
        boolean wasWritten = false;

        private MemoryCommitResult(long memoryStateGeneration, @Nullable List<String> keysModified,
                @Nullable Set<OnSharedPreferenceChangeListener> listeners,
                Map<String, Object> mapToWriteToDisk) {
            this.memoryStateGeneration = memoryStateGeneration;
            this.keysModified = keysModified;
            this.listeners = listeners;
            this.mapToWriteToDisk = mapToWriteToDisk;
        }

        void setDiskWriteResult(boolean wasWritten, boolean result) {
            this.wasWritten = wasWritten;
            writeToDiskResult = result;
            writtenToDiskLatch.countDown();
        }
    }

    public final class EditorImpl implements Editor {
        // 编辑器锁
        private final Object mEditorLock = new Object();
        
        // 需修改记录，记录在内存中
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
        
        // 存入Int
        @Override
        public Editor putInt(String key, int value) {
            synchronized (mEditorLock) {
                // 向HashMap记录最新修改
                mModified.put(key, value);
                return this;
            }
        }
        
        // 存入Long
        @Override
        public Editor putLong(String key, long value) {
            synchronized (mEditorLock) {
                // 向HashMap记录最新修改
                mModified.put(key, value);
                return this;
            }
        }
        
        // 存入Float
        @Override
        public Editor putFloat(String key, float value) {
            synchronized (mEditorLock) {
                // 向HashMap记录最新修改
                mModified.put(key, value);
                return this;
            }
        }
        
        // 存入Boolean
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
                mClear = true;
                return this;
            }
        }

        @Override
        public void apply() {
            final long startTime = System.currentTimeMillis();

            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }

                        if (DEBUG && mcr.wasWritten) {
                            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                                    + " applied after " + (System.currentTimeMillis() - startTime)
                                    + " ms");
                        }
                    }
                };

            QueuedWork.addFinisher(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    @Override
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.removeFinisher(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }

        // Returns true if any changes were made
        private MemoryCommitResult commitToMemory() {
            long memoryStateGeneration;
            List<String> keysModified = null;
            Set<OnSharedPreferenceChangeListener> listeners = null;
            Map<String, Object> mapToWriteToDisk;

            synchronized (SharedPreferencesImpl.this.mLock) {
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;

                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    keysModified = new ArrayList<String>();
                    listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }

                synchronized (mEditorLock) {
                    boolean changesMade = false;

                    if (mClear) {
                        if (!mapToWriteToDisk.isEmpty()) {
                            changesMade = true;
                            mapToWriteToDisk.clear();
                        }
                        mClear = false;
                    }

                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) {
                            if (!mapToWriteToDisk.containsKey(k)) {
                                continue;
                            }
                            mapToWriteToDisk.remove(k);
                        } else {
                            if (mapToWriteToDisk.containsKey(k)) {
                                Object existingValue = mapToWriteToDisk.get(k);
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

                    mModified.clear();

                    if (changesMade) {
                        mCurrentMemoryStateGeneration++;
                    }

                    memoryStateGeneration = mCurrentMemoryStateGeneration;
                }
            }
            return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
                    mapToWriteToDisk);
        }

        @Override
        public boolean commit() {
            long startTime = 0;

            if (DEBUG) {
                startTime = System.currentTimeMillis();
            }

            MemoryCommitResult mcr = commitToMemory();

            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            } finally {
                if (DEBUG) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " committed after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }

        private void notifyListeners(final MemoryCommitResult mcr) {
            if (mcr.listeners == null || mcr.keysModified == null ||
                mcr.keysModified.size() == 0) {
                return;
            }
            if (Looper.myLooper() == Looper.getMainLooper()) {
                for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
                    final String key = mcr.keysModified.get(i);
                    for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                        if (listener != null) {
                            listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                        }
                    }
                }
            } else {
                // 在主线程调用
                ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
            }
        }
    }

    /**
     * Enqueue an already-committed-to-memory result to be written
     * to disk.
     *
     * They will be written to disk one-at-a-time in the order
     * that they're enqueued.
     *
     * @param postWriteRunnable if non-null, we're being called
     *   from apply() and this is the runnable to run after
     *   the write proceeds.  if null (from a regular commit()),
     *   then we're allowed to do this disk write on the main
     *   thread (which in addition to reducing allocations and
     *   creating a background thread, this has the advantage that
     *   we catch them in userdebug StrictMode reports to convert
     *   them where possible to apply() ...)
     */
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final boolean isFromSyncCommit = (postWriteRunnable == null);

        final Runnable writeToDiskRunnable = new Runnable() {
                @Override
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr, isFromSyncCommit);
                    }
                    synchronized (mLock) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (mLock) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
    }

    private static FileOutputStream createFileOutputStream(File file) {
        FileOutputStream str = null;
        try {
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

        if (DEBUG) {
            startTime = System.currentTimeMillis();
        }

        boolean fileExists = mFile.exists();

        if (DEBUG) {
            existsTime = System.currentTimeMillis();

            // Might not be set, hence init them to a default value
            backupExistsTime = existsTime;
        }

        // Rename the current file so it may be used as a backup during the next read
        if (fileExists) {
            boolean needsWrite = false;

            // Only need to write if the disk state is older than this commit
            if (mDiskStateGeneration < mcr.memoryStateGeneration) {
                if (isFromSyncCommit) {
                    needsWrite = true;
                } else {
                    synchronized (mLock) {
                        // No need to persist intermediate states. Just wait for the latest state to
                        // be persisted.
                        if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                            needsWrite = true;
                        }
                    }
                }
            }

            if (!needsWrite) {
                mcr.setDiskWriteResult(false, true);
                return;
            }

            boolean backupFileExists = mBackupFile.exists();

            if (DEBUG) {
                backupExistsTime = System.currentTimeMillis();
            }

            if (!backupFileExists) {
                if (!mFile.renameTo(mBackupFile)) {
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    mcr.setDiskWriteResult(false, false);
                    return;
                }
            } else {
                mFile.delete();
            }
        }

        // Attempt to write the file, delete the backup and return true as atomically as
        // possible.  If any exception occurs, delete the new file; next time we will restore
        // from the backup.
        try {
            FileOutputStream str = createFileOutputStream(mFile);

            if (DEBUG) {
                outputStreamCreateTime = System.currentTimeMillis();
            }

            if (str == null) {
                mcr.setDiskWriteResult(false, false);
                return;
            }
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

            writeTime = System.currentTimeMillis();

            FileUtils.sync(str);

            fsyncTime = System.currentTimeMillis();

            str.close();
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

            if (DEBUG) {
                setPermTime = System.currentTimeMillis();
            }

            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (mLock) {
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }

            if (DEBUG) {
                fstatTime = System.currentTimeMillis();
            }

            // Writing was successful, delete the backup file if there is one.
            mBackupFile.delete();

            if (DEBUG) {
                deleteTime = System.currentTimeMillis();
            }

            mDiskStateGeneration = mcr.memoryStateGeneration;

            mcr.setDiskWriteResult(true, true);

            if (DEBUG) {
                Log.d(TAG, "write: " + (existsTime - startTime) + "/"
                        + (backupExistsTime - startTime) + "/"
                        + (outputStreamCreateTime - startTime) + "/"
                        + (writeTime - startTime) + "/"
                        + (fsyncTime - startTime) + "/"
                        + (setPermTime - startTime) + "/"
                        + (fstatTime - startTime) + "/"
                        + (deleteTime - startTime));
            }

            long fsyncDuration = fsyncTime - writeTime;
            mSyncTimes.add(Long.valueOf(fsyncDuration).intValue());
            mNumSync++;

            if (DEBUG || mNumSync % 1024 == 0 || fsyncDuration > MAX_FSYNC_DURATION_MILLIS) {
                mSyncTimes.log(TAG, "Time required to fsync " + mFile + ": ");
            }

            return;
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }

        // Clean up an unsuccessfully written file
        if (mFile.exists()) {
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        mcr.setDiskWriteResult(false, false);
    }
}
```