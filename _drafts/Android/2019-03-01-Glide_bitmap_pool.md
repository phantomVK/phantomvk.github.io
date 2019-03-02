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

## MemorySizeCalculator

返回以这台设备的条件为准，按字节为单位的bitmap池推荐容量值

```java
public int getBitmapPoolSize() {
  return bitmapPoolSize;
}
```

上述方法没有实际计算逻辑，只是返回下面这个数据成员的大小。这个数组用 __final__ 关键字修饰，可知这个变量只能在首次初始化时计算值。

```java
private final int bitmapPoolSize;
```

实际上，这个 __bitmapPoolSize__ 确实在 __MemorySizeCalculator__ 初始化的时候计算的，看具体构造方法

```java
MemorySizeCalculator(MemorySizeCalculator.Builder builder) {
  this.context = builder.context;

  arrayPoolSize =
      isLowMemoryDevice(builder.activityManager)
          ? builder.arrayPoolSizeBytes / LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR
          : builder.arrayPoolSizeBytes;
  int maxSize =
      getMaxSize(
          builder.activityManager, builder.maxSizeMultiplier, builder.lowMemoryMaxSizeMultiplier);

  // 获取当前设备屏幕宽度
  int widthPixels = builder.screenDimensions.getWidthPixels();
  // 获取当前设备屏幕高度
  int heightPixels = builder.screenDimensions.getHeightPixels();
  // 根据ARGB_8888为标准算屏幕像素的内存占用大小
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
```

## LruBitmapPool

__GlideBuilder__ 的 __build()__ 逻辑内， 

```java
if (bitmapPool == null) {
  int size = memorySizeCalculator.getBitmapPoolSize();
  if (size > 0) {
    bitmapPool = new LruBitmapPool(size);
  } else {
    bitmapPool = new BitmapPoolAdapter();
  }
}
```


```java
/**
 * An {@link com.bumptech.glide.load.engine.bitmap_recycle.BitmapPool} implementation that uses an
 * {@link com.bumptech.glide.load.engine.bitmap_recycle.LruPoolStrategy} to bucket {@link Bitmap}s
 * and then uses an LRU eviction policy to evict {@link android.graphics.Bitmap}s from the least
 * recently used bucket in order to keep the pool below a given maximum size limit.
 */
public class LruBitmapPool implements BitmapPool {
  private static final String TAG = "LruBitmapPool";
  private static final Bitmap.Config DEFAULT_CONFIG = Bitmap.Config.ARGB_8888;

  private final LruPoolStrategy strategy;
  private final Set<Bitmap.Config> allowedConfigs;
  private final long initialMaxSize;
  private final BitmapTracker tracker;

  private long maxSize;
  private long currentSize;
  private int hits;
  private int misses;
  private int puts;
  private int evictions;

  // Exposed for testing only.
  LruBitmapPool(long maxSize, LruPoolStrategy strategy, Set<Bitmap.Config> allowedConfigs) {
    this.initialMaxSize = maxSize;
    this.maxSize = maxSize;
    this.strategy = strategy;
    this.allowedConfigs = allowedConfigs;
    this.tracker = new NullBitmapTracker();
  }

  /**
   * Constructor for LruBitmapPool.
   *
   * @param maxSize The initial maximum size of the pool in bytes.
   */
  public LruBitmapPool(long maxSize) {
    this(maxSize, getDefaultStrategy(), getDefaultAllowedConfigs());
  }

  /**
   * Constructor for LruBitmapPool.
   *
   * @param maxSize        The initial maximum size of the pool in bytes.
   * @param allowedConfigs A white listed put of {@link android.graphics.Bitmap.Config} that are
   *                       allowed to be put into the pool. Configs not in the allowed put will be
   *                       rejected.
   */
  // Public API.
  @SuppressWarnings("unused")
  public LruBitmapPool(long maxSize, Set<Bitmap.Config> allowedConfigs) {
    this(maxSize, getDefaultStrategy(), allowedConfigs);
  }

  @Override
  public long getMaxSize() {
    return maxSize;
  }

  @Override
  public synchronized void setSizeMultiplier(float sizeMultiplier) {
    maxSize = Math.round(initialMaxSize * sizeMultiplier);
    evict();
  }

  @Override
  public synchronized void put(Bitmap bitmap) {
    if (bitmap == null) {
      throw new NullPointerException("Bitmap must not be null");
    }
    if (bitmap.isRecycled()) {
      throw new IllegalStateException("Cannot pool recycled bitmap");
    }
    if (!bitmap.isMutable() || strategy.getSize(bitmap) > maxSize
        || !allowedConfigs.contains(bitmap.getConfig())) {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Reject bitmap from pool"
                + ", bitmap: " + strategy.logBitmap(bitmap)
                + ", is mutable: " + bitmap.isMutable()
                + ", is allowed config: " + allowedConfigs.contains(bitmap.getConfig()));
      }
      bitmap.recycle();
      return;
    }

    final int size = strategy.getSize(bitmap);
    strategy.put(bitmap);
    tracker.add(bitmap);

    puts++;
    currentSize += size;

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Put bitmap in pool=" + strategy.logBitmap(bitmap));
    }
    dump();

    evict();
  }

  private void evict() {
    trimToSize(maxSize);
  }

  @Override
  @NonNull
  public Bitmap get(int width, int height, Bitmap.Config config) {
    Bitmap result = getDirtyOrNull(width, height, config);
    if (result != null) {
      // Bitmaps in the pool contain random data that in some cases must be cleared for an image
      // to be rendered correctly. we shouldn't force all consumers to independently erase the
      // contents individually, so we do so here. See issue #131.
      result.eraseColor(Color.TRANSPARENT);
    } else {
      result = createBitmap(width, height, config);
    }

    return result;
  }

  @NonNull
  @Override
  public Bitmap getDirty(int width, int height, Bitmap.Config config) {
    Bitmap result = getDirtyOrNull(width, height, config);
    if (result == null) {
      result = createBitmap(width, height, config);
    }
    return result;
  }

  @NonNull
  private static Bitmap createBitmap(int width, int height, @Nullable Bitmap.Config config) {
    return Bitmap.createBitmap(width, height, config != null ? config : DEFAULT_CONFIG);
  }

  @TargetApi(Build.VERSION_CODES.O)
  private static void assertNotHardwareConfig(Bitmap.Config config) {
    // Avoid short circuiting on sdk int since it breaks on some versions of Android.
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
      return;
    }

    if (config == Bitmap.Config.HARDWARE) {
      throw new IllegalArgumentException("Cannot create a mutable Bitmap with config: " + config
          + ". Consider setting Downsampler#ALLOW_HARDWARE_CONFIG to false in your RequestOptions"
          + " and/or in GlideBuilder.setDefaultRequestOptions");
    }
  }

  @Nullable
  private synchronized Bitmap getDirtyOrNull(
      int width, int height, @Nullable Bitmap.Config config) {
    assertNotHardwareConfig(config);
    // Config will be null for non public config types, which can lead to transformations naively
    // passing in null as the requested config here. See issue #194.
    final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
    if (result == null) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Missing bitmap=" + strategy.logBitmap(width, height, config));
      }
      misses++;
    } else {
      hits++;
      currentSize -= strategy.getSize(result);
      tracker.remove(result);
      normalize(result);
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Get bitmap=" + strategy.logBitmap(width, height, config));
    }
    dump();

    return result;
  }

  // Setting these two values provides Bitmaps that are essentially equivalent to those returned
  // from Bitmap.createBitmap.
  private static void normalize(Bitmap bitmap) {
    bitmap.setHasAlpha(true);
    maybeSetPreMultiplied(bitmap);
  }

  @TargetApi(Build.VERSION_CODES.KITKAT)
  private static void maybeSetPreMultiplied(Bitmap bitmap) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      bitmap.setPremultiplied(true);
    }
  }

  @Override
  public void clearMemory() {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "clearMemory");
    }
    trimToSize(0);
  }

  @SuppressLint("InlinedApi")
  @Override
  public void trimMemory(int level) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "trimMemory, level=" + level);
    }
    if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
      clearMemory();
    } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
        || level == android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {
      trimToSize(getMaxSize() / 2);
    }
  }

  private synchronized void trimToSize(long size) {
    while (currentSize > size) {
      final Bitmap removed = strategy.removeLast();
      // TODO: This shouldn't ever happen, see #331.
      if (removed == null) {
        if (Log.isLoggable(TAG, Log.WARN)) {
          Log.w(TAG, "Size mismatch, resetting");
          dumpUnchecked();
        }
        currentSize = 0;
        return;
      }
      tracker.remove(removed);
      currentSize -= strategy.getSize(removed);
      evictions++;
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Evicting bitmap=" + strategy.logBitmap(removed));
      }
      dump();
      removed.recycle();
    }
  }

  private void dump() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      dumpUnchecked();
    }
  }

  private void dumpUnchecked() {
    Log.v(TAG, "Hits=" + hits + ", misses=" + misses + ", puts=" + puts + ", evictions=" + evictions
        + ", currentSize=" + currentSize + ", maxSize=" + maxSize + "\nStrategy=" + strategy);
  }

  private static LruPoolStrategy getDefaultStrategy() {
    final LruPoolStrategy strategy;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      strategy = new SizeConfigStrategy();
    } else {
      strategy = new AttributeStrategy();
    }
    return strategy;
  }

  @TargetApi(Build.VERSION_CODES.O)
  private static Set<Bitmap.Config> getDefaultAllowedConfigs() {
    Set<Bitmap.Config> configs = new HashSet<>(Arrays.asList(Bitmap.Config.values()));
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      // GIFs, among other types, end up with a native Bitmap config that doesn't map to a java
      // config and is treated as null in java code. On KitKat+ these Bitmaps can be reconfigured
      // and are suitable for re-use.
      configs.add(null);
    }
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      configs.remove(Bitmap.Config.HARDWARE);
    }
    return Collections.unmodifiableSet(configs);
  }

  private interface BitmapTracker {
    void add(Bitmap bitmap);

    void remove(Bitmap bitmap);
  }

  @SuppressWarnings("unused")
  // Only used for debugging
  private static class ThrowingBitmapTracker implements BitmapTracker {
    private final Set<Bitmap> bitmaps = Collections.synchronizedSet(new HashSet<Bitmap>());

    @Override
    public void add(Bitmap bitmap) {
      if (bitmaps.contains(bitmap)) {
        throw new IllegalStateException(
            "Can't add already added bitmap: " + bitmap + " [" + bitmap.getWidth() + "x" + bitmap
                .getHeight() + "]");
      }
      bitmaps.add(bitmap);
    }

    @Override
    public void remove(Bitmap bitmap) {
      if (!bitmaps.contains(bitmap)) {
        throw new IllegalStateException("Cannot remove bitmap not in tracker");
      }
      bitmaps.remove(bitmap);
    }
  }

  private static final class NullBitmapTracker implements BitmapTracker {

    @Synthetic
    NullBitmapTracker() { }

    @Override
    public void add(Bitmap bitmap) {
      // Do nothing.
    }

    @Override
    public void remove(Bitmap bitmap) {
      // Do nothing.
    }
  }
}
```

## LruResourceCache

```java
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

## Engine

```java
[/**
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

  // 通过多个参数计算该图片的Key；如果参数完全一样，下一也能算出相同Key
  EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
      resourceClass, transcodeClass, options);

  // 算出key先从loadFromActiveResources()获取活跃的资源
  EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
  if (active != null) {
    cb.onResourceReady(active, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
  }

  // 上面方法没有成功获取资源，尝试loadFromCache()
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
```

```java
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
```

```java
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
```

