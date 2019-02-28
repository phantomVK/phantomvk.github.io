---
layout:     post
title:      "Glide缓存策略"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Glide
---

Glide 4.9.0

## 用法

```java
Glide.with(this)
        .load("http://www.abc.com")
        .into(imageView)
```

## GlideBuilder

```java
@NonNull
Glide build(@NonNull Context context) {
  if (sourceExecutor == null) {
    sourceExecutor = GlideExecutor.newSourceExecutor();
  }

  if (diskCacheExecutor == null) {
    diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
  }

  if (animationExecutor == null) {
    animationExecutor = GlideExecutor.newAnimationExecutor();
  }

  if (memorySizeCalculator == null) {
    memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
  }

  if (connectivityMonitorFactory == null) {
    connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
  }

  if (bitmapPool == null) {
    int size = memorySizeCalculator.getBitmapPoolSize();
    if (size > 0) {
      bitmapPool = new LruBitmapPool(size);
    } else {
      bitmapPool = new BitmapPoolAdapter();
    }
  }

  if (arrayPool == null) {
    arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
  }

  if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
  }

  if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
  }

  if (engine == null) {
    engine =
        new Engine(
            memoryCache,
            diskCacheFactory,
            diskCacheExecutor,
            sourceExecutor,
            GlideExecutor.newUnlimitedSourceExecutor(),
            GlideExecutor.newAnimationExecutor(),
            isActiveResourceRetentionAllowed);
  }

  if (defaultRequestListeners == null) {
    defaultRequestListeners = Collections.emptyList();
  } else {
    defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
  }

  RequestManagerRetriever requestManagerRetriever =
      new RequestManagerRetriever(requestManagerFactory);

  return new Glide(
      context,
      engine,
      memoryCache,
      bitmapPool,
      arrayPool,
      requestManagerRetriever,
      connectivityMonitorFactory,
      logLevel,
      defaultRequestOptions.lock(),
      defaultTransitionOptions,
      defaultRequestListeners,
      isLoggingRequestOriginsEnabled);
}
```

```java
/**
 * A calculator that tries to intelligently determine cache sizes for a given device based on some
 * constants and the devices screen density, width, and height.
 */
public final class MemorySizeCalculator {
  private static final String TAG = "MemorySizeCalculator";
  @VisibleForTesting
  static final int BYTES_PER_ARGB_8888_PIXEL = 4;
  private static final int LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR = 2;

  private final int bitmapPoolSize;
  private final int memoryCacheSize;
  private final Context context;
  private final int arrayPoolSize;

  interface ScreenDimensions {
    int getWidthPixels();
    int getHeightPixels();
  }

  // Package private to avoid PMD warning.
  MemorySizeCalculator(MemorySizeCalculator.Builder builder) {
    this.context = builder.context;

    arrayPoolSize =
        isLowMemoryDevice(builder.activityManager)
            ? builder.arrayPoolSizeBytes / LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR
            : builder.arrayPoolSizeBytes;
    int maxSize =
        getMaxSize(
            builder.activityManager, builder.maxSizeMultiplier, builder.lowMemoryMaxSizeMultiplier);

    int widthPixels = builder.screenDimensions.getWidthPixels();
    int heightPixels = builder.screenDimensions.getHeightPixels();
    int screenSize = widthPixels * heightPixels * BYTES_PER_ARGB_8888_PIXEL;

    int targetBitmapPoolSize = Math.round(screenSize * builder.bitmapPoolScreens);

    int targetMemoryCacheSize = Math.round(screenSize * builder.memoryCacheScreens);
    int availableSize = maxSize - arrayPoolSize;

    if (targetMemoryCacheSize + targetBitmapPoolSize <= availableSize) {
      memoryCacheSize = targetMemoryCacheSize;
      bitmapPoolSize = targetBitmapPoolSize;
    } else {
      float part = availableSize / (builder.bitmapPoolScreens + builder.memoryCacheScreens);
      memoryCacheSize = Math.round(part * builder.memoryCacheScreens);
      bitmapPoolSize = Math.round(part * builder.bitmapPoolScreens);
    }
  }

  /**
   * Returns the recommended memory cache size for the device it is run on in bytes.
   */
  public int getMemoryCacheSize() {
    return memoryCacheSize;
  }

  /**
   * Returns the recommended bitmap pool size for the device it is run on in bytes.
   */
  public int getBitmapPoolSize() {
    return bitmapPoolSize;
  }

  /**
   * Returns the recommended array pool size for the device it is run on in bytes.
   */
  public int getArrayPoolSizeInBytes() {
    return arrayPoolSize;
  }

  private static int getMaxSize(ActivityManager activityManager, float maxSizeMultiplier,
      float lowMemoryMaxSizeMultiplier) {
    final int memoryClassBytes = activityManager.getMemoryClass() * 1024 * 1024;
    final boolean isLowMemoryDevice = isLowMemoryDevice(activityManager);
    return Math.round(memoryClassBytes * (isLowMemoryDevice ? lowMemoryMaxSizeMultiplier
        : maxSizeMultiplier));
  }

  private String toMb(int bytes) {
    return Formatter.formatFileSize(context, bytes);
  }

  @TargetApi(Build.VERSION_CODES.KITKAT)
  @Synthetic static boolean isLowMemoryDevice(ActivityManager activityManager) {
    // Explicitly check with an if statement, on some devices both parts of boolean expressions
    // can be evaluated even if we'd normally expect a short circuit.
    //noinspection SimplifiableIfStatement
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      return activityManager.isLowRamDevice();
    } else {
      return true;
    }
  }

  /**
   * Constructs an {@link MemorySizeCalculator} with reasonable defaults that can be optionally
   * overridden.
   */
  // Public API.
  @SuppressWarnings({"WeakerAccess", "unused"})
  public static final class Builder {
    @VisibleForTesting
    static final int MEMORY_CACHE_TARGET_SCREENS = 2;

    /**
     * On Android O+, we use {@link android.graphics.Bitmap.Config#HARDWARE} for all reasonably
     * sized images unless we're creating thumbnails for the first time. As a result, the Bitmap
     * pool is much less important on O than it was on previous versions.
     */
    static final int BITMAP_POOL_TARGET_SCREENS =
        Build.VERSION.SDK_INT < Build.VERSION_CODES.O ? 4 : 1;

    static final float MAX_SIZE_MULTIPLIER = 0.4f;
    static final float LOW_MEMORY_MAX_SIZE_MULTIPLIER = 0.33f;
    // 4MB.
    static final int ARRAY_POOL_SIZE_BYTES = 4 * 1024 * 1024;

    @Synthetic final Context context;

    // Modifiable (non-final) for testing.
    @Synthetic ActivityManager activityManager;
    @Synthetic ScreenDimensions screenDimensions;

    @Synthetic float memoryCacheScreens = MEMORY_CACHE_TARGET_SCREENS;
    @Synthetic float bitmapPoolScreens = BITMAP_POOL_TARGET_SCREENS;
    @Synthetic float maxSizeMultiplier = MAX_SIZE_MULTIPLIER;
    @Synthetic float lowMemoryMaxSizeMultiplier = LOW_MEMORY_MAX_SIZE_MULTIPLIER;
    @Synthetic int arrayPoolSizeBytes = ARRAY_POOL_SIZE_BYTES;

    public Builder(Context context) {
      this.context = context;
      activityManager =
          (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
      screenDimensions =
          new DisplayMetricsScreenDimensions(context.getResources().getDisplayMetrics());

      // On Android O+ Bitmaps are allocated natively, ART is much more efficient at managing
      // garbage and we rely heavily on HARDWARE Bitmaps, making Bitmap re-use much less important.
      // We prefer to preserve RAM on these devices and take the small performance hit of not
      // re-using Bitmaps and textures when loading very small images or generating thumbnails.
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O && isLowMemoryDevice(activityManager)) {
        bitmapPoolScreens = 0;
      }
    }

    /**
     * Sets the number of device screens worth of pixels the
     * {@link com.bumptech.glide.load.engine.cache.MemoryCache} should be able to hold and
     * returns this Builder.
     */
    public Builder setMemoryCacheScreens(float memoryCacheScreens) {
      Preconditions.checkArgument(memoryCacheScreens >= 0,
          "Memory cache screens must be greater than or equal to 0");
      this.memoryCacheScreens = memoryCacheScreens;
      return this;
    }

    /**
     * Sets the number of device screens worth of pixels the
     * {@link com.bumptech.glide.load.engine.bitmap_recycle.BitmapPool} should be able to hold and
     * returns this Builder.
     */
    public Builder setBitmapPoolScreens(float bitmapPoolScreens) {
      Preconditions.checkArgument(bitmapPoolScreens >= 0,
          "Bitmap pool screens must be greater than or equal to 0");
      this.bitmapPoolScreens = bitmapPoolScreens;
      return this;
    }

    /**
     * Sets the maximum percentage of the device's memory class for standard devices that can be
     * taken up by Glide's {@link com.bumptech.glide.load.engine.cache.MemoryCache} and
     * {@link com.bumptech.glide.load.engine.bitmap_recycle.BitmapPool} put together, and returns
     * this builder.
     */
    public Builder setMaxSizeMultiplier(float maxSizeMultiplier) {
      Preconditions.checkArgument(maxSizeMultiplier >= 0 && maxSizeMultiplier <= 1,
          "Size multiplier must be between 0 and 1");
      this.maxSizeMultiplier = maxSizeMultiplier;
      return this;
    }

    /**
     * Sets the maximum percentage of the device's memory class for low ram devices that can be
     * taken up by Glide's {@link com.bumptech.glide.load.engine.cache.MemoryCache} and
     * {@link com.bumptech.glide.load.engine.bitmap_recycle.BitmapPool} put together, and returns
     * this builder.
     *
     * @see ActivityManager#isLowRamDevice()
     */
    public Builder setLowMemoryMaxSizeMultiplier(float lowMemoryMaxSizeMultiplier) {
      Preconditions.checkArgument(
          lowMemoryMaxSizeMultiplier >= 0 && lowMemoryMaxSizeMultiplier <= 1,
              "Low memory max size multiplier must be between 0 and 1");
      this.lowMemoryMaxSizeMultiplier = lowMemoryMaxSizeMultiplier;
      return this;
    }

    /**
     * Sets the size in bytes of the {@link
     * com.bumptech.glide.load.engine.bitmap_recycle.ArrayPool} to use to store temporary
     * arrays while decoding data and returns this builder.
     *
     * <p>This number will be halved on low memory devices that return {@code true} from
     * {@link ActivityManager#isLowRamDevice()}.
     */
    public Builder setArrayPoolSize(int arrayPoolSizeBytes) {
      this.arrayPoolSizeBytes = arrayPoolSizeBytes;
      return this;
    }

    @VisibleForTesting
    Builder setActivityManager(ActivityManager activityManager) {
      this.activityManager = activityManager;
      return this;
    }

    @VisibleForTesting
    Builder setScreenDimensions(ScreenDimensions screenDimensions) {
      this.screenDimensions = screenDimensions;
      return this;
    }

    public MemorySizeCalculator build() {
      return new MemorySizeCalculator(this);
    }
  }

  private static final class DisplayMetricsScreenDimensions implements ScreenDimensions {
    private final DisplayMetrics displayMetrics;

    DisplayMetricsScreenDimensions(DisplayMetrics displayMetrics) {
      this.displayMetrics = displayMetrics;
    }

    @Override
    public int getWidthPixels() {
      return displayMetrics.widthPixels;
    }

    @Override
    public int getHeightPixels() {
      return displayMetrics.heightPixels;
    }
  }
}
```

```java
package com.bumptech.glide.load.engine.cache;

import android.annotation.SuppressLint;
import android.support.annotation.NonNull;
import android.support.annotation.Nullable;
import com.bumptech.glide.load.Key;
import com.bumptech.glide.load.engine.Resource;
import com.bumptech.glide.util.LruCache;

/**
 * An LRU in memory cache for {@link com.bumptech.glide.load.engine.Resource}s.
 */
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
  private ResourceRemovedListener listener;

  /**
   * Constructor for LruResourceCache.
   *
   * @param size The maximum size in bytes the in memory cache can use.
   */
  public LruResourceCache(long size) {
    super(size);
  }

  @Override
  public void setResourceRemovedListener(@NonNull ResourceRemovedListener listener) {
    this.listener = listener;
  }

  @Override
  protected void onItemEvicted(@NonNull Key key, @Nullable Resource<?> item) {
    if (listener != null && item != null) {
      listener.onResourceRemoved(item);
    }
  }

  @Override
  protected int getSize(@Nullable Resource<?> item) {
    if (item == null) {
      return super.getSize(null);
    } else {
      return item.getSize();
    }
  }

  @SuppressLint("InlinedApi")
  @Override
  public void trimMemory(int level) {
    if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
      // Entering list of cached background apps
      // Evict our entire bitmap cache
      clearMemory();
    } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
        || level == android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {
      // The app's UI is no longer visible, or app is in the foreground but system is running
      // critically low on memory
      // Evict oldest half of our bitmap cache
      trimToSize(getMaxSize() / 2);
    }
  }
}
```

```java
/**
 * Responsible for starting loads and managing active and cached resources.
 */
public class Engine implements EngineJobListener,
    MemoryCache.ResourceRemovedListener,
    EngineResource.ResourceListener {
  private static final String TAG = "Engine";
  private static final int JOB_POOL_SIZE = 150;
  private static final boolean VERBOSE_IS_LOGGABLE = Log.isLoggable(TAG, Log.VERBOSE);
  private final Jobs jobs;
  private final EngineKeyFactory keyFactory;
  private final MemoryCache cache;
  private final EngineJobFactory engineJobFactory;
  private final ResourceRecycler resourceRecycler;
  private final LazyDiskCacheProvider diskCacheProvider;
  private final DecodeJobFactory decodeJobFactory;
  private final ActiveResources activeResources;

  public Engine(
      MemoryCache memoryCache,
      DiskCache.Factory diskCacheFactory,
      GlideExecutor diskCacheExecutor,
      GlideExecutor sourceExecutor,
      GlideExecutor sourceUnlimitedExecutor,
      GlideExecutor animationExecutor,
      boolean isActiveResourceRetentionAllowed) {
    this(
        memoryCache,
        diskCacheFactory,
        diskCacheExecutor,
        sourceExecutor,
        sourceUnlimitedExecutor,
        animationExecutor,
        /*jobs=*/ null,
        /*keyFactory=*/ null,
        /*activeResources=*/ null,
        /*engineJobFactory=*/ null,
        /*decodeJobFactory=*/ null,
        /*resourceRecycler=*/ null,
        isActiveResourceRetentionAllowed);
  }

  @VisibleForTesting
  Engine(MemoryCache cache,
      DiskCache.Factory diskCacheFactory,
      GlideExecutor diskCacheExecutor,
      GlideExecutor sourceExecutor,
      GlideExecutor sourceUnlimitedExecutor,
      GlideExecutor animationExecutor,
      Jobs jobs,
      EngineKeyFactory keyFactory,
      ActiveResources activeResources,
      EngineJobFactory engineJobFactory,
      DecodeJobFactory decodeJobFactory,
      ResourceRecycler resourceRecycler,
      boolean isActiveResourceRetentionAllowed) {
    this.cache = cache;
    this.diskCacheProvider = new LazyDiskCacheProvider(diskCacheFactory);

    if (activeResources == null) {
      activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
    }
    this.activeResources = activeResources;
    activeResources.setListener(this);

    if (keyFactory == null) {
      keyFactory = new EngineKeyFactory();
    }
    this.keyFactory = keyFactory;

    if (jobs == null) {
      jobs = new Jobs();
    }
    this.jobs = jobs;

    if (engineJobFactory == null) {
      engineJobFactory =
          new EngineJobFactory(
              diskCacheExecutor, sourceExecutor, sourceUnlimitedExecutor, animationExecutor, this);
    }
    this.engineJobFactory = engineJobFactory;

    if (decodeJobFactory == null) {
      decodeJobFactory = new DecodeJobFactory(diskCacheProvider);
    }
    this.decodeJobFactory = decodeJobFactory;

    if (resourceRecycler == null) {
      resourceRecycler = new ResourceRecycler();
    }
    this.resourceRecycler = resourceRecycler;

    cache.setResourceRemovedListener(this);
  }

  /**
   * Starts a load for the given arguments.
   *
   * <p>Must be called on the main thread.
   *
   * <p>The flow for any request is as follows:
   *
   * <ul>
   *   <li>Check the current set of actively used resources, return the active resource if present,
   *       and move any newly inactive resources into the memory cache.
   *   <li>Check the memory cache and provide the cached resource if present.
   *   <li>Check the current set of in progress loads and add the cb to the in progress load if one
   *       is present.
   *   <li>Start a new load.
   * </ul>
   *
   * <p>Active resources are those that have been provided to at least one request and have not yet
   * been released. Once all consumers of a resource have released that resource, the resource then
   * goes to cache. If the resource is ever returned to a new consumer from cache, it is re-added to
   * the active resources. If the resource is evicted from the cache, its resources are recycled and
   * re-used if possible and the resource is discarded. There is no strict requirement that
   * consumers release their resources so active resources are held weakly.
   *
   * @param width The target width in pixels of the desired resource.
   * @param height The target height in pixels of the desired resource.
   * @param cb The callback that will be called when the load completes.
   */
  public synchronized <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);

    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);

    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }

  private static void logWithTimeAndKey(String log, long startTime, Key key) {
    Log.v(TAG, log + " in " + LogTime.getElapsedMillis(startTime) + "ms, key: " + key);
  }

  @Nullable
  private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }

    return active;
  }

  private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
  }

  private EngineResource<?> getEngineResourceFromCache(Key key) {
    Resource<?> cached = cache.remove(key);

    final EngineResource<?> result;
    if (cached == null) {
      result = null;
    } else if (cached instanceof EngineResource) {
      // Save an object allocation if we've cached an EngineResource (the typical case).
      result = (EngineResource<?>) cached;
    } else {
      result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
    }
    return result;
  }

  public void release(Resource<?> resource) {
    if (resource instanceof EngineResource) {
      ((EngineResource<?>) resource).release();
    } else {
      throw new IllegalArgumentException("Cannot release anything but an EngineResource");
    }
  }

  @SuppressWarnings("unchecked")
  @Override
  public synchronized void onEngineJobComplete(
      EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null) {
      resource.setResourceListener(key, this);

      if (resource.isCacheable()) {
        activeResources.activate(key, resource);
      }
    }

    jobs.removeIfCurrent(key, engineJob);
  }

  @Override
  public synchronized void onEngineJobCancelled(EngineJob<?> engineJob, Key key) {
    jobs.removeIfCurrent(key, engineJob);
  }

  @Override
  public void onResourceRemoved(@NonNull final Resource<?> resource) {
    resourceRecycler.recycle(resource);
  }

  @Override
  public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
      cache.put(cacheKey, resource);
    } else {
      resourceRecycler.recycle(resource);
    }
  }

  public void clearDiskCache() {
    diskCacheProvider.getDiskCache().clear();
  }

  @VisibleForTesting
  public void shutdown() {
    engineJobFactory.shutdown();
    diskCacheProvider.clearDiskCacheIfCreated();
    activeResources.shutdown();
  }

  /**
   * Allows a request to indicate it no longer is interested in a given load.
   *
   * <p>Non-final for mocking.
   */
  public class LoadStatus {
    private final EngineJob<?> engineJob;
    private final ResourceCallback cb;

    LoadStatus(ResourceCallback cb, EngineJob<?> engineJob) {
      this.cb = cb;
      this.engineJob = engineJob;
    }

    public void cancel() {
      // Acquire the Engine lock so that a new request can't get access to a particular EngineJob
      // just after the EngineJob has been cancelled. Without this lock, we'd allow new requests
      // to find the cancelling EngineJob in our Jobs data structure. With this lock, the EngineJob
      // is both cancelled and removed from Jobs atomically.
      synchronized (Engine.this) {
        engineJob.removeCallback(cb);
      }
    }
  }

  private static class LazyDiskCacheProvider implements DecodeJob.DiskCacheProvider {

    private final DiskCache.Factory factory;
    private volatile DiskCache diskCache;

    LazyDiskCacheProvider(DiskCache.Factory factory) {
      this.factory = factory;
    }

    @VisibleForTesting
    synchronized void clearDiskCacheIfCreated() {
      if (diskCache == null) {
        return;
      }
      diskCache.clear();
    }

    @Override
    public DiskCache getDiskCache() {
      if (diskCache == null) {
        synchronized (this) {
          if (diskCache == null) {
            diskCache = factory.build();
          }
          if (diskCache == null) {
            diskCache = new DiskCacheAdapter();
          }
        }
      }
      return diskCache;
    }
  }

  @VisibleForTesting
  static class DecodeJobFactory {
    @Synthetic final DecodeJob.DiskCacheProvider diskCacheProvider;
    @Synthetic final Pools.Pool<DecodeJob<?>> pool =
        FactoryPools.threadSafe(JOB_POOL_SIZE,
            new FactoryPools.Factory<DecodeJob<?>>() {
          @Override
          public DecodeJob<?> create() {
            return new DecodeJob<>(diskCacheProvider, pool);
          }
        });
    private int creationOrder;

    DecodeJobFactory(DecodeJob.DiskCacheProvider diskCacheProvider) {
      this.diskCacheProvider = diskCacheProvider;
    }

    @SuppressWarnings("unchecked")
    <R> DecodeJob<R> build(GlideContext glideContext,
        Object model,
        EngineKey loadKey,
        Key signature,
        int width,
        int height,
        Class<?> resourceClass,
        Class<R> transcodeClass,
        Priority priority,
        DiskCacheStrategy diskCacheStrategy,
        Map<Class<?>, Transformation<?>> transformations,
        boolean isTransformationRequired,
        boolean isScaleOnlyOrNoTransform,
        boolean onlyRetrieveFromCache,
        Options options,
        DecodeJob.Callback<R> callback) {
      DecodeJob<R> result = Preconditions.checkNotNull((DecodeJob<R>) pool.acquire());
      return result.init(
          glideContext,
          model,
          loadKey,
          signature,
          width,
          height,
          resourceClass,
          transcodeClass,
          priority,
          diskCacheStrategy,
          transformations,
          isTransformationRequired,
          isScaleOnlyOrNoTransform,
          onlyRetrieveFromCache,
          options,
          callback,
          creationOrder++);
    }
  }

  @VisibleForTesting
  static class EngineJobFactory {
    @Synthetic final GlideExecutor diskCacheExecutor;
    @Synthetic final GlideExecutor sourceExecutor;
    @Synthetic final GlideExecutor sourceUnlimitedExecutor;
    @Synthetic final GlideExecutor animationExecutor;
    @Synthetic final EngineJobListener listener;
    @Synthetic final Pools.Pool<EngineJob<?>> pool =
        FactoryPools.threadSafe(
            JOB_POOL_SIZE,
            new FactoryPools.Factory<EngineJob<?>>() {
              @Override
              public EngineJob<?> create() {
                return new EngineJob<>(
                    diskCacheExecutor,
                    sourceExecutor,
                    sourceUnlimitedExecutor,
                    animationExecutor,
                    listener,
                    pool);
              }
            });

    EngineJobFactory(
        GlideExecutor diskCacheExecutor,
        GlideExecutor sourceExecutor,
        GlideExecutor sourceUnlimitedExecutor,
        GlideExecutor animationExecutor,
        EngineJobListener listener) {
      this.diskCacheExecutor = diskCacheExecutor;
      this.sourceExecutor = sourceExecutor;
      this.sourceUnlimitedExecutor = sourceUnlimitedExecutor;
      this.animationExecutor = animationExecutor;
      this.listener = listener;
    }

    @VisibleForTesting
    void shutdown() {
      Executors.shutdownAndAwaitTermination(diskCacheExecutor);
      Executors.shutdownAndAwaitTermination(sourceExecutor);
      Executors.shutdownAndAwaitTermination(sourceUnlimitedExecutor);
      Executors.shutdownAndAwaitTermination(animationExecutor);
    }

    @SuppressWarnings("unchecked")
    <R> EngineJob<R> build(
        Key key,
        boolean isMemoryCacheable,
        boolean useUnlimitedSourceGeneratorPool,
        boolean useAnimationPool,
        boolean onlyRetrieveFromCache) {
      EngineJob<R> result = Preconditions.checkNotNull((EngineJob<R>) pool.acquire());
      return result.init(
          key,
          isMemoryCacheable,
          useUnlimitedSourceGeneratorPool,
          useAnimationPool,
          onlyRetrieveFromCache);
    }
  }
}
```

