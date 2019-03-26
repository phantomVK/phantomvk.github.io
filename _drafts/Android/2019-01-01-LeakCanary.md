---
layout:     post
title:      "LeakCanary 源码解析"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

```java
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (LeakCanary.isInAnalyzerProcess(this)) return
        LeakCanary.install(this)
    }
}
```

## LeakCanary

```java
  /**
   * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
   * references (on ICS+).
   */
  public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

```java
  public static @NonNull AndroidRefWatcherBuilder refWatcher(@NonNull Context context) {
    return new AndroidRefWatcherBuilder(context);
  }
```

```java
/** A {@link RefWatcherBuilder} with appropriate Android defaults. */
public final class AndroidRefWatcherBuilder extends RefWatcherBuilder<AndroidRefWatcherBuilder> {

  private static final long DEFAULT_WATCH_DELAY_MILLIS = SECONDS.toMillis(5);

  private final Context context;
  private boolean watchActivities = true;
  private boolean watchFragments = true;
  private boolean enableDisplayLeakActivity = false;

  AndroidRefWatcherBuilder(@NonNull Context context) {
    this.context = context.getApplicationContext();
  }

  /**
   * Sets a custom {@link AbstractAnalysisResultService} to listen to analysis results. This
   * overrides any call to {@link #heapDumpListener(HeapDump.Listener)}.
   */
  public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }

  /**
   * Sets a custom delay for how long the {@link RefWatcher} should wait until it checks if a
   * tracked object has been garbage collected. This overrides any call to {@link
   * #watchExecutor(WatchExecutor)}.
   */
  public @NonNull AndroidRefWatcherBuilder watchDelay(long delay, @NonNull TimeUnit unit) {
    return watchExecutor(new AndroidWatchExecutor(unit.toMillis(delay)));
  }

  /**
   * Whether we should automatically watch activities when calling {@link #buildAndInstall()}.
   * Default is true.
   */
  public @NonNull AndroidRefWatcherBuilder watchActivities(boolean watchActivities) {
    this.watchActivities = watchActivities;
    return this;
  }

  /**
   * Whether we should automatically watch fragments when calling {@link #buildAndInstall()}.
   * Default is true. When true, LeakCanary watches native fragments on Android O+ and support
   * fragments if the leakcanary-support-fragment dependency is in the classpath.
   */
  public @NonNull AndroidRefWatcherBuilder watchFragments(boolean watchFragments) {
    this.watchFragments = watchFragments;
    return this;
  }

  /**
   * Sets the maximum number of heap dumps stored. This overrides any call to
   * {@link LeakCanary#setLeakDirectoryProvider(LeakDirectoryProvider)}
   *
   * @throws IllegalArgumentException if maxStoredHeapDumps < 1.
   */
  public @NonNull AndroidRefWatcherBuilder maxStoredHeapDumps(int maxStoredHeapDumps) {
    LeakDirectoryProvider leakDirectoryProvider =
        new DefaultLeakDirectoryProvider(context, maxStoredHeapDumps);
    LeakCanary.setLeakDirectoryProvider(leakDirectoryProvider);
    return self();
  }

  /**
   * Creates a {@link RefWatcher} instance and makes it available through {@link
   * LeakCanary#installedRefWatcher()}.
   *
   * Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
   *
   * @throws UnsupportedOperationException if called more than once per Android process.
   */
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }

  @Override protected boolean isDisabled() {
    return LeakCanary.isInAnalyzerProcess(context);
  }

  @Override protected @NonNull HeapDumper defaultHeapDumper() {
    LeakDirectoryProvider leakDirectoryProvider =
        LeakCanaryInternals.getLeakDirectoryProvider(context);
    return new AndroidHeapDumper(context, leakDirectoryProvider);
  }

  @Override protected @NonNull DebuggerControl defaultDebuggerControl() {
    return new AndroidDebuggerControl();
  }

  @Override protected @NonNull HeapDump.Listener defaultHeapDumpListener() {
    return new ServiceHeapDumpListener(context, DisplayLeakService.class);
  }

  @Override protected @NonNull ExcludedRefs defaultExcludedRefs() {
    return AndroidExcludedRefs.createAppDefaults().build();
  }

  @Override protected @NonNull WatchExecutor defaultWatchExecutor() {
    return new AndroidWatchExecutor(DEFAULT_WATCH_DELAY_MILLIS);
  }

  @Override protected @NonNull
  List<Class<? extends Reachability.Inspector>> defaultReachabilityInspectorClasses() {
    return AndroidReachabilityInspectors.defaultAndroidInspectors();
  }
}
```

```java
/**
 * Watches references that should become weakly reachable. When the {@link RefWatcher} detects that
 * a reference might not be weakly reachable when it should, it triggers the {@link HeapDumper}.
 *
 * <p>This class is thread-safe: you can call {@link #watch(Object)} from any thread.
 */
public final class RefWatcher {

  public static final RefWatcher DISABLED = new RefWatcherBuilder<>().build();

  private final WatchExecutor watchExecutor;
  private final DebuggerControl debuggerControl;
  private final GcTrigger gcTrigger;
  private final HeapDumper heapDumper;
  private final HeapDump.Listener heapdumpListener;
  private final HeapDump.Builder heapDumpBuilder;
  private final Set<String> retainedKeys;
  private final ReferenceQueue<Object> queue;

  RefWatcher(WatchExecutor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger,
      HeapDumper heapDumper, HeapDump.Listener heapdumpListener, HeapDump.Builder heapDumpBuilder) {
    this.watchExecutor = checkNotNull(watchExecutor, "watchExecutor");
    this.debuggerControl = checkNotNull(debuggerControl, "debuggerControl");
    this.gcTrigger = checkNotNull(gcTrigger, "gcTrigger");
    this.heapDumper = checkNotNull(heapDumper, "heapDumper");
    this.heapdumpListener = checkNotNull(heapdumpListener, "heapdumpListener");
    this.heapDumpBuilder = heapDumpBuilder;
    retainedKeys = new CopyOnWriteArraySet<>();
    queue = new ReferenceQueue<>();
  }

  /**
   * Identical to {@link #watch(Object, String)} with an empty string reference name.
   *
   * @see #watch(Object, String)
   */
  public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
   * with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }

  /**
   * LeakCanary will stop watching any references that were passed to {@link #watch(Object, String)}
   * so far.
   */
  public void clearWatchedReferences() {
    retainedKeys.clear();
  }

  boolean isEmpty() {
    removeWeaklyReachableReferences();
    return retainedKeys.isEmpty();
  }

  HeapDump.Builder getHeapDumpBuilder() {
    return heapDumpBuilder;
  }

  Set<String> getRetainedKeys() {
    return new HashSet<>(retainedKeys);
  }

  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }

  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
}
```

```java


/**
 * Called when a watched reference is expected to be weakly reachable, but hasn't been enqueued
 * in the reference queue yet. This gives the application a hook to run the GC before the {@link
 * RefWatcher} checks the reference queue again, to avoid taking a heap dump if possible.
 */
public interface GcTrigger {
  GcTrigger DEFAULT = new GcTrigger() {
    @Override public void runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime().gc();
      enqueueReferences();
      System.runFinalization();
    }

    private void enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100);
      } catch (InterruptedException e) {
        throw new AssertionError();
      }
    }
  };

  void runGc();
}
```

```java
/** Dumps the heap into a file. */
public interface HeapDumper {
  HeapDumper NONE = new HeapDumper() {
    @Override public File dumpHeap() {
      return RETRY_LATER;
    }
  };

  File RETRY_LATER = null;

  /**
   * @return a {@link File} referencing the dumped heap, or {@link #RETRY_LATER} if the heap could
   * not be dumped.
   */
  File dumpHeap();
}
```

```java
public final class AndroidHeapDumper implements HeapDumper {

  private final Context context;
  private final LeakDirectoryProvider leakDirectoryProvider;
  private final Handler mainHandler;

  private Activity resumedActivity;

  public AndroidHeapDumper(@NonNull Context context,
      @NonNull LeakDirectoryProvider leakDirectoryProvider) {
    this.leakDirectoryProvider = leakDirectoryProvider;
    this.context = context.getApplicationContext();
    mainHandler = new Handler(Looper.getMainLooper());

    Application application = (Application) context.getApplicationContext();
    application.registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacksAdapter() {
      @Override public void onActivityResumed(Activity activity) {
        resumedActivity = activity;
      }

      @Override public void onActivityPaused(Activity activity) {
        if (resumedActivity == activity) {
          resumedActivity = null;
        }
      }
    });
  }

  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  @Override @Nullable
  public File dumpHeap() {
    File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

    if (heapDumpFile == RETRY_LATER) {
      return RETRY_LATER;
    }

    FutureResult<Toast> waitingForToast = new FutureResult<>();
    showToast(waitingForToast);

    if (!waitingForToast.wait(5, SECONDS)) {
      CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
      return RETRY_LATER;
    }

    Notification.Builder builder = new Notification.Builder(context)
        .setContentTitle(context.getString(R.string.leak_canary_notification_dumping));
    Notification notification = LeakCanaryInternals.buildNotification(context, builder);
    NotificationManager notificationManager =
        (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    int notificationId = (int) SystemClock.uptimeMillis();
    notificationManager.notify(notificationId, notification);

    Toast toast = waitingForToast.get();
    try {
      Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
      cancelToast(toast);
      notificationManager.cancel(notificationId);
      return heapDumpFile;
    } catch (Exception e) {
      CanaryLog.d(e, "Could not dump heap");
      // Abort heap dump
      return RETRY_LATER;
    }
  }

  private void showToast(final FutureResult<Toast> waitingForToast) {
    mainHandler.post(new Runnable() {
      @Override public void run() {
        if (resumedActivity == null) {
          waitingForToast.set(null);
          return;
        }
        final Toast toast = new Toast(resumedActivity);
        toast.setGravity(Gravity.CENTER_VERTICAL, 0, 0);
        toast.setDuration(Toast.LENGTH_LONG);
        LayoutInflater inflater = LayoutInflater.from(resumedActivity);
        toast.setView(inflater.inflate(R.layout.leak_canary_heap_dump_toast, null));
        toast.show();
        // Waiting for Idle to make sure Toast gets rendered.
        Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
          @Override public boolean queueIdle() {
            waitingForToast.set(toast);
            return false;
          }
        });
      }
    });
  }

  private void cancelToast(final Toast toast) {
    if (toast == null) {
      return;
    }
    mainHandler.post(new Runnable() {
      @Override public void run() {
        toast.cancel();
      }
    });
  }
}
```

```java
/** Data structure holding information about a heap dump. */
public final class HeapDump implements Serializable {

  public static Builder builder() {
    return new Builder();
  }

  /** Receives a heap dump to analyze. */
  public interface Listener {
    Listener NONE = new Listener() {
      @Override public void analyze(HeapDump heapDump) {
      }
    };

    void analyze(HeapDump heapDump);
  }

  /** The heap dump file, which you might want to upload somewhere. */
  public final File heapDumpFile;

  /**
   * Key associated to the {@link KeyedWeakReference} used to detect the memory leak.
   * When analyzing a heap dump, search for all {@link KeyedWeakReference} instances, then open
   * the one that has its "key" field set to this value. Its "referent" field contains the
   * leaking object. Computing the shortest path to GC roots on that leaking object should enable
   * you to figure out the cause of the leak.
   */
  public final String referenceKey;

  /**
   * User defined name to help identify the leaking instance.
   */
  public final String referenceName;

  /** References that should be ignored when analyzing this heap dump. */
  public final ExcludedRefs excludedRefs;

  /** Time from the request to watch the reference until the GC was triggered. */
  public final long watchDurationMs;
  public final long gcDurationMs;
  public final long heapDumpDurationMs;
  public final boolean computeRetainedHeapSize;
  public final List<Class<? extends Reachability.Inspector>> reachabilityInspectorClasses;

  /**
   * Calls {@link #HeapDump(Builder)} with computeRetainedHeapSize set to true.
   *
   * @deprecated Use {@link #HeapDump(Builder)}  instead.
   */
  @Deprecated
  public HeapDump(File heapDumpFile, String referenceKey, String referenceName,
      ExcludedRefs excludedRefs, long watchDurationMs, long gcDurationMs, long heapDumpDurationMs) {
    this(new Builder().heapDumpFile(heapDumpFile)
        .referenceKey(referenceKey)
        .referenceName(referenceName)
        .excludedRefs(excludedRefs)
        .computeRetainedHeapSize(true)
        .watchDurationMs(watchDurationMs)
        .gcDurationMs(gcDurationMs)
        .heapDumpDurationMs(heapDumpDurationMs));
  }

  HeapDump(Builder builder) {
    this.heapDumpFile = builder.heapDumpFile;
    this.referenceKey = builder.referenceKey;
    this.referenceName = builder.referenceName;
    this.excludedRefs = builder.excludedRefs;
    this.computeRetainedHeapSize = builder.computeRetainedHeapSize;
    this.watchDurationMs = builder.watchDurationMs;
    this.gcDurationMs = builder.gcDurationMs;
    this.heapDumpDurationMs = builder.heapDumpDurationMs;
    this.reachabilityInspectorClasses = builder.reachabilityInspectorClasses;
  }

  public Builder buildUpon() {
    return new Builder(this);
  }

  public static final class Builder {
    File heapDumpFile;
    String referenceKey;
    String referenceName;
    ExcludedRefs excludedRefs;
    long watchDurationMs;
    long gcDurationMs;
    long heapDumpDurationMs;
    boolean computeRetainedHeapSize;
    List<Class<? extends Reachability.Inspector>> reachabilityInspectorClasses;

    Builder() {
      this.heapDumpFile = null;
      this.referenceKey = null;
      referenceName = "";
      excludedRefs = null;
      watchDurationMs = 0;
      gcDurationMs = 0;
      heapDumpDurationMs = 0;
      computeRetainedHeapSize = false;
      reachabilityInspectorClasses = null;
    }

    Builder(HeapDump heapDump) {
      this.heapDumpFile = heapDump.heapDumpFile;
      this.referenceKey = heapDump.referenceKey;
      this.referenceName = heapDump.referenceName;
      this.excludedRefs = heapDump.excludedRefs;
      this.computeRetainedHeapSize = heapDump.computeRetainedHeapSize;
      this.watchDurationMs = heapDump.watchDurationMs;
      this.gcDurationMs = heapDump.gcDurationMs;
      this.heapDumpDurationMs = heapDump.heapDumpDurationMs;
      this.reachabilityInspectorClasses = heapDump.reachabilityInspectorClasses;
    }

    public Builder heapDumpFile(File heapDumpFile) {
      this.heapDumpFile = checkNotNull(heapDumpFile, "heapDumpFile");
      return this;
    }

    public Builder referenceKey(String referenceKey) {
      this.referenceKey = checkNotNull(referenceKey, "referenceKey");
      return this;
    }

    public Builder referenceName(String referenceName) {
      this.referenceName = checkNotNull(referenceName, "referenceName");
      return this;
    }

    public Builder excludedRefs(ExcludedRefs excludedRefs) {
      this.excludedRefs = checkNotNull(excludedRefs, "excludedRefs");
      return this;
    }

    public Builder watchDurationMs(long watchDurationMs) {
      this.watchDurationMs = watchDurationMs;
      return this;
    }

    public Builder gcDurationMs(long gcDurationMs) {
      this.gcDurationMs = gcDurationMs;
      return this;
    }

    public Builder heapDumpDurationMs(long heapDumpDurationMs) {
      this.heapDumpDurationMs = heapDumpDurationMs;
      return this;
    }

    public Builder computeRetainedHeapSize(boolean computeRetainedHeapSize) {
      this.computeRetainedHeapSize = computeRetainedHeapSize;
      return this;
    }

    public Builder reachabilityInspectorClasses(
        List<Class<? extends Reachability.Inspector>> reachabilityInspectorClasses) {
      checkNotNull(reachabilityInspectorClasses, "reachabilityInspectorClasses");
      this.reachabilityInspectorClasses =
          unmodifiableList(new ArrayList<>(reachabilityInspectorClasses));
      return this;
    }

    public HeapDump build() {
      checkNotNull(excludedRefs, "excludedRefs");
      checkNotNull(heapDumpFile, "heapDumpFile");
      checkNotNull(referenceKey, "referenceKey");
      checkNotNull(reachabilityInspectorClasses, "reachabilityInspectorClasses");
      return new HeapDump(this);
    }
  }
}

```

```java
public final class ServiceHeapDumpListener implements HeapDump.Listener {

  private final Context context;
  private final Class<? extends AbstractAnalysisResultService> listenerServiceClass;

  public ServiceHeapDumpListener(@NonNull final Context context,
      @NonNull final Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");
    this.context = checkNotNull(context, "context").getApplicationContext();
  }

  @Override public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
}
```

```java
/*
 * Copyright (C) 2015 Square, Inc.
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
package com.squareup.leakcanary;

import android.content.Context;
import android.content.Intent;
import android.support.annotation.NonNull;
import android.support.annotation.Nullable;
import android.support.v4.content.ContextCompat;
import com.squareup.leakcanary.internal.ForegroundService;
import java.io.File;

public abstract class AbstractAnalysisResultService extends ForegroundService {

  private static final String ANALYZED_HEAP_PATH_EXTRA = "analyzed_heap_path_extra";

  public static void sendResultToListener(@NonNull Context context,
      @NonNull String listenerServiceClassName,
      @NonNull HeapDump heapDump,
      @NonNull AnalysisResult result) {
    Class<?> listenerServiceClass;
    try {
      listenerServiceClass = Class.forName(listenerServiceClassName);
    } catch (ClassNotFoundException e) {
      throw new RuntimeException(e);
    }
    Intent intent = new Intent(context, listenerServiceClass);

    File analyzedHeapFile = AnalyzedHeap.save(heapDump, result);
    if (analyzedHeapFile != null) {
      intent.putExtra(ANALYZED_HEAP_PATH_EXTRA, analyzedHeapFile.getAbsolutePath());
    }
    ContextCompat.startForegroundService(context, intent);
  }

  public AbstractAnalysisResultService() {
    super(AbstractAnalysisResultService.class.getName(),
        R.string.leak_canary_notification_reporting);
  }

  @Override protected final void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("AbstractAnalysisResultService received a null intent, ignoring.");
      return;
    }
    if (!intent.hasExtra(ANALYZED_HEAP_PATH_EXTRA)) {
      onAnalysisResultFailure(getString(R.string.leak_canary_result_failure_no_disk_space));
      return;
    }
    File analyzedHeapFile = new File(intent.getStringExtra(ANALYZED_HEAP_PATH_EXTRA));
    AnalyzedHeap analyzedHeap = AnalyzedHeap.load(analyzedHeapFile);
    if (analyzedHeap == null) {
      onAnalysisResultFailure(getString(R.string.leak_canary_result_failure_no_file));
      return;
    }
    try {
      onHeapAnalyzed(analyzedHeap);
    } finally {
      //noinspection ResultOfMethodCallIgnored
      analyzedHeap.heapDump.heapDumpFile.delete();
      //noinspection ResultOfMethodCallIgnored
      analyzedHeap.selfFile.delete();
    }
  }

  /**
   * Called after a heap dump is analyzed, whether or not a leak was found.
   * In {@link AnalyzedHeap#result} check {@link AnalysisResult#leakFound} and {@link
   * AnalysisResult#excludedLeak} to see if there was a leak and if it can be ignored.
   * <p>
   * This will be called from a background intent service thread.
   * <p>
   * It's OK to block here and wait for the heap dump to be uploaded.
   * <p>
   * The analyzed heap file and heap dump file will be deleted immediately after this callback
   * returns.
   */
  protected void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
    onHeapAnalyzed(analyzedHeap.heapDump, analyzedHeap.result);
  }

  /**
   * @deprecated Maintained for backward compatibility. You should override {@link
   * #onHeapAnalyzed(AnalyzedHeap)} instead.
   */
  @SuppressWarnings("DeprecatedIsStillUsed")
  @Deprecated
  protected void onHeapAnalyzed(@NonNull HeapDump heapDump, @NonNull AnalysisResult result) {
  }

  /**
   * Called when there was an error saving or loading the analysis result. This will be called from
   * a background intent service thread.
   */
  protected void onAnalysisResultFailure(String failureMessage) {
    CanaryLog.d(failureMessage);
  }
}
```

```java
/*
 * Copyright (C) 2018 Square, Inc.
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
package com.squareup.leakcanary.internal;

import android.app.IntentService;
import android.app.Notification;
import android.content.Intent;
import android.os.IBinder;
import android.os.SystemClock;
import android.support.annotation.Nullable;
import com.squareup.leakcanary.R;

public abstract class ForegroundService extends IntentService {

  private final int notificationContentTitleResId;
  private final int notificationId;

  public ForegroundService(String name, int notificationContentTitleResId) {
    super(name);
    this.notificationContentTitleResId = notificationContentTitleResId;
    notificationId = (int) SystemClock.uptimeMillis();
  }

  @Override
  public void onCreate() {
    super.onCreate();
    showForegroundNotification(100, 0, true,
        getString(R.string.leak_canary_notification_foreground_text));
  }

  protected void showForegroundNotification(int max, int progress, boolean indeterminate,
      String contentText) {
    Notification.Builder builder = new Notification.Builder(this)
        .setContentTitle(getString(notificationContentTitleResId))
        .setContentText(contentText)
        .setProgress(max, progress, indeterminate);
    Notification notification = LeakCanaryInternals.buildNotification(this, builder);
    startForeground(notificationId, notification);
  }

  @Override protected void onHandleIntent(@Nullable Intent intent) {
    onHandleIntentInForeground(intent);
  }

  protected abstract void onHandleIntentInForeground(@Nullable Intent intent);

  @Override public void onDestroy() {
    super.onDestroy();
    stopForeground(true);
  }

  @Override public IBinder onBind(Intent intent) {
    return null;
  }
}
```

AbstractAnalysisResultService 实现类

```java
/*
 * Copyright (C) 2015 Square, Inc.
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
package com.squareup.leakcanary;

import android.app.PendingIntent;
import android.os.SystemClock;
import android.support.annotation.NonNull;
import com.squareup.leakcanary.internal.DisplayLeakActivity;
import com.squareup.leakcanary.internal.LeakCanaryInternals;
import java.io.File;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

import static android.text.format.Formatter.formatShortFileSize;
import static com.squareup.leakcanary.LeakCanary.leakInfo;
import static com.squareup.leakcanary.internal.LeakCanaryInternals.classSimpleName;

/**
 * Logs leak analysis results, and then shows a notification which will start {@link
 * DisplayLeakActivity}.
 * <p>
 * You can extend this class and override {@link #afterDefaultHandling(HeapDump, AnalysisResult,
 * String)} to add custom behavior, e.g. uploading the heap dump.
 */
public class DisplayLeakService extends AbstractAnalysisResultService {

  @Override
  protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
    HeapDump heapDump = analyzedHeap.heapDump;
    AnalysisResult result = analyzedHeap.result;

    String leakInfo = leakInfo(this, heapDump, result, true);
    CanaryLog.d("%s", leakInfo);

    heapDump = renameHeapdump(heapDump);
    boolean resultSaved = saveResult(heapDump, result);

    String contentTitle;
    if (resultSaved) {
      PendingIntent pendingIntent =
          DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);
      if (result.failure != null) {
        contentTitle = getString(R.string.leak_canary_analysis_failed);
      } else {
        String className = classSimpleName(result.className);
        if (result.leakFound) {
          if (result.retainedHeapSize == AnalysisResult.RETAINED_HEAP_SKIPPED) {
            if (result.excludedLeak) {
              contentTitle = getString(R.string.leak_canary_leak_excluded, className);
            } else {
              contentTitle = getString(R.string.leak_canary_class_has_leaked, className);
            }
          } else {
            String size = formatShortFileSize(this, result.retainedHeapSize);
            if (result.excludedLeak) {
              contentTitle =
                  getString(R.string.leak_canary_leak_excluded_retaining, className, size);
            } else {
              contentTitle =
                  getString(R.string.leak_canary_class_has_leaked_retaining, className, size);
            }
          }
        } else {
          contentTitle = getString(R.string.leak_canary_class_no_leak, className);
        }
      }
      String contentText = getString(R.string.leak_canary_notification_message);
      showNotification(pendingIntent, contentTitle, contentText);
    } else {
      onAnalysisResultFailure(getString(R.string.leak_canary_could_not_save_text));
    }

    afterDefaultHandling(heapDump, result, leakInfo);
  }

  @Override protected final void onAnalysisResultFailure(String failureMessage) {
    super.onAnalysisResultFailure(failureMessage);
    String failureTitle = getString(R.string.leak_canary_result_failure_title);
    showNotification(null, failureTitle, failureMessage);
  }

  private void showNotification(PendingIntent pendingIntent, String contentTitle,
      String contentText) {
    // New notification id every second.
    int notificationId = (int) (SystemClock.uptimeMillis() / 1000);
    LeakCanaryInternals.showNotification(this, contentTitle, contentText, pendingIntent,
        notificationId);
  }

  private boolean saveResult(HeapDump heapDump, AnalysisResult result) {
    File resultFile = AnalyzedHeap.save(heapDump, result);
    return resultFile != null;
  }

  private HeapDump renameHeapdump(HeapDump heapDump) {
    String fileName =
        new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss_SSS'.hprof'", Locale.US).format(new Date());

    File newFile = new File(heapDump.heapDumpFile.getParent(), fileName);
    boolean renamed = heapDump.heapDumpFile.renameTo(newFile);
    if (!renamed) {
      CanaryLog.d("Could not rename heap dump file %s to %s", heapDump.heapDumpFile.getPath(),
          newFile.getPath());
    }
    return heapDump.buildUpon().heapDumpFile(newFile).build();
  }

  /**
   * You can override this method and do a blocking call to a server to upload the leak trace and
   * the heap dump. Don't forget to check {@link AnalysisResult#leakFound} and {@link
   * AnalysisResult#excludedLeak} first.
   */
  protected void afterDefaultHandling(@NonNull HeapDump heapDump, @NonNull AnalysisResult result,
      @NonNull String leakInfo) {
  }
}
```

```java
/*
 * Copyright (C) 2015 Square, Inc.
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
package com.squareup.leakcanary;

import android.support.annotation.NonNull;
import com.squareup.haha.perflib.ArrayInstance;
import com.squareup.haha.perflib.ClassInstance;
import com.squareup.haha.perflib.ClassObj;
import com.squareup.haha.perflib.Field;
import com.squareup.haha.perflib.HprofParser;
import com.squareup.haha.perflib.Instance;
import com.squareup.haha.perflib.RootObj;
import com.squareup.haha.perflib.RootType;
import com.squareup.haha.perflib.Snapshot;
import com.squareup.haha.perflib.Type;
import com.squareup.haha.perflib.io.HprofBuffer;
import com.squareup.haha.perflib.io.MemoryMappedFileBuffer;
import gnu.trove.THashMap;
import gnu.trove.TObjectProcedure;
import java.io.File;
import java.lang.reflect.Constructor;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

import static android.os.Build.VERSION.SDK_INT;
import static android.os.Build.VERSION_CODES.N_MR1;
import static com.squareup.leakcanary.AnalysisResult.failure;
import static com.squareup.leakcanary.AnalysisResult.leakDetected;
import static com.squareup.leakcanary.AnalysisResult.noLeak;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.BUILDING_LEAK_TRACE;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.COMPUTING_BITMAP_SIZE;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.COMPUTING_DOMINATORS;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.DEDUPLICATING_GC_ROOTS;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.FINDING_LEAKING_REF;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.FINDING_SHORTEST_PATH;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.PARSING_HEAP_DUMP;
import static com.squareup.leakcanary.AnalyzerProgressListener.Step.READING_HEAP_DUMP_FILE;
import static com.squareup.leakcanary.HahaHelper.asString;
import static com.squareup.leakcanary.HahaHelper.classInstanceValues;
import static com.squareup.leakcanary.HahaHelper.extendsThread;
import static com.squareup.leakcanary.HahaHelper.fieldValue;
import static com.squareup.leakcanary.HahaHelper.hasField;
import static com.squareup.leakcanary.HahaHelper.threadName;
import static com.squareup.leakcanary.HahaHelper.valueAsString;
import static com.squareup.leakcanary.LeakTraceElement.Holder.ARRAY;
import static com.squareup.leakcanary.LeakTraceElement.Holder.CLASS;
import static com.squareup.leakcanary.LeakTraceElement.Holder.OBJECT;
import static com.squareup.leakcanary.LeakTraceElement.Holder.THREAD;
import static com.squareup.leakcanary.LeakTraceElement.Type.ARRAY_ENTRY;
import static com.squareup.leakcanary.LeakTraceElement.Type.INSTANCE_FIELD;
import static com.squareup.leakcanary.LeakTraceElement.Type.STATIC_FIELD;
import static com.squareup.leakcanary.Reachability.REACHABLE;
import static com.squareup.leakcanary.Reachability.UNKNOWN;
import static com.squareup.leakcanary.Reachability.UNREACHABLE;
import static java.util.concurrent.TimeUnit.NANOSECONDS;

/**
 * Analyzes heap dumps generated by a {@link RefWatcher} to verify if suspected leaks are real.
 */
public final class HeapAnalyzer {

  private static final String ANONYMOUS_CLASS_NAME_PATTERN = "^.+\\$\\d+$";

  private final ExcludedRefs excludedRefs;
  private final AnalyzerProgressListener listener;
  private final List<Reachability.Inspector> reachabilityInspectors;

  /**
   * @deprecated Use {@link #HeapAnalyzer(ExcludedRefs, AnalyzerProgressListener, List)}.
   */
  @Deprecated
  public HeapAnalyzer(@NonNull ExcludedRefs excludedRefs) {
    this(excludedRefs, AnalyzerProgressListener.NONE,
        Collections.<Class<? extends Reachability.Inspector>>emptyList());
  }

  public HeapAnalyzer(@NonNull ExcludedRefs excludedRefs,
      @NonNull AnalyzerProgressListener listener,
      @NonNull List<Class<? extends Reachability.Inspector>> reachabilityInspectorClasses) {
    this.excludedRefs = excludedRefs;
    this.listener = listener;

    this.reachabilityInspectors = new ArrayList<>();
    for (Class<? extends Reachability.Inspector> reachabilityInspectorClass
        : reachabilityInspectorClasses) {
      try {
        Constructor<? extends Reachability.Inspector> defaultConstructor =
            reachabilityInspectorClass.getDeclaredConstructor();
        reachabilityInspectors.add(defaultConstructor.newInstance());
      } catch (Exception e) {
        throw new RuntimeException(e);
      }
    }
  }

  public @NonNull List<TrackedReference> findTrackedReferences(@NonNull File heapDumpFile) {
    if (!heapDumpFile.exists()) {
      throw new IllegalArgumentException("File does not exist: " + heapDumpFile);
    }
    try {
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      Snapshot snapshot = parser.parse();
      deduplicateGcRoots(snapshot);

      ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
      List<TrackedReference> references = new ArrayList<>();
      for (Instance weakRef : refClass.getInstancesList()) {
        List<ClassInstance.FieldValue> values = classInstanceValues(weakRef);
        String key = asString(fieldValue(values, "key"));
        String name =
            hasField(values, "name") ? asString(fieldValue(values, "name")) : "(No name field)";
        Instance instance = fieldValue(values, "referent");
        if (instance != null) {
          String className = getClassName(instance);
          List<LeakReference> fields = describeFields(instance);
          references.add(new TrackedReference(key, name, className, fields));
        }
      }
      return references;
    } catch (Throwable e) {
      throw new RuntimeException(e);
    }
  }

  /**
   * Calls {@link #checkForLeak(File, String, boolean)} with computeRetainedSize set to true.
   *
   * @deprecated Use {@link #checkForLeak(File, String, boolean)} instead.
   */
  @Deprecated
  public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey) {
    return checkForLeak(heapDumpFile, referenceKey, true);
  }

  /**
   * Searches the heap dump for a {@link KeyedWeakReference} instance with the corresponding key,
   * and then computes the shortest strong reference path from that instance to the GC roots.
   */
  public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
      Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
      return failure(exception, since(analysisStartNanoTime));
    }

    try {
      listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      listener.onProgressUpdate(PARSING_HEAP_DUMP);
      Snapshot snapshot = parser.parse();
      listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
      deduplicateGcRoots(snapshot);
      listener.onProgressUpdate(FINDING_LEAKING_REF);
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        String className = leakingRef.getClassObj().getClassName();
        return noLeak(className, since(analysisStartNanoTime));
      }
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }

  /**
   * Pruning duplicates reduces memory pressure from hprof bloat added in Marshmallow.
   */
  void deduplicateGcRoots(Snapshot snapshot) {
    // THashMap has a smaller memory footprint than HashMap.
    final THashMap<String, RootObj> uniqueRootMap = new THashMap<>();

    final Collection<RootObj> gcRoots = snapshot.getGCRoots();
    for (RootObj root : gcRoots) {
      String key = generateRootKey(root);
      if (!uniqueRootMap.containsKey(key)) {
        uniqueRootMap.put(key, root);
      }
    }

    // Repopulate snapshot with unique GC roots.
    gcRoots.clear();
    uniqueRootMap.forEach(new TObjectProcedure<String>() {
      @Override public boolean execute(String key) {
        return gcRoots.add(uniqueRootMap.get(key));
      }
    });
  }

  private String generateRootKey(RootObj root) {
    return String.format("%s@0x%08x", root.getRootType().getName(), root.getId());
  }

  private Instance findLeakingReference(String key, Snapshot snapshot) {
    ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
    if (refClass == null) {
      throw new IllegalStateException(
          "Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
    }
    List<String> keysFound = new ArrayList<>();
    for (Instance instance : refClass.getInstancesList()) {
      List<ClassInstance.FieldValue> values = classInstanceValues(instance);
      Object keyFieldValue = fieldValue(values, "key");
      if (keyFieldValue == null) {
        keysFound.add(null);
        continue;
      }
      String keyCandidate = asString(keyFieldValue);
      if (keyCandidate.equals(key)) {
        return fieldValue(values, "referent");
      }
      keysFound.add(keyCandidate);
    }
    throw new IllegalStateException(
        "Could not find weak reference with key " + key + " in " + keysFound);
  }

  private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
      Instance leakingRef, boolean computeRetainedSize) {

    listener.onProgressUpdate(FINDING_SHORTEST_PATH);
    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);

    String className = leakingRef.getClassObj().getClassName();

    // False alarm, no strong reference path to GC Roots.
    if (result.leakingNode == null) {
      return noLeak(className, since(analysisStartNanoTime));
    }

    listener.onProgressUpdate(BUILDING_LEAK_TRACE);
    LeakTrace leakTrace = buildLeakTrace(result.leakingNode);

    long retainedSize;
    if (computeRetainedSize) {

      listener.onProgressUpdate(COMPUTING_DOMINATORS);
      // Side effect: computes retained size.
      snapshot.computeDominators();

      Instance leakingInstance = result.leakingNode.instance;

      retainedSize = leakingInstance.getTotalRetainedSize();

      // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
      if (SDK_INT <= N_MR1) {
        listener.onProgressUpdate(COMPUTING_BITMAP_SIZE);
        retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
      }
    } else {
      retainedSize = AnalysisResult.RETAINED_HEAP_SKIPPED;
    }

    return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
        since(analysisStartNanoTime));
  }

  /**
   * Bitmaps and bitmap byte arrays are sometimes held by native gc roots, so they aren't included
   * in the retained size because their root dominator is a native gc root.
   * To fix this, we check if the leaking instance is a dominator for each bitmap instance and then
   * add the bitmap size.
   *
   * From experience, we've found that bitmap created in code (Bitmap.createBitmap()) are correctly
   * accounted for, however bitmaps set in layouts are not.
   */
  private long computeIgnoredBitmapRetainedSize(Snapshot snapshot, Instance leakingInstance) {
    long bitmapRetainedSize = 0;
    ClassObj bitmapClass = snapshot.findClass("android.graphics.Bitmap");

    for (Instance bitmapInstance : bitmapClass.getInstancesList()) {
      if (isIgnoredDominator(leakingInstance, bitmapInstance)) {
        ArrayInstance mBufferInstance = fieldValue(classInstanceValues(bitmapInstance), "mBuffer");
        // Native bitmaps have mBuffer set to null. We sadly can't account for them.
        if (mBufferInstance == null) {
          continue;
        }
        long bufferSize = mBufferInstance.getTotalRetainedSize();
        long bitmapSize = bitmapInstance.getTotalRetainedSize();
        // Sometimes the size of the buffer isn't accounted for in the bitmap retained size. Since
        // the buffer is large, it's easy to detect by checking for bitmap size < buffer size.
        if (bitmapSize < bufferSize) {
          bitmapSize += bufferSize;
        }
        bitmapRetainedSize += bitmapSize;
      }
    }
    return bitmapRetainedSize;
  }

  private boolean isIgnoredDominator(Instance dominator, Instance instance) {
    boolean foundNativeRoot = false;
    while (true) {
      Instance immediateDominator = instance.getImmediateDominator();
      if (immediateDominator instanceof RootObj
          && ((RootObj) immediateDominator).getRootType() == RootType.UNKNOWN) {
        // Ignore native roots
        instance = instance.getNextInstanceToGcRoot();
        foundNativeRoot = true;
      } else {
        instance = immediateDominator;
      }
      if (instance == null) {
        return false;
      }
      if (instance == dominator) {
        return foundNativeRoot;
      }
    }
  }

  private LeakTrace buildLeakTrace(LeakNode leakingNode) {
    List<LeakTraceElement> elements = new ArrayList<>();
    // We iterate from the leak to the GC root
    LeakNode node = new LeakNode(null, null, leakingNode, null);
    while (node != null) {
      LeakTraceElement element = buildLeakElement(node);
      if (element != null) {
        elements.add(0, element);
      }
      node = node.parent;
    }

    List<Reachability> expectedReachability =
        computeExpectedReachability(elements);

    return new LeakTrace(elements, expectedReachability);
  }

  private List<Reachability> computeExpectedReachability(
      List<LeakTraceElement> elements) {
    int lastReachableElement = 0;
    int lastElementIndex = elements.size() - 1;
    int firstUnreachableElement = lastElementIndex;
    // No need to inspect the first and last element. We know the first should be reachable (gc
    // root) and the last should be unreachable (watched instance).
    elementLoop:
    for (int i = 1; i < lastElementIndex; i++) {
      LeakTraceElement element = elements.get(i);

      for (Reachability.Inspector reachabilityInspector : reachabilityInspectors) {
        Reachability reachability = reachabilityInspector.expectedReachability(element);
        if (reachability == REACHABLE) {
          lastReachableElement = i;
          break;
        } else if (reachability == UNREACHABLE) {
          firstUnreachableElement = i;
          break elementLoop;
        }
      }
    }

    List<Reachability> expectedReachability = new ArrayList<>();
    for (int i = 0; i < elements.size(); i++) {
      Reachability status;
      if (i <= lastReachableElement) {
        status = REACHABLE;
      } else if (i >= firstUnreachableElement) {
        status = UNREACHABLE;
      } else {
        status = UNKNOWN;
      }
      expectedReachability.add(status);
    }
    return expectedReachability;
  }

  private LeakTraceElement buildLeakElement(LeakNode node) {
    if (node.parent == null) {
      // Ignore any root node.
      return null;
    }
    Instance holder = node.parent.instance;

    if (holder instanceof RootObj) {
      return null;
    }
    LeakTraceElement.Holder holderType;
    String className;
    String extra = null;
    List<LeakReference> leakReferences = describeFields(holder);

    className = getClassName(holder);

    List<String> classHierarchy = new ArrayList<>();
    classHierarchy.add(className);
    String rootClassName = Object.class.getName();
    if (holder instanceof ClassInstance) {
      ClassObj classObj = holder.getClassObj();
      while (!(classObj = classObj.getSuperClassObj()).getClassName().equals(rootClassName)) {
        classHierarchy.add(classObj.getClassName());
      }
    }

    if (holder instanceof ClassObj) {
      holderType = CLASS;
    } else if (holder instanceof ArrayInstance) {
      holderType = ARRAY;
    } else {
      ClassObj classObj = holder.getClassObj();
      if (extendsThread(classObj)) {
        holderType = THREAD;
        String threadName = threadName(holder);
        extra = "(named '" + threadName + "')";
      } else if (className.matches(ANONYMOUS_CLASS_NAME_PATTERN)) {
        String parentClassName = classObj.getSuperClassObj().getClassName();
        if (rootClassName.equals(parentClassName)) {
          holderType = OBJECT;
          try {
            // This is an anonymous class implementing an interface. The API does not give access
            // to the interfaces implemented by the class. We check if it's in the class path and
            // use that instead.
            Class<?> actualClass = Class.forName(classObj.getClassName());
            Class<?>[] interfaces = actualClass.getInterfaces();
            if (interfaces.length > 0) {
              Class<?> implementedInterface = interfaces[0];
              extra = "(anonymous implementation of " + implementedInterface.getName() + ")";
            } else {
              extra = "(anonymous subclass of java.lang.Object)";
            }
          } catch (ClassNotFoundException ignored) {
          }
        } else {
          holderType = OBJECT;
          // Makes it easier to figure out which anonymous class we're looking at.
          extra = "(anonymous subclass of " + parentClassName + ")";
        }
      } else {
        holderType = OBJECT;
      }
    }
    return new LeakTraceElement(node.leakReference, holderType, classHierarchy, extra,
        node.exclusion, leakReferences);
  }

  private List<LeakReference> describeFields(Instance instance) {
    List<LeakReference> leakReferences = new ArrayList<>();
    if (instance instanceof ClassObj) {
      ClassObj classObj = (ClassObj) instance;
      for (Map.Entry<Field, Object> entry : classObj.getStaticFieldValues().entrySet()) {
        String name = entry.getKey().getName();
        String stringValue = valueAsString(entry.getValue());
        leakReferences.add(new LeakReference(STATIC_FIELD, name, stringValue));
      }
    } else if (instance instanceof ArrayInstance) {
      ArrayInstance arrayInstance = (ArrayInstance) instance;
      if (arrayInstance.getArrayType() == Type.OBJECT) {
        Object[] values = arrayInstance.getValues();
        for (int i = 0; i < values.length; i++) {
          String name = Integer.toString(i);
          String stringValue = valueAsString(values[i]);
          leakReferences.add(new LeakReference(ARRAY_ENTRY, name, stringValue));
        }
      }
    } else {
      ClassObj classObj = instance.getClassObj();
      for (Map.Entry<Field, Object> entry : classObj.getStaticFieldValues().entrySet()) {
        String name = entry.getKey().getName();
        String stringValue = valueAsString(entry.getValue());
        leakReferences.add(new LeakReference(STATIC_FIELD, name, stringValue));
      }
      ClassInstance classInstance = (ClassInstance) instance;
      for (ClassInstance.FieldValue field : classInstance.getValues()) {
        String name = field.getField().getName();
        String stringValue = valueAsString(field.getValue());
        leakReferences.add(new LeakReference(INSTANCE_FIELD, name, stringValue));
      }
    }
    return leakReferences;
  }

  private String getClassName(Instance instance) {
    String className;
    if (instance instanceof ClassObj) {
      ClassObj classObj = (ClassObj) instance;
      className = classObj.getClassName();
    } else if (instance instanceof ArrayInstance) {
      ArrayInstance arrayInstance = (ArrayInstance) instance;
      className = arrayInstance.getClassObj().getClassName();
    } else {
      ClassObj classObj = instance.getClassObj();
      className = classObj.getClassName();
    }
    return className;
  }

  private long since(long analysisStartNanoTime) {
    return NANOSECONDS.toMillis(System.nanoTime() - analysisStartNanoTime);
  }
}
```

