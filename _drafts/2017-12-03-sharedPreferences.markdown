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
/**
 * Interface for accessing and modifying preference data returned by {@link
 * Context#getSharedPreferences}.  For any particular set of preferences,
 * there is a single instance of this class that all clients share.
 * Modifications to the preferences must go through an {@link Editor} object
 * to ensure the preference values remain in a consistent state and control
 * when they are committed to storage.  Objects that are returned from the
 * various <code>get</code> methods must be treated as immutable by the application.
 *
 * <p><em>Note: This class does not support use across multiple processes.</em>
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 * <p>For more information about using SharedPreferences, read the
 * <a href="{@docRoot}guide/topics/data/data-storage.html#pref">Data Storage</a>
 * developer guide.</p></div>
 *
 * @see Context#getSharedPreferences
 */
public interface SharedPreferences {
    /**
     * Interface definition for a callback to be invoked when a shared
     * preference is changed.
     */
    public interface OnSharedPreferenceChangeListener {
        /**
         * Called when a shared preference is changed, added, or removed. This
         * may be called even if a preference is set to its existing value.
         *
         * <p>This callback will be run on your main thread.
         *
         * @param sharedPreferences The {@link SharedPreferences} that received
         *            the change.
         * @param key The key of the preference that was changed, added, or
         *            removed.
         */
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }
    
    /**
     * Interface used for modifying values in a {@link SharedPreferences}
     * object.  All changes you make in an editor are batched, and not copied
     * back to the original {@link SharedPreferences} until you call {@link #commit}
     * or {@link #apply}
     */
    public interface Editor {
        /**
         * Set a String value in the preferences editor, to be written back once
         * {@link #commit} or {@link #apply} are called.
         * 
         * @param key The name of the preference to modify.
         * @param value The new value for the preference.  Passing {@code null}
         *    for this argument is equivalent to calling {@link #remove(String)} with
         *    this key.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putString(String key, @Nullable String value);
        
        /**
         * Set a set of String values in the preferences editor, to be written
         * back once {@link #commit} or {@link #apply} is called.
         * 
         * @param key The name of the preference to modify.
         * @param values The set of new values for the preference.  Passing {@code null}
         *    for this argument is equivalent to calling {@link #remove(String)} with
         *    this key.
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putStringSet(String key, @Nullable Set<String> values);
        
        /**
         * Set an int value in the preferences editor, to be written back once
         * {@link #commit} or {@link #apply} are called.
         * 
         * @param key The name of the preference to modify.
         * @param value The new value for the preference.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putInt(String key, int value);
        
        /**
         * Set a long value in the preferences editor, to be written back once
         * {@link #commit} or {@link #apply} are called.
         * 
         * @param key The name of the preference to modify.
         * @param value The new value for the preference.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putLong(String key, long value);
        
        /**
         * Set a float value in the preferences editor, to be written back once
         * {@link #commit} or {@link #apply} are called.
         * 
         * @param key The name of the preference to modify.
         * @param value The new value for the preference.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putFloat(String key, float value);
        
        /**
         * Set a boolean value in the preferences editor, to be written back
         * once {@link #commit} or {@link #apply} are called.
         * 
         * @param key The name of the preference to modify.
         * @param value The new value for the preference.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor putBoolean(String key, boolean value);

        /**
         * Mark in the editor that a preference value should be removed, which
         * will be done in the actual preferences once {@link #commit} is
         * called.
         * 
         * <p>Note that when committing back to the preferences, all removals
         * are done first, regardless of whether you called remove before
         * or after put methods on this editor.
         * 
         * @param key The name of the preference to remove.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor remove(String key);

        /**
         * Mark in the editor to remove <em>all</em> values from the
         * preferences.  Once commit is called, the only remaining preferences
         * will be any that you have defined in this editor.
         * 
         * <p>Note that when committing back to the preferences, the clear
         * is done first, regardless of whether you called clear before
         * or after put methods on this editor.
         * 
         * @return Returns a reference to the same Editor object, so you can
         * chain put calls together.
         */
        Editor clear();

        /**
         * Commit your preferences changes back from this Editor to the
         * {@link SharedPreferences} object it is editing.  This atomically
         * performs the requested modifications, replacing whatever is currently
         * in the SharedPreferences.
         *
         * <p>Note that when two editors are modifying preferences at the same
         * time, the last one to call commit wins.
         *
         * <p>If you don't care about the return value and you're
         * using this from your application's main thread, consider
         * using {@link #apply} instead.
         *
         * @return Returns true if the new values were successfully written
         * to persistent storage.
         */
        boolean commit();

        /**
         * Commit your preferences changes back from this Editor to the
         * {@link SharedPreferences} object it is editing.  This atomically
         * performs the requested modifications, replacing whatever is currently
         * in the SharedPreferences.
         *
         * <p>Note that when two editors are modifying preferences at the same
         * time, the last one to call apply wins.
         *
         * <p>Unlike {@link #commit}, which writes its preferences out
         * to persistent storage synchronously, {@link #apply}
         * commits its changes to the in-memory
         * {@link SharedPreferences} immediately but starts an
         * asynchronous commit to disk and you won't be notified of
         * any failures.  If another editor on this
         * {@link SharedPreferences} does a regular {@link #commit}
         * while a {@link #apply} is still outstanding, the
         * {@link #commit} will block until all async commits are
         * completed as well as the commit itself.
         *
         * <p>As {@link SharedPreferences} instances are singletons within
         * a process, it's safe to replace any instance of {@link #commit} with
         * {@link #apply} if you were already ignoring the return value.
         *
         * <p>You don't need to worry about Android component
         * lifecycles and their interaction with <code>apply()</code>
         * writing to disk.  The framework makes sure in-flight disk
         * writes from <code>apply()</code> complete before switching
         * states.
         *
         * <p class='note'>The SharedPreferences.Editor interface
         * isn't expected to be implemented directly.  However, if you
         * previously did implement it and are now getting errors
         * about missing <code>apply()</code>, you can simply call
         * {@link #commit} from <code>apply()</code>.
         */
        void apply();
    }

    /**
     * Retrieve all values from the preferences.
     *
     * <p>Note that you <em>must not</em> modify the collection returned
     * by this method, or alter any of its contents.  The consistency of your
     * stored data is not guaranteed if you do.
     *
     * @return Returns a map containing a list of pairs key/value representing
     * the preferences.
     *
     * @throws NullPointerException
     */
    Map<String, ?> getAll();

    /**
     * Retrieve a String value from the preferences.
     * 
     * @param key The name of the preference to retrieve.
     * @param defValue Value to return if this preference does not exist.
     * 
     * @return Returns the preference value if it exists, or defValue.  Throws
     * ClassCastException if there is a preference with this name that is not
     * a String.
     * 
     * @throws ClassCastException
     */
    @Nullable
    String getString(String key, @Nullable String defValue);
    
    /**
     * Retrieve a set of String values from the preferences.
     * 
     * <p>Note that you <em>must not</em> modify the set instance returned
     * by this call.  The consistency of the stored data is not guaranteed
     * if you do, nor is your ability to modify the instance at all.
     *
     * @param key The name of the preference to retrieve.
     * @param defValues Values to return if this preference does not exist.
     * 
     * @return Returns the preference values if they exist, or defValues.
     * Throws ClassCastException if there is a preference with this name
     * that is not a Set.
     * 
     * @throws ClassCastException
     */
    @Nullable
    Set<String> getStringSet(String key, @Nullable Set<String> defValues);
    
    /**
     * Retrieve an int value from the preferences.
     * 
     * @param key The name of the preference to retrieve.
     * @param defValue Value to return if this preference does not exist.
     * 
     * @return Returns the preference value if it exists, or defValue.  Throws
     * ClassCastException if there is a preference with this name that is not
     * an int.
     * 
     * @throws ClassCastException
     */
    int getInt(String key, int defValue);
    
    /**
     * Retrieve a long value from the preferences.
     * 
     * @param key The name of the preference to retrieve.
     * @param defValue Value to return if this preference does not exist.
     * 
     * @return Returns the preference value if it exists, or defValue.  Throws
     * ClassCastException if there is a preference with this name that is not
     * a long.
     * 
     * @throws ClassCastException
     */
    long getLong(String key, long defValue);
    
    /**
     * Retrieve a float value from the preferences.
     * 
     * @param key The name of the preference to retrieve.
     * @param defValue Value to return if this preference does not exist.
     * 
     * @return Returns the preference value if it exists, or defValue.  Throws
     * ClassCastException if there is a preference with this name that is not
     * a float.
     * 
     * @throws ClassCastException
     */
    float getFloat(String key, float defValue);
    
    /**
     * Retrieve a boolean value from the preferences.
     * 
     * @param key The name of the preference to retrieve.
     * @param defValue Value to return if this preference does not exist.
     * 
     * @return Returns the preference value if it exists, or defValue.  Throws
     * ClassCastException if there is a preference with this name that is not
     * a boolean.
     * 
     * @throws ClassCastException
     */
    boolean getBoolean(String key, boolean defValue);

    /**
     * Checks whether the preferences contains a preference.
     * 
     * @param key The name of the preference to check.
     * @return Returns true if the preference exists in the preferences,
     *         otherwise false.
     */
    boolean contains(String key);
    
    /**
     * Create a new Editor for these preferences, through which you can make
     * modifications to the data in the preferences and atomically commit those
     * changes back to the SharedPreferences object.
     * 
     * <p>Note that you <em>must</em> call {@link Editor#commit} to have any
     * changes you perform in the Editor actually show up in the
     * SharedPreferences.
     * 
     * @return Returns a new instance of the {@link Editor} interface, allowing
     * you to modify the values in this SharedPreferences object.
     */
    Editor edit();
    
    /**
     * Registers a callback to be invoked when a change happens to a preference.
     *
     * <p class="caution"><strong>Caution:</strong> The preference manager does
     * not currently store a strong reference to the listener. You must store a
     * strong reference to the listener, or it will be susceptible to garbage
     * collection. We recommend you keep a reference to the listener in the
     * instance data of an object that will exist as long as you need the
     * listener.</p>
     *
     * @param listener The callback that will run.
     * @see #unregisterOnSharedPreferenceChangeListener
     */
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    
    /**
     * Unregisters a previous callback.
     * 
     * @param listener The callback that should be unregistered.
     * @see #registerOnSharedPreferenceChangeListener
     */
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```

## 2.2 SharedPreferencesImpl实现类

`/frameworks/base/core/java/android/app`

```java
/*
 * Copyright (C) 2010 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.app;

import android.annotation.Nullable;
import android.content.SharedPreferences;
import android.os.FileUtils;
import android.os.Looper;
import android.system.ErrnoException;
import android.system.Os;
import android.system.StructStat;
import android.system.StructTimespec;
import android.util.Log;

import com.android.internal.annotations.GuardedBy;
import com.android.internal.util.ExponentiallyBucketedHistogram;
import com.android.internal.util.XmlUtils;

import dalvik.system.BlockGuard;

import libcore.io.IoUtils;

import org.xmlpull.v1.XmlPullParserException;

import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.WeakHashMap;
import java.util.concurrent.CountDownLatch;

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

    /** Current memory state (always increasing) */
    @GuardedBy("this")
    private long mCurrentMemoryStateGeneration;

    /** Latest memory state that was committed to disk */
    @GuardedBy("mWritingToDiskLock")
    private long mDiskStateGeneration;

    /** Time (and number of instances) of file-system sync requests */
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
            if (mLoaded) {
                return;
            }
            if (mBackupFile.exists()) {
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map<String, Object> map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
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
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            } else {
                mMap = new HashMap<>();
            }
            mLock.notifyAll();
        }
    }

    static File makeBackupFile(File prefsFile) {
        return new File(prefsFile.getPath() + ".bak");
    }

    void startReloadIfChangedUnexpectedly() {
        synchronized (mLock) {
            // TODO: wait for any pending writes to disk?
            if (!hasFileChangedUnexpectedly()) {
                return;
            }
            startLoadFromDisk();
        }
    }

    // Has the file changed out from under us?  i.e. writes that
    // we didn't instigate.
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

    @Override
    public Map<String, ?> getAll() {
        synchronized (mLock) {
            awaitLoadedLocked();
            //noinspection unchecked
            return new HashMap<String, Object>(mMap);
        }
    }

    @Override
    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

    @Override
    @Nullable
    public Set<String> getStringSet(String key, @Nullable Set<String> defValues) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Set<String> v = (Set<String>) mMap.get(key);
            return v != null ? v : defValues;
        }
    }

    @Override
    public int getInt(String key, int defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Integer v = (Integer)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    @Override
    public long getLong(String key, long defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Long v = (Long)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    @Override
    public float getFloat(String key, float defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Float v = (Float)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
    @Override
    public boolean getBoolean(String key, boolean defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Boolean v = (Boolean)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

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
        private final Object mEditorLock = new Object();

        @GuardedBy("mEditorLock")
        private final Map<String, Object> mModified = new HashMap<>();

        @GuardedBy("mEditorLock")
        private boolean mClear = false;

        @Override
        public Editor putString(String key, @Nullable String value) {
            synchronized (mEditorLock) {
                mModified.put(key, value);
                return this;
            }
        }
        @Override
        public Editor putStringSet(String key, @Nullable Set<String> values) {
            synchronized (mEditorLock) {
                mModified.put(key,
                        (values == null) ? null : new HashSet<String>(values));
                return this;
            }
        }
        @Override
        public Editor putInt(String key, int value) {
            synchronized (mEditorLock) {
                mModified.put(key, value);
                return this;
            }
        }
        @Override
        public Editor putLong(String key, long value) {
            synchronized (mEditorLock) {
                mModified.put(key, value);
                return this;
            }
        }
        @Override
        public Editor putFloat(String key, float value) {
            synchronized (mEditorLock) {
                mModified.put(key, value);
                return this;
            }
        }
        @Override
        public Editor putBoolean(String key, boolean value) {
            synchronized (mEditorLock) {
                mModified.put(key, value);
                return this;
            }
        }

        @Override
        public Editor remove(String key) {
            synchronized (mEditorLock) {
                mModified.put(key, this);
                return this;
            }
        }

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
                // Run this function on the main thread.
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

## 2.3 ContextImpl

```java
/*
 * Copyright (C) 2006 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.app;

import android.annotation.NonNull;
import android.annotation.Nullable;
import android.content.BroadcastReceiver;
import android.content.ComponentName;
import android.content.ContentProvider;
import android.content.ContentResolver;
import android.content.Context;
import android.content.ContextWrapper;
import android.content.IContentProvider;
import android.content.IIntentReceiver;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.IntentSender;
import android.content.ReceiverCallNotAllowedException;
import android.content.ServiceConnection;
import android.content.SharedPreferences;
import android.content.pm.ActivityInfo;
import android.content.pm.ApplicationInfo;
import android.content.pm.IPackageManager;
import android.content.pm.PackageManager;
import android.content.pm.PackageManager.NameNotFoundException;
import android.content.res.AssetManager;
import android.content.res.CompatResources;
import android.content.res.CompatibilityInfo;
import android.content.res.Configuration;
import android.content.res.Resources;
import android.database.DatabaseErrorHandler;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteDatabase.CursorFactory;
import android.graphics.Bitmap;
import android.graphics.drawable.Drawable;
import android.net.Uri;
import android.os.Binder;
import android.os.Build;
import android.os.Bundle;
import android.os.Debug;
import android.os.Environment;
import android.os.FileUtils;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Process;
import android.os.RemoteException;
import android.os.ServiceManager;
import android.os.SystemProperties;
import android.os.Trace;
import android.os.UserHandle;
import android.os.storage.IStorageManager;
import android.os.storage.StorageManager;
import android.system.ErrnoException;
import android.system.Os;
import android.system.OsConstants;
import android.system.StructStat;
import android.util.AndroidRuntimeException;
import android.util.ArrayMap;
import android.util.Log;
import android.util.Slog;
import android.view.Display;
import android.view.DisplayAdjustments;

import com.android.internal.annotations.GuardedBy;
import com.android.internal.util.Preconditions;

import libcore.io.Memory;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.FilenameFilter;
import java.io.IOException;
import java.io.InputStream;
import java.nio.ByteOrder;
import java.util.ArrayList;
import java.util.Objects;

class ReceiverRestrictedContext extends ContextWrapper {
    ReceiverRestrictedContext(Context base) {
        super(base);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        if (receiver == null) {
            // Allow retrieving current sticky broadcast; this is safe since we
            // aren't actually registering a receiver.
            return super.registerReceiver(null, filter, broadcastPermission, scheduler);
        } else {
            throw new ReceiverCallNotAllowedException(
                    "BroadcastReceiver components are not allowed to register to receive intents");
        }
    }

    @Override
    public Intent registerReceiverAsUser(BroadcastReceiver receiver, UserHandle user,
            IntentFilter filter, String broadcastPermission, Handler scheduler) {
        if (receiver == null) {
            // Allow retrieving current sticky broadcast; this is safe since we
            // aren't actually registering a receiver.
            return super.registerReceiverAsUser(null, user, filter, broadcastPermission, scheduler);
        } else {
            throw new ReceiverCallNotAllowedException(
                    "BroadcastReceiver components are not allowed to register to receive intents");
        }
    }

    @Override
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        throw new ReceiverCallNotAllowedException(
                "BroadcastReceiver components are not allowed to bind to services");
    }
}

/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {
    private final static String TAG = "ContextImpl";
    private final static boolean DEBUG = false;

    private static final String XATTR_INODE_CACHE = "user.inode_cache";
    private static final String XATTR_INODE_CODE_CACHE = "user.inode_code_cache";

    /**
     * Map from package name, to preference name, to cached preferences.
     */
    @GuardedBy("ContextImpl.class")
    private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

    /**
     * Map from preference name to generated path.
     */
    @GuardedBy("ContextImpl.class")
    private ArrayMap<String, File> mSharedPrefsPaths;

    final @NonNull ActivityThread mMainThread;
    final @NonNull LoadedApk mPackageInfo;
    private @Nullable ClassLoader mClassLoader;

    private final @Nullable IBinder mActivityToken;

    private final @Nullable UserHandle mUser;

    private final ApplicationContentResolver mContentResolver;

    private final String mBasePackageName;
    private final String mOpPackageName;

    private final @NonNull ResourcesManager mResourcesManager;
    private @NonNull Resources mResources;
    private @Nullable Display mDisplay; // may be null if default display

    private final int mFlags;

    private Context mOuterContext;
    private int mThemeResource = 0;
    private Resources.Theme mTheme = null;
    private PackageManager mPackageManager;
    private Context mReceiverRestrictedContext = null;

    // The name of the split this Context is representing. May be null.
    private @Nullable String mSplitName = null;

    private final Object mSync = new Object();

    @GuardedBy("mSync")
    private File mDatabasesDir;
    @GuardedBy("mSync")
    private File mPreferencesDir;
    @GuardedBy("mSync")
    private File mFilesDir;
    @GuardedBy("mSync")
    private File mNoBackupFilesDir;
    @GuardedBy("mSync")
    private File mCacheDir;
    @GuardedBy("mSync")
    private File mCodeCacheDir;

    // The system service cache for the system services that are cached per-ContextImpl.
    final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();

    static ContextImpl getImpl(Context context) {
        Context nextContext;
        while ((context instanceof ContextWrapper) &&
                (nextContext=((ContextWrapper)context).getBaseContext()) != null) {
            context = nextContext;
        }
        return (ContextImpl)context;
    }

    @Override
    public AssetManager getAssets() {
        return getResources().getAssets();
    }

    @Override
    public Resources getResources() {
        return mResources;
    }

    @Override
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
            // Doesn't matter if we make more than one instance.
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }

        return null;
    }

    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }

    @Override
    public Looper getMainLooper() {
        return mMainThread.getLooper();
    }

    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }

    @Override
    public void setTheme(int resId) {
        synchronized (mSync) {
            if (mThemeResource != resId) {
                mThemeResource = resId;
                initializeTheme();
            }
        }
    }

    @Override
    public int getThemeResId() {
        synchronized (mSync) {
            return mThemeResource;
        }
    }

    @Override
    public Resources.Theme getTheme() {
        synchronized (mSync) {
            if (mTheme != null) {
                return mTheme;
            }

            mThemeResource = Resources.selectDefaultTheme(mThemeResource,
                    getOuterContext().getApplicationInfo().targetSdkVersion);
            initializeTheme();

            return mTheme;
        }
    }

    private void initializeTheme() {
        if (mTheme == null) {
            mTheme = mResources.newTheme();
        }
        mTheme.applyStyle(mThemeResource, true);
    }

    @Override
    public ClassLoader getClassLoader() {
        return mClassLoader != null ? mClassLoader : (mPackageInfo != null ? mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader());
    }

    @Override
    public String getPackageName() {
        if (mPackageInfo != null) {
            return mPackageInfo.getPackageName();
        }
        // No mPackageInfo means this is a Context for the system itself,
        // and this here is its name.
        return "android";
    }

    /** @hide */
    @Override
    public String getBasePackageName() {
        return mBasePackageName != null ? mBasePackageName : getPackageName();
    }

    /** @hide */
    @Override
    public String getOpPackageName() {
        return mOpPackageName != null ? mOpPackageName : getBasePackageName();
    }

    @Override
    public ApplicationInfo getApplicationInfo() {
        if (mPackageInfo != null) {
            return mPackageInfo.getApplicationInfo();
        }
        throw new RuntimeException("Not supported in system context");
    }

    @Override
    public String getPackageResourcePath() {
        if (mPackageInfo != null) {
            return mPackageInfo.getResDir();
        }
        throw new RuntimeException("Not supported in system context");
    }

    @Override
    public String getPackageCodePath() {
        if (mPackageInfo != null) {
            return mPackageInfo.getAppDir();
        }
        throw new RuntimeException("Not supported in system context");
    }

    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }

        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }

    private boolean isBuggy() {
        // STOPSHIP: fix buggy apps
        if (SystemProperties.getBoolean("fw.ignore_buggy", false)) return false;
        if ("com.google.android.tts".equals(getApplicationInfo().packageName)) return true;
        return false;
    }

    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                checkMode(mode);
                if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                    if (isCredentialProtectedStorage()
                            && !getSystemService(StorageManager.class).isUserKeyUnlocked(
                            UserHandle.myUserId())
                            && !isBuggy()) {
                        throw new IllegalStateException("SharedPreferences in credential encrypted "
                                + "storage are not available until after user is unlocked");
                    }
                }
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }

    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }

    @Override
    public void reloadSharedPreferences() {
        // Build the list of all per-context impls (i.e. caches) we know about
        ArrayList<SharedPreferencesImpl> spImpls = new ArrayList<>();
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            for (int i = 0; i < cache.size(); i++) {
                final SharedPreferencesImpl sp = cache.valueAt(i);
                if (sp != null) {
                    spImpls.add(sp);
                }
            }
        }

        // Issue the reload outside the cache lock
        for (int i = 0; i < spImpls.size(); i++) {
            spImpls.get(i).startReloadIfChangedUnexpectedly();
        }
    }

    /**
     * Try our best to migrate all files from source to target that match
     * requested prefix.
     *
     * @return the number of files moved, or -1 if there was trouble.
     */
    private static int moveFiles(File sourceDir, File targetDir, final String prefix) {
        final File[] sourceFiles = FileUtils.listFilesOrEmpty(sourceDir, new FilenameFilter() {
            @Override
            public boolean accept(File dir, String name) {
                return name.startsWith(prefix);
            }
        });

        int res = 0;
        for (File sourceFile : sourceFiles) {
            final File targetFile = new File(targetDir, sourceFile.getName());
            Log.d(TAG, "Migrating " + sourceFile + " to " + targetFile);
            try {
                FileUtils.copyFileOrThrow(sourceFile, targetFile);
                FileUtils.copyPermissions(sourceFile, targetFile);
                if (!sourceFile.delete()) {
                    throw new IOException("Failed to clean up " + sourceFile);
                }
                if (res != -1) {
                    res++;
                }
            } catch (IOException e) {
                Log.w(TAG, "Failed to migrate " + sourceFile + ": " + e);
                res = -1;
            }
        }
        return res;
    }

    @Override
    public boolean moveSharedPreferencesFrom(Context sourceContext, String name) {
        synchronized (ContextImpl.class) {
            final File source = sourceContext.getSharedPreferencesPath(name);
            final File target = getSharedPreferencesPath(name);

            final int res = moveFiles(source.getParentFile(), target.getParentFile(),
                    source.getName());
            if (res > 0) {
                // We moved at least one file, so evict any in-memory caches for
                // either location
                final ArrayMap<File, SharedPreferencesImpl> cache =
                        getSharedPreferencesCacheLocked();
                cache.remove(source);
                cache.remove(target);
            }
            return res != -1;
        }
    }

    @Override
    public boolean deleteSharedPreferences(String name) {
        synchronized (ContextImpl.class) {
            final File prefs = getSharedPreferencesPath(name);
            final File prefsBackup = SharedPreferencesImpl.makeBackupFile(prefs);

            // Evict any in-memory caches
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            cache.remove(prefs);

            prefs.delete();
            prefsBackup.delete();

            // We failed if files are still lingering
            return !(prefs.exists() || prefsBackup.exists());
        }
    }

    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDir(), "shared_prefs");
            }
            return ensurePrivateDirExists(mPreferencesDir);
        }
    }

    @Override
    public FileInputStream openFileInput(String name)
        throws FileNotFoundException {
        File f = makeFilename(getFilesDir(), name);
        return new FileInputStream(f);
    }

    @Override
    public FileOutputStream openFileOutput(String name, int mode) throws FileNotFoundException {
        checkMode(mode);
        final boolean append = (mode&MODE_APPEND) != 0;
        File f = makeFilename(getFilesDir(), name);
        try {
            FileOutputStream fos = new FileOutputStream(f, append);
            setFilePermissionsFromMode(f.getPath(), mode, 0);
            return fos;
        } catch (FileNotFoundException e) {
        }

        File parent = f.getParentFile();
        parent.mkdir();
        FileUtils.setPermissions(
            parent.getPath(),
            FileUtils.S_IRWXU|FileUtils.S_IRWXG|FileUtils.S_IXOTH,
            -1, -1);
        FileOutputStream fos = new FileOutputStream(f, append);
        setFilePermissionsFromMode(f.getPath(), mode, 0);
        return fos;
    }

    @Override
    public boolean deleteFile(String name) {
        File f = makeFilename(getFilesDir(), name);
        return f.delete();
    }

    /**
     * Common-path handling of app data dir creation
     */
    private static File ensurePrivateDirExists(File file) {
        return ensurePrivateDirExists(file, 0771, -1, null);
    }

    private static File ensurePrivateCacheDirExists(File file, String xattr) {
        final int gid = UserHandle.getCacheAppGid(Process.myUid());
        return ensurePrivateDirExists(file, 02771, gid, xattr);
    }

    private static File ensurePrivateDirExists(File file, int mode, int gid, String xattr) {
        if (!file.exists()) {
            final String path = file.getAbsolutePath();
            try {
                Os.mkdir(path, mode);
                Os.chmod(path, mode);
                if (gid != -1) {
                    Os.chown(path, -1, gid);
                }
            } catch (ErrnoException e) {
                if (e.errno == OsConstants.EEXIST) {
                    // We must have raced with someone; that's okay
                } else {
                    Log.w(TAG, "Failed to ensure " + file + ": " + e.getMessage());
                }
            }

            if (xattr != null) {
                try {
                    final StructStat stat = Os.stat(file.getAbsolutePath());
                    final byte[] value = new byte[8];
                    Memory.pokeLong(value, 0, stat.st_ino, ByteOrder.nativeOrder());
                    Os.setxattr(file.getParentFile().getAbsolutePath(), xattr, value, 0);
                } catch (ErrnoException e) {
                    Log.w(TAG, "Failed to update " + xattr + ": " + e.getMessage());
                }
            }
        }
        return file;
    }

    @Override
    public File getFilesDir() {
        synchronized (mSync) {
            if (mFilesDir == null) {
                mFilesDir = new File(getDataDir(), "files");
            }
            return ensurePrivateDirExists(mFilesDir);
        }
    }

    @Override
    public File getNoBackupFilesDir() {
        synchronized (mSync) {
            if (mNoBackupFilesDir == null) {
                mNoBackupFilesDir = new File(getDataDir(), "no_backup");
            }
            return ensurePrivateDirExists(mNoBackupFilesDir);
        }
    }

    @Override
    public File getExternalFilesDir(String type) {
        // Operates on primary external storage
        return getExternalFilesDirs(type)[0];
    }

    @Override
    public File[] getExternalFilesDirs(String type) {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
            if (type != null) {
                dirs = Environment.buildPaths(dirs, type);
            }
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }

    @Override
    public File getObbDir() {
        // Operates on primary external storage
        return getObbDirs()[0];
    }

    @Override
    public File[] getObbDirs() {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppObbDirs(getPackageName());
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }

    @Override
    public File getCacheDir() {
        synchronized (mSync) {
            if (mCacheDir == null) {
                mCacheDir = new File(getDataDir(), "cache");
            }
            return ensurePrivateCacheDirExists(mCacheDir, XATTR_INODE_CACHE);
        }
    }

    @Override
    public File getCodeCacheDir() {
        synchronized (mSync) {
            if (mCodeCacheDir == null) {
                mCodeCacheDir = new File(getDataDir(), "code_cache");
            }
            return ensurePrivateCacheDirExists(mCodeCacheDir, XATTR_INODE_CODE_CACHE);
        }
    }

    @Override
    public File getExternalCacheDir() {
        // Operates on primary external storage
        return getExternalCacheDirs()[0];
    }

    @Override
    public File[] getExternalCacheDirs() {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppCacheDirs(getPackageName());
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }

    @Override
    public File[] getExternalMediaDirs() {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppMediaDirs(getPackageName());
            return ensureExternalDirsExistOrFilter(dirs);
        }
    }

    /**
     * @hide
     */
    @Nullable
    @Override
    public File getPreloadsFileCache() {
        return Environment.getDataPreloadsFileCacheDirectory(getPackageName());
    }

    @Override
    public File getFileStreamPath(String name) {
        return makeFilename(getFilesDir(), name);
    }

    @Override
    public File getSharedPreferencesPath(String name) {
        return makeFilename(getPreferencesDir(), name + ".xml");
    }

    @Override
    public String[] fileList() {
        return FileUtils.listOrEmpty(getFilesDir());
    }

    @Override
    public SQLiteDatabase openOrCreateDatabase(String name, int mode, CursorFactory factory) {
        return openOrCreateDatabase(name, mode, factory, null);
    }

    @Override
    public SQLiteDatabase openOrCreateDatabase(String name, int mode, CursorFactory factory,
            DatabaseErrorHandler errorHandler) {
        checkMode(mode);
        File f = getDatabasePath(name);
        int flags = SQLiteDatabase.CREATE_IF_NECESSARY;
        if ((mode & MODE_ENABLE_WRITE_AHEAD_LOGGING) != 0) {
            flags |= SQLiteDatabase.ENABLE_WRITE_AHEAD_LOGGING;
        }
        if ((mode & MODE_NO_LOCALIZED_COLLATORS) != 0) {
            flags |= SQLiteDatabase.NO_LOCALIZED_COLLATORS;
        }
        SQLiteDatabase db = SQLiteDatabase.openDatabase(f.getPath(), factory, flags, errorHandler);
        setFilePermissionsFromMode(f.getPath(), mode, 0);
        return db;
    }

    @Override
    public boolean moveDatabaseFrom(Context sourceContext, String name) {
        synchronized (ContextImpl.class) {
            final File source = sourceContext.getDatabasePath(name);
            final File target = getDatabasePath(name);
            return moveFiles(source.getParentFile(), target.getParentFile(),
                    source.getName()) != -1;
        }
    }

    @Override
    public boolean deleteDatabase(String name) {
        try {
            File f = getDatabasePath(name);
            return SQLiteDatabase.deleteDatabase(f);
        } catch (Exception e) {
        }
        return false;
    }

    @Override
    public File getDatabasePath(String name) {
        File dir;
        File f;

        if (name.charAt(0) == File.separatorChar) {
            String dirPath = name.substring(0, name.lastIndexOf(File.separatorChar));
            dir = new File(dirPath);
            name = name.substring(name.lastIndexOf(File.separatorChar));
            f = new File(dir, name);

            if (!dir.isDirectory() && dir.mkdir()) {
                FileUtils.setPermissions(dir.getPath(),
                    FileUtils.S_IRWXU|FileUtils.S_IRWXG|FileUtils.S_IXOTH,
                    -1, -1);
            }
        } else {
            dir = getDatabasesDir();
            f = makeFilename(dir, name);
        }

        return f;
    }

    @Override
    public String[] databaseList() {
        return FileUtils.listOrEmpty(getDatabasesDir());
    }

    private File getDatabasesDir() {
        synchronized (mSync) {
            if (mDatabasesDir == null) {
                if ("android".equals(getPackageName())) {
                    mDatabasesDir = new File("/data/system");
                } else {
                    mDatabasesDir = new File(getDataDir(), "databases");
                }
            }
            return ensurePrivateDirExists(mDatabasesDir);
        }
    }

    @Override
    @Deprecated
    public Drawable getWallpaper() {
        return getWallpaperManager().getDrawable();
    }

    @Override
    @Deprecated
    public Drawable peekWallpaper() {
        return getWallpaperManager().peekDrawable();
    }

    @Override
    @Deprecated
    public int getWallpaperDesiredMinimumWidth() {
        return getWallpaperManager().getDesiredMinimumWidth();
    }

    @Override
    @Deprecated
    public int getWallpaperDesiredMinimumHeight() {
        return getWallpaperManager().getDesiredMinimumHeight();
    }

    @Override
    @Deprecated
    public void setWallpaper(Bitmap bitmap) throws IOException {
        getWallpaperManager().setBitmap(bitmap);
    }

    @Override
    @Deprecated
    public void setWallpaper(InputStream data) throws IOException {
        getWallpaperManager().setStream(data);
    }

    @Override
    @Deprecated
    public void clearWallpaper() throws IOException {
        getWallpaperManager().clear();
    }

    private WallpaperManager getWallpaperManager() {
        return getSystemService(WallpaperManager.class);
    }

    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }

    /** @hide */
    @Override
    public void startActivityAsUser(Intent intent, UserHandle user) {
        startActivityAsUser(intent, null, user);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in.
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }

    /** @hide */
    @Override
    public void startActivityAsUser(Intent intent, Bundle options, UserHandle user) {
        try {
            ActivityManager.getService().startActivityAsUser(
                mMainThread.getApplicationThread(), getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(getContentResolver()),
                null, null, 0, Intent.FLAG_ACTIVITY_NEW_TASK, null, options,
                user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void startActivities(Intent[] intents) {
        warnIfCallingFromSystemProcess();
        startActivities(intents, null);
    }

    /** @hide */
    @Override
    public void startActivitiesAsUser(Intent[] intents, Bundle options, UserHandle userHandle) {
        if ((intents[0].getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivities() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag on first Intent."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivitiesAsUser(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intents, options, userHandle.getIdentifier());
    }

    @Override
    public void startActivities(Intent[] intents, Bundle options) {
        warnIfCallingFromSystemProcess();
        if ((intents[0].getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivities() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag on first Intent."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivities(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intents, options);
    }

    @Override
    public void startIntentSender(IntentSender intent,
            Intent fillInIntent, int flagsMask, int flagsValues, int extraFlags)
            throws IntentSender.SendIntentException {
        startIntentSender(intent, fillInIntent, flagsMask, flagsValues, extraFlags, null);
    }

    @Override
    public void startIntentSender(IntentSender intent, Intent fillInIntent,
            int flagsMask, int flagsValues, int extraFlags, Bundle options)
            throws IntentSender.SendIntentException {
        try {
            String resolvedType = null;
            if (fillInIntent != null) {
                fillInIntent.migrateExtraStreamToClipData();
                fillInIntent.prepareToLeaveProcess(this);
                resolvedType = fillInIntent.resolveTypeIfNeeded(getContentResolver());
            }
            int result = ActivityManager.getService()
                .startActivityIntentSender(mMainThread.getApplicationThread(),
                        intent != null ? intent.getTarget() : null,
                        intent != null ? intent.getWhitelistToken() : null,
                        fillInIntent, resolvedType, null, null,
                        0, flagsMask, flagsValues, options);
            if (result == ActivityManager.START_CANCELED) {
                throw new IntentSender.SendIntentException();
            }
            Instrumentation.checkStartActivityResult(result, null);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcast(Intent intent, String receiverPermission) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    null, false, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcastMultiplePermissions(Intent intent, String[] receiverPermissions) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    null, false, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcast(Intent intent, String receiverPermission, Bundle options) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    options, false, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcast(Intent intent, String receiverPermission, int appOp) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, appOp, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendOrderedBroadcast(Intent intent, String receiverPermission) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    null, true, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendOrderedBroadcast(Intent intent,
            String receiverPermission, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras) {
        sendOrderedBroadcast(intent, receiverPermission, AppOpsManager.OP_NONE,
                resultReceiver, scheduler, initialCode, initialData, initialExtras, null);
    }

    @Override
    public void sendOrderedBroadcast(Intent intent,
            String receiverPermission, Bundle options, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras) {
        sendOrderedBroadcast(intent, receiverPermission, AppOpsManager.OP_NONE,
                resultReceiver, scheduler, initialCode, initialData, initialExtras, options);
    }

    @Override
    public void sendOrderedBroadcast(Intent intent,
            String receiverPermission, int appOp, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras) {
        sendOrderedBroadcast(intent, receiverPermission, appOp,
                resultReceiver, scheduler, initialCode, initialData, initialExtras, null);
    }

    void sendOrderedBroadcast(Intent intent,
            String receiverPermission, int appOp, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras, Bundle options) {
        warnIfCallingFromSystemProcess();
        IIntentReceiver rd = null;
        if (resultReceiver != null) {
            if (mPackageInfo != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    resultReceiver, getOuterContext(), scheduler,
                    mMainThread.getInstrumentation(), false);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        resultReceiver, getOuterContext(), scheduler, null, false).getIIntentReceiver();
            }
        }
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, rd,
                initialCode, initialData, initialExtras, receiverPermissions, appOp,
                    options, true, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcastAsUser(Intent intent, UserHandle user) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(mMainThread.getApplicationThread(),
                    intent, resolvedType, null, Activity.RESULT_OK, null, null, null,
                    AppOpsManager.OP_NONE, null, false, false, user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcastAsUser(Intent intent, UserHandle user,
            String receiverPermission) {
        sendBroadcastAsUser(intent, user, receiverPermission, AppOpsManager.OP_NONE);
    }

    @Override
    public void sendBroadcastAsUser(Intent intent, UserHandle user, String receiverPermission,
            Bundle options) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    options, false, false, user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendBroadcastAsUser(Intent intent, UserHandle user,
            String receiverPermission, int appOp) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, appOp, null, false, false,
                    user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void sendOrderedBroadcastAsUser(Intent intent, UserHandle user,
            String receiverPermission, BroadcastReceiver resultReceiver, Handler scheduler,
            int initialCode, String initialData, Bundle initialExtras) {
        sendOrderedBroadcastAsUser(intent, user, receiverPermission, AppOpsManager.OP_NONE,
                null, resultReceiver, scheduler, initialCode, initialData, initialExtras);
    }

    @Override
    public void sendOrderedBroadcastAsUser(Intent intent, UserHandle user,
            String receiverPermission, int appOp, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData, Bundle initialExtras) {
        sendOrderedBroadcastAsUser(intent, user, receiverPermission, appOp,
                null, resultReceiver, scheduler, initialCode, initialData, initialExtras);
    }

    @Override
    public void sendOrderedBroadcastAsUser(Intent intent, UserHandle user,
            String receiverPermission, int appOp, Bundle options, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData, Bundle initialExtras) {
        IIntentReceiver rd = null;
        if (resultReceiver != null) {
            if (mPackageInfo != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    resultReceiver, getOuterContext(), scheduler,
                    mMainThread.getInstrumentation(), false);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(resultReceiver, getOuterContext(),
                        scheduler, null, false).getIIntentReceiver();
            }
        }
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, rd,
                initialCode, initialData, initialExtras, receiverPermissions,
                    appOp, options, true, false, user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void sendStickyBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, true,
                getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void sendStickyOrderedBroadcast(Intent intent,
            BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras) {
        warnIfCallingFromSystemProcess();
        IIntentReceiver rd = null;
        if (resultReceiver != null) {
            if (mPackageInfo != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    resultReceiver, getOuterContext(), scheduler,
                    mMainThread.getInstrumentation(), false);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        resultReceiver, getOuterContext(), scheduler, null, false).getIIntentReceiver();
            }
        }
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, rd,
                initialCode, initialData, initialExtras, null,
                    AppOpsManager.OP_NONE, null, true, true, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void removeStickyBroadcast(Intent intent) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        if (resolvedType != null) {
            intent = new Intent(intent);
            intent.setDataAndType(intent.getData(), resolvedType);
        }
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().unbroadcastIntent(
                    mMainThread.getApplicationThread(), intent, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void sendStickyBroadcastAsUser(Intent intent, UserHandle user) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, true,
                    user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void sendStickyBroadcastAsUser(Intent intent, UserHandle user, Bundle options) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, options, false, true,
                user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void sendStickyOrderedBroadcastAsUser(Intent intent,
            UserHandle user, BroadcastReceiver resultReceiver,
            Handler scheduler, int initialCode, String initialData,
            Bundle initialExtras) {
        IIntentReceiver rd = null;
        if (resultReceiver != null) {
            if (mPackageInfo != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    resultReceiver, getOuterContext(), scheduler,
                    mMainThread.getInstrumentation(), false);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        resultReceiver, getOuterContext(), scheduler, null, false).getIIntentReceiver();
            }
        }
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, rd,
                initialCode, initialData, initialExtras, null,
                    AppOpsManager.OP_NONE, null, true, true, user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    @Deprecated
    public void removeStickyBroadcastAsUser(Intent intent, UserHandle user) {
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        if (resolvedType != null) {
            intent = new Intent(intent);
            intent.setDataAndType(intent.getData(), resolvedType);
        }
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().unbroadcastIntent(
                    mMainThread.getApplicationThread(), intent, user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            int flags) {
        return registerReceiver(receiver, filter, null, null, flags);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), 0);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler, int flags) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), flags);
    }

    @Override
    public Intent registerReceiverAsUser(BroadcastReceiver receiver, UserHandle user,
            IntentFilter filter, String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, user.getIdentifier(),
                filter, broadcastPermission, scheduler, getOuterContext(), 0);
    }

    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void unregisterReceiver(BroadcastReceiver receiver) {
        if (mPackageInfo != null) {
            IIntentReceiver rd = mPackageInfo.forgetReceiverDispatcher(
                    getOuterContext(), receiver);
            try {
                ActivityManager.getService().unregisterReceiver(rd);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
    }

    private void validateServiceIntent(Intent service) {
        if (service.getComponent() == null && service.getPackage() == null) {
            if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                IllegalArgumentException ex = new IllegalArgumentException(
                        "Service Intent must be explicit: " + service);
                throw ex;
            } else {
                Log.w(TAG, "Implicit intents with startService are not safe: " + service
                        + " " + Debug.getCallers(2, 3));
            }
        }
    }

    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }

    @Override
    public ComponentName startForegroundService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, true, mUser);
    }

    @Override
    public boolean stopService(Intent service) {
        warnIfCallingFromSystemProcess();
        return stopServiceCommon(service, mUser);
    }

    @Override
    public ComponentName startServiceAsUser(Intent service, UserHandle user) {
        return startServiceCommon(service, false, user);
    }

    @Override
    public ComponentName startForegroundServiceAsUser(Intent service, UserHandle user) {
        return startServiceCommon(service, true, user);
    }

    private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), requireForeground,
                            getOpPackageName(), user.getIdentifier());
            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                            + ": " + cn.getClassName());
                } else if (cn.getPackageName().equals("?")) {
                    throw new IllegalStateException(
                            "Not allowed to start service " + service + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public boolean stopServiceAsUser(Intent service, UserHandle user) {
        return stopServiceCommon(service, user);
    }

    private boolean stopServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            int res = ActivityManager.getService().stopService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to stop service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
                Process.myUserHandle());
    }

    /** @hide */
    @Override
    public boolean bindServiceAsUser(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        return bindServiceCommon(service, conn, flags, mMainThread.getHandler(), user);
    }

    /** @hide */
    @Override
    public boolean bindServiceAsUser(Intent service, ServiceConnection conn, int flags,
            Handler handler, UserHandle user) {
        if (handler == null) {
            throw new IllegalArgumentException("handler must not be null.");
        }
        return bindServiceCommon(service, conn, flags, handler, user);
    }

    /** @hide */
    @Override
    public IServiceConnection getServiceDispatcher(ServiceConnection conn, Handler handler,
            int flags) {
        return mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    }

    /** @hide */
    @Override
    public IApplicationThread getIApplicationThread() {
        return mMainThread.getApplicationThread();
    }

    /** @hide */
    @Override
    public Handler getMainThreadHandler() {
        return mMainThread.getHandler();
    }

    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
        // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
        IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess(this);
            int res = ActivityManager.getService().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void unbindService(ServiceConnection conn) {
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            IServiceConnection sd = mPackageInfo.forgetServiceDispatcher(
                    getOuterContext(), conn);
            try {
                ActivityManager.getService().unbindService(sd);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
    }

    @Override
    public boolean startInstrumentation(ComponentName className,
            String profileFile, Bundle arguments) {
        try {
            if (arguments != null) {
                arguments.setAllowFds(false);
            }
            return ActivityManager.getService().startInstrumentation(
                    className, profileFile, 0, arguments, null, null, getUserId(),
                    null /* ABI override */);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }

    @Override
    public String getSystemServiceName(Class<?> serviceClass) {
        return SystemServiceRegistry.getSystemServiceName(serviceClass);
    }

    @Override
    public int checkPermission(String permission, int pid, int uid) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        final IActivityManager am = ActivityManager.getService();
        if (am == null) {
            // Well this is super awkward; we somehow don't have an active
            // ActivityManager instance. If we're testing a root or system
            // UID, then they totally have whatever permission this is.
            final int appId = UserHandle.getAppId(uid);
            if (appId == Process.ROOT_UID || appId == Process.SYSTEM_UID) {
                Slog.w(TAG, "Missing ActivityManager; assuming " + uid + " holds " + permission);
                return PackageManager.PERMISSION_GRANTED;
            }
        }

        try {
            return am.checkPermission(permission, pid, uid);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    /** @hide */
    @Override
    public int checkPermission(String permission, int pid, int uid, IBinder callerToken) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        try {
            return ActivityManager.getService().checkPermissionWithToken(
                    permission, pid, uid, callerToken);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public int checkCallingPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        int pid = Binder.getCallingPid();
        if (pid != Process.myPid()) {
            return checkPermission(permission, pid, Binder.getCallingUid());
        }
        return PackageManager.PERMISSION_DENIED;
    }

    @Override
    public int checkCallingOrSelfPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        return checkPermission(permission, Binder.getCallingPid(),
                Binder.getCallingUid());
    }

    @Override
    public int checkSelfPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        return checkPermission(permission, Process.myPid(), Process.myUid());
    }

    private void enforce(
            String permission, int resultOfCheck,
            boolean selfToo, int uid, String message) {
        if (resultOfCheck != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException(
                    (message != null ? (message + ": ") : "") +
                    (selfToo
                     ? "Neither user " + uid + " nor current process has "
                     : "uid " + uid + " does not have ") +
                    permission +
                    ".");
        }
    }

    @Override
    public void enforcePermission(
            String permission, int pid, int uid, String message) {
        enforce(permission,
                checkPermission(permission, pid, uid),
                false,
                uid,
                message);
    }

    @Override
    public void enforceCallingPermission(String permission, String message) {
        enforce(permission,
                checkCallingPermission(permission),
                false,
                Binder.getCallingUid(),
                message);
    }

    @Override
    public void enforceCallingOrSelfPermission(
            String permission, String message) {
        enforce(permission,
                checkCallingOrSelfPermission(permission),
                true,
                Binder.getCallingUid(),
                message);
    }

    @Override
    public void grantUriPermission(String toPackage, Uri uri, int modeFlags) {
         try {
            ActivityManager.getService().grantUriPermission(
                    mMainThread.getApplicationThread(), toPackage,
                    ContentProvider.getUriWithoutUserId(uri), modeFlags, resolveUserId(uri));
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void revokeUriPermission(Uri uri, int modeFlags) {
         try {
            ActivityManager.getService().revokeUriPermission(
                    mMainThread.getApplicationThread(), null,
                    ContentProvider.getUriWithoutUserId(uri), modeFlags, resolveUserId(uri));
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public void revokeUriPermission(String targetPackage, Uri uri, int modeFlags) {
        try {
            ActivityManager.getService().revokeUriPermission(
                    mMainThread.getApplicationThread(), targetPackage,
                    ContentProvider.getUriWithoutUserId(uri), modeFlags, resolveUserId(uri));
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public int checkUriPermission(Uri uri, int pid, int uid, int modeFlags) {
        try {
            return ActivityManager.getService().checkUriPermission(
                    ContentProvider.getUriWithoutUserId(uri), pid, uid, modeFlags,
                    resolveUserId(uri), null);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    /** @hide */
    @Override
    public int checkUriPermission(Uri uri, int pid, int uid, int modeFlags, IBinder callerToken) {
        try {
            return ActivityManager.getService().checkUriPermission(
                    ContentProvider.getUriWithoutUserId(uri), pid, uid, modeFlags,
                    resolveUserId(uri), callerToken);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    private int resolveUserId(Uri uri) {
        return ContentProvider.getUserIdFromUri(uri, getUserId());
    }

    @Override
    public int checkCallingUriPermission(Uri uri, int modeFlags) {
        int pid = Binder.getCallingPid();
        if (pid != Process.myPid()) {
            return checkUriPermission(uri, pid,
                    Binder.getCallingUid(), modeFlags);
        }
        return PackageManager.PERMISSION_DENIED;
    }

    @Override
    public int checkCallingOrSelfUriPermission(Uri uri, int modeFlags) {
        return checkUriPermission(uri, Binder.getCallingPid(),
                Binder.getCallingUid(), modeFlags);
    }

    @Override
    public int checkUriPermission(Uri uri, String readPermission,
            String writePermission, int pid, int uid, int modeFlags) {
        if (DEBUG) {
            Log.i("foo", "checkUriPermission: uri=" + uri + "readPermission="
                    + readPermission + " writePermission=" + writePermission
                    + " pid=" + pid + " uid=" + uid + " mode" + modeFlags);
        }
        if ((modeFlags&Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
            if (readPermission == null
                    || checkPermission(readPermission, pid, uid)
                    == PackageManager.PERMISSION_GRANTED) {
                return PackageManager.PERMISSION_GRANTED;
            }
        }
        if ((modeFlags&Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
            if (writePermission == null
                    || checkPermission(writePermission, pid, uid)
                    == PackageManager.PERMISSION_GRANTED) {
                return PackageManager.PERMISSION_GRANTED;
            }
        }
        return uri != null ? checkUriPermission(uri, pid, uid, modeFlags)
                : PackageManager.PERMISSION_DENIED;
    }

    private String uriModeFlagToString(int uriModeFlags) {
        StringBuilder builder = new StringBuilder();
        if ((uriModeFlags & Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
            builder.append("read and ");
        }
        if ((uriModeFlags & Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
            builder.append("write and ");
        }
        if ((uriModeFlags & Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION) != 0) {
            builder.append("persistable and ");
        }
        if ((uriModeFlags & Intent.FLAG_GRANT_PREFIX_URI_PERMISSION) != 0) {
            builder.append("prefix and ");
        }

        if (builder.length() > 5) {
            builder.setLength(builder.length() - 5);
            return builder.toString();
        } else {
            throw new IllegalArgumentException("Unknown permission mode flags: " + uriModeFlags);
        }
    }

    private void enforceForUri(
            int modeFlags, int resultOfCheck, boolean selfToo,
            int uid, Uri uri, String message) {
        if (resultOfCheck != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException(
                    (message != null ? (message + ": ") : "") +
                    (selfToo
                     ? "Neither user " + uid + " nor current process has "
                     : "User " + uid + " does not have ") +
                    uriModeFlagToString(modeFlags) +
                    " permission on " +
                    uri +
                    ".");
        }
    }

    @Override
    public void enforceUriPermission(
            Uri uri, int pid, int uid, int modeFlags, String message) {
        enforceForUri(
                modeFlags, checkUriPermission(uri, pid, uid, modeFlags),
                false, uid, uri, message);
    }

    @Override
    public void enforceCallingUriPermission(
            Uri uri, int modeFlags, String message) {
        enforceForUri(
                modeFlags, checkCallingUriPermission(uri, modeFlags),
                false,
                Binder.getCallingUid(), uri, message);
    }

    @Override
    public void enforceCallingOrSelfUriPermission(
            Uri uri, int modeFlags, String message) {
        enforceForUri(
                modeFlags,
                checkCallingOrSelfUriPermission(uri, modeFlags), true,
                Binder.getCallingUid(), uri, message);
    }

    @Override
    public void enforceUriPermission(
            Uri uri, String readPermission, String writePermission,
            int pid, int uid, int modeFlags, String message) {
        enforceForUri(modeFlags,
                      checkUriPermission(
                              uri, readPermission, writePermission, pid, uid,
                              modeFlags),
                      false,
                      uid,
                      uri,
                      message);
    }

    /**
     * Logs a warning if the system process directly called a method such as
     * {@link #startService(Intent)} instead of {@link #startServiceAsUser(Intent, UserHandle)}.
     * The "AsUser" variants allow us to properly enforce the user's restrictions.
     */
    private void warnIfCallingFromSystemProcess() {
        if (Process.myUid() == Process.SYSTEM_UID) {
            Slog.w(TAG, "Calling a method in the system process without a qualified user: "
                    + Debug.getCallers(5));
        }
    }

    private static Resources createResources(IBinder activityToken, LoadedApk pi, String splitName,
            int displayId, Configuration overrideConfig, CompatibilityInfo compatInfo) {
        final String[] splitResDirs;
        final ClassLoader classLoader;
        try {
            splitResDirs = pi.getSplitPaths(splitName);
            classLoader = pi.getSplitClassLoader(splitName);
        } catch (NameNotFoundException e) {
            throw new RuntimeException(e);
        }
        return ResourcesManager.getInstance().getResources(activityToken,
                pi.getResDir(),
                splitResDirs,
                pi.getOverlayDirs(),
                pi.getApplicationInfo().sharedLibraryFiles,
                displayId,
                overrideConfig,
                compatInfo,
                classLoader);
    }

    @Override
    public Context createApplicationContext(ApplicationInfo application, int flags)
            throws NameNotFoundException {
        LoadedApk pi = mMainThread.getPackageInfo(application, mResources.getCompatibilityInfo(),
                flags | CONTEXT_REGISTER_PACKAGE);
        if (pi != null) {
            ContextImpl c = new ContextImpl(this, mMainThread, pi, null, mActivityToken,
                    new UserHandle(UserHandle.getUserId(application.uid)), flags, null);

            final int displayId = mDisplay != null
                    ? mDisplay.getDisplayId() : Display.DEFAULT_DISPLAY;

            c.setResources(createResources(mActivityToken, pi, null, displayId, null,
                    getDisplayAdjustments(displayId).getCompatibilityInfo()));
            if (c.mResources != null) {
                return c;
            }
        }

        throw new PackageManager.NameNotFoundException(
                "Application package " + application.packageName + " not found");
    }

    @Override
    public Context createPackageContext(String packageName, int flags)
            throws NameNotFoundException {
        return createPackageContextAsUser(packageName, flags,
                mUser != null ? mUser : Process.myUserHandle());
    }

    @Override
    public Context createPackageContextAsUser(String packageName, int flags, UserHandle user)
            throws NameNotFoundException {
        if (packageName.equals("system") || packageName.equals("android")) {
            // The system resources are loaded in every application, so we can safely copy
            // the context without reloading Resources.
            return new ContextImpl(this, mMainThread, mPackageInfo, null, mActivityToken, user,
                    flags, null);
        }

        LoadedApk pi = mMainThread.getPackageInfo(packageName, mResources.getCompatibilityInfo(),
                flags | CONTEXT_REGISTER_PACKAGE, user.getIdentifier());
        if (pi != null) {
            ContextImpl c = new ContextImpl(this, mMainThread, pi, null, mActivityToken, user,
                    flags, null);

            final int displayId = mDisplay != null
                    ? mDisplay.getDisplayId() : Display.DEFAULT_DISPLAY;

            c.setResources(createResources(mActivityToken, pi, null, displayId, null,
                    getDisplayAdjustments(displayId).getCompatibilityInfo()));
            if (c.mResources != null) {
                return c;
            }
        }

        // Should be a better exception.
        throw new PackageManager.NameNotFoundException(
                "Application package " + packageName + " not found");
    }

    @Override
    public Context createContextForSplit(String splitName) throws NameNotFoundException {
        if (!mPackageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
            // All Splits are always loaded.
            return this;
        }

        final ClassLoader classLoader = mPackageInfo.getSplitClassLoader(splitName);
        final String[] paths = mPackageInfo.getSplitPaths(splitName);

        final ContextImpl context = new ContextImpl(this, mMainThread, mPackageInfo, splitName,
                mActivityToken, mUser, mFlags, classLoader);

        final int displayId = mDisplay != null
                ? mDisplay.getDisplayId() : Display.DEFAULT_DISPLAY;

        context.setResources(ResourcesManager.getInstance().getResources(
                mActivityToken,
                mPackageInfo.getResDir(),
                paths,
                mPackageInfo.getOverlayDirs(),
                mPackageInfo.getApplicationInfo().sharedLibraryFiles,
                displayId,
                null,
                mPackageInfo.getCompatibilityInfo(),
                classLoader));
        return context;
    }

    @Override
    public Context createConfigurationContext(Configuration overrideConfiguration) {
        if (overrideConfiguration == null) {
            throw new IllegalArgumentException("overrideConfiguration must not be null");
        }

        ContextImpl context = new ContextImpl(this, mMainThread, mPackageInfo, mSplitName,
                mActivityToken, mUser, mFlags, mClassLoader);

        final int displayId = mDisplay != null ? mDisplay.getDisplayId() : Display.DEFAULT_DISPLAY;
        context.setResources(createResources(mActivityToken, mPackageInfo, mSplitName, displayId,
                overrideConfiguration, getDisplayAdjustments(displayId).getCompatibilityInfo()));
        return context;
    }

    @Override
    public Context createDisplayContext(Display display) {
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }

        ContextImpl context = new ContextImpl(this, mMainThread, mPackageInfo, mSplitName,
                mActivityToken, mUser, mFlags, mClassLoader);

        final int displayId = display.getDisplayId();
        context.setResources(createResources(mActivityToken, mPackageInfo, mSplitName, displayId,
                null, getDisplayAdjustments(displayId).getCompatibilityInfo()));
        context.mDisplay = display;
        return context;
    }

    @Override
    public Context createDeviceProtectedStorageContext() {
        final int flags = (mFlags & ~Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE)
                | Context.CONTEXT_DEVICE_PROTECTED_STORAGE;
        return new ContextImpl(this, mMainThread, mPackageInfo, mSplitName, mActivityToken, mUser,
                flags, mClassLoader);
    }

    @Override
    public Context createCredentialProtectedStorageContext() {
        final int flags = (mFlags & ~Context.CONTEXT_DEVICE_PROTECTED_STORAGE)
                | Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE;
        return new ContextImpl(this, mMainThread, mPackageInfo, mSplitName, mActivityToken, mUser,
                flags, mClassLoader);
    }

    @Override
    public boolean isRestricted() {
        return (mFlags & Context.CONTEXT_RESTRICTED) != 0;
    }

    @Override
    public boolean isDeviceProtectedStorage() {
        return (mFlags & Context.CONTEXT_DEVICE_PROTECTED_STORAGE) != 0;
    }

    @Override
    public boolean isCredentialProtectedStorage() {
        return (mFlags & Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE) != 0;
    }

    @Override
    public boolean canLoadUnsafeResources() {
        if (getPackageName().equals(getOpPackageName())) {
            return true;
        }
        return (mFlags & Context.CONTEXT_IGNORE_SECURITY) != 0;
    }

    @Override
    public Display getDisplay() {
        if (mDisplay == null) {
            return mResourcesManager.getAdjustedDisplay(Display.DEFAULT_DISPLAY,
                    mResources);
        }

        return mDisplay;
    }

    @Override
    public void updateDisplay(int displayId) {
        mDisplay = mResourcesManager.getAdjustedDisplay(displayId, mResources);
    }

    @Override
    public DisplayAdjustments getDisplayAdjustments(int displayId) {
        return mResources.getDisplayAdjustments();
    }

    @Override
    public File getDataDir() {
        if (mPackageInfo != null) {
            File res = null;
            if (isCredentialProtectedStorage()) {
                res = mPackageInfo.getCredentialProtectedDataDirFile();
            } else if (isDeviceProtectedStorage()) {
                res = mPackageInfo.getDeviceProtectedDataDirFile();
            } else {
                res = mPackageInfo.getDataDirFile();
            }

            if (res != null) {
                if (!res.exists() && android.os.Process.myUid() == android.os.Process.SYSTEM_UID) {
                    Log.wtf(TAG, "Data directory doesn't exist for package " + getPackageName(),
                            new Throwable());
                }
                return res;
            } else {
                throw new RuntimeException(
                        "No data directory found for package " + getPackageName());
            }
        } else {
            throw new RuntimeException(
                    "No package details found for package " + getPackageName());
        }
    }

    @Override
    public File getDir(String name, int mode) {
        checkMode(mode);
        name = "app_" + name;
        File file = makeFilename(getDataDir(), name);
        if (!file.exists()) {
            file.mkdir();
            setFilePermissionsFromMode(file.getPath(), mode,
                    FileUtils.S_IRWXU|FileUtils.S_IRWXG|FileUtils.S_IXOTH);
        }
        return file;
    }

    /** {@hide} */
    @Override
    public int getUserId() {
        return mUser.getIdentifier();
    }

    static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
        context.setResources(packageInfo.getResources());
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetrics());
        return context;
    }

    /**
     * System Context to be used for UI. This Context has resources that can be themed.
     * Make sure that the created system UI context shares the same LoadedApk as the system context.
     */
    static ContextImpl createSystemUiContext(ContextImpl systemContext) {
        final LoadedApk packageInfo = systemContext.mPackageInfo;
        ContextImpl context = new ContextImpl(null, systemContext.mMainThread, packageInfo, null,
                null, null, 0, null);
        context.setResources(createResources(null, packageInfo, null, Display.DEFAULT_DISPLAY, null,
                packageInfo.getCompatibilityInfo()));
        return context;
    }

    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
        context.setResources(packageInfo.getResources());
        return context;
    }

    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");

        String[] splitDirs = packageInfo.getSplitResDirs();
        ClassLoader classLoader = packageInfo.getClassLoader();

        if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "SplitDependencies");
            try {
                classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
                splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
            } catch (NameNotFoundException e) {
                // Nothing above us can handle a NameNotFoundException, better crash.
                throw new RuntimeException(e);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
            }
        }

        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
                activityToken, null, 0, classLoader);

        // Clamp display ID to DEFAULT_DISPLAY if it is INVALID_DISPLAY.
        displayId = (displayId != Display.INVALID_DISPLAY) ? displayId : Display.DEFAULT_DISPLAY;

        final CompatibilityInfo compatInfo = (displayId == Display.DEFAULT_DISPLAY)
                ? packageInfo.getCompatibilityInfo()
                : CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO;

        final ResourcesManager resourcesManager = ResourcesManager.getInstance();

        // Create the base resources for which all configuration contexts for this Activity
        // will be rebased upon.
        context.setResources(resourcesManager.createBaseActivityResources(activityToken,
                packageInfo.getResDir(),
                splitDirs,
                packageInfo.getOverlayDirs(),
                packageInfo.getApplicationInfo().sharedLibraryFiles,
                displayId,
                overrideConfiguration,
                compatInfo,
                classLoader));
        context.mDisplay = resourcesManager.getAdjustedDisplay(displayId,
                context.getResources());
        return context;
    }

    private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
            @NonNull LoadedApk packageInfo, @Nullable String splitName,
            @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
            @Nullable ClassLoader classLoader) {
        mOuterContext = this;

        // If creator didn't specify which storage to use, use the default
        // location for application.
        if ((flags & (Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE
                | Context.CONTEXT_DEVICE_PROTECTED_STORAGE)) == 0) {
            final File dataDir = packageInfo.getDataDirFile();
            if (Objects.equals(dataDir, packageInfo.getCredentialProtectedDataDirFile())) {
                flags |= Context.CONTEXT_CREDENTIAL_PROTECTED_STORAGE;
            } else if (Objects.equals(dataDir, packageInfo.getDeviceProtectedDataDirFile())) {
                flags |= Context.CONTEXT_DEVICE_PROTECTED_STORAGE;
            }
        }

        mMainThread = mainThread;
        mActivityToken = activityToken;
        mFlags = flags;

        if (user == null) {
            user = Process.myUserHandle();
        }
        mUser = user;

        mPackageInfo = packageInfo;
        mSplitName = splitName;
        mClassLoader = classLoader;
        mResourcesManager = ResourcesManager.getInstance();

        if (container != null) {
            mBasePackageName = container.mBasePackageName;
            mOpPackageName = container.mOpPackageName;
            setResources(container.mResources);
            mDisplay = container.mDisplay;
        } else {
            mBasePackageName = packageInfo.mPackageName;
            ApplicationInfo ainfo = packageInfo.getApplicationInfo();
            if (ainfo.uid == Process.SYSTEM_UID && ainfo.uid != Process.myUid()) {
                // Special case: system components allow themselves to be loaded in to other
                // processes.  For purposes of app ops, we must then consider the context as
                // belonging to the package of this process, not the system itself, otherwise
                // the package+uid verifications in app ops will fail.
                mOpPackageName = ActivityThread.currentPackageName();
            } else {
                mOpPackageName = mBasePackageName;
            }
        }

        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }

    void setResources(Resources r) {
        if (r instanceof CompatResources) {
            ((CompatResources) r).setContext(this);
        }
        mResources = r;
    }

    void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        mPackageInfo.installSystemApplicationInfo(info, classLoader);
    }

    final void scheduleFinalCleanup(String who, String what) {
        mMainThread.scheduleContextCleanup(this, who, what);
    }

    final void performFinalCleanup(String who, String what) {
        //Log.i(TAG, "Cleanup up context: " + this);
        mPackageInfo.removeContextRegistrations(getOuterContext(), who, what);
    }

    final Context getReceiverRestrictedContext() {
        if (mReceiverRestrictedContext != null) {
            return mReceiverRestrictedContext;
        }
        return mReceiverRestrictedContext = new ReceiverRestrictedContext(getOuterContext());
    }

    final void setOuterContext(Context context) {
        mOuterContext = context;
    }

    final Context getOuterContext() {
        return mOuterContext;
    }

    @Override
    public IBinder getActivityToken() {
        return mActivityToken;
    }

    private void checkMode(int mode) {
        if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
            if ((mode & MODE_WORLD_READABLE) != 0) {
                throw new SecurityException("MODE_WORLD_READABLE no longer supported");
            }
            if ((mode & MODE_WORLD_WRITEABLE) != 0) {
                throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
            }
        }
    }

    @SuppressWarnings("deprecation")
    static void setFilePermissionsFromMode(String name, int mode,
            int extraPermissions) {
        int perms = FileUtils.S_IRUSR|FileUtils.S_IWUSR
            |FileUtils.S_IRGRP|FileUtils.S_IWGRP
            |extraPermissions;
        if ((mode&MODE_WORLD_READABLE) != 0) {
            perms |= FileUtils.S_IROTH;
        }
        if ((mode&MODE_WORLD_WRITEABLE) != 0) {
            perms |= FileUtils.S_IWOTH;
        }
        if (DEBUG) {
            Log.i(TAG, "File " + name + ": mode=0x" + Integer.toHexString(mode)
                  + ", perms=0x" + Integer.toHexString(perms));
        }
        FileUtils.setPermissions(name, perms, -1, -1);
    }

    private File makeFilename(File base, String name) {
        if (name.indexOf(File.separatorChar) < 0) {
            return new File(base, name);
        }
        throw new IllegalArgumentException(
                "File " + name + " contains a path separator");
    }

    /**
     * Ensure that given directories exist, trying to create them if missing. If
     * unable to create, they are filtered by replacing with {@code null}.
     */
    private File[] ensureExternalDirsExistOrFilter(File[] dirs) {
        File[] result = new File[dirs.length];
        for (int i = 0; i < dirs.length; i++) {
            File dir = dirs[i];
            if (!dir.exists()) {
                if (!dir.mkdirs()) {
                    // recheck existence in case of cross-process race
                    if (!dir.exists()) {
                        // Failing to mkdir() may be okay, since we might not have
                        // enough permissions; ask vold to create on our behalf.
                        final IStorageManager storageManager = IStorageManager.Stub.asInterface(
                                ServiceManager.getService("mount"));
                        try {
                            final int res = storageManager.mkdirs(
                                    getPackageName(), dir.getAbsolutePath());
                            if (res != 0) {
                                Log.w(TAG, "Failed to ensure " + dir + ": " + res);
                                dir = null;
                            }
                        } catch (Exception e) {
                            Log.w(TAG, "Failed to ensure " + dir + ": " + e);
                            dir = null;
                        }
                    }
                }
            }
            result[i] = dir;
        }
        return result;
    }

    // ----------------------------------------------------------------------
    // ----------------------------------------------------------------------
    // ----------------------------------------------------------------------

    private static final class ApplicationContentResolver extends ContentResolver {
        private final ActivityThread mMainThread;
        private final UserHandle mUser;

        public ApplicationContentResolver(
                Context context, ActivityThread mainThread, UserHandle user) {
            super(context);
            mMainThread = Preconditions.checkNotNull(mainThread);
            mUser = Preconditions.checkNotNull(user);
        }

        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }

        @Override
        protected IContentProvider acquireExistingProvider(Context context, String auth) {
            return mMainThread.acquireExistingProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }

        @Override
        public boolean releaseProvider(IContentProvider provider) {
            return mMainThread.releaseProvider(provider, true);
        }

        @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }

        @Override
        public boolean releaseUnstableProvider(IContentProvider icp) {
            return mMainThread.releaseProvider(icp, false);
        }

        @Override
        public void unstableProviderDied(IContentProvider icp) {
            mMainThread.handleUnstableProviderDied(icp.asBinder(), true);
        }

        @Override
        public void appNotRespondingViaProvider(IContentProvider icp) {
            mMainThread.appNotRespondingViaProvider(icp.asBinder());
        }

        /** @hide */
        protected int resolveUserIdFromAuthority(String auth) {
            return ContentProvider.getUserIdFromAuthority(auth, mUser.getIdentifier());
        }
    }
}
```


