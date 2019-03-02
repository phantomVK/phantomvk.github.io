---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

```java
Glide.with(this)
        .load("http://www.abc.com")
        .diskCacheStrategy(DiskCacheStrategy.ALL)
        .skipMemoryCache(true)
        .into(imageView)
```

#### 缓存策略

```java
/**
 * Set of available caching strategies for media.
 */
public abstract class DiskCacheStrategy {

  /**
   * Caches remote data with both {@link #DATA} and {@link #RESOURCE}, and local data with
   * {@link #RESOURCE} only.
   */
  public static final DiskCacheStrategy ALL = new DiskCacheStrategy() {
    @Override
    public boolean isDataCacheable(DataSource dataSource) {
      return dataSource == DataSource.REMOTE;
    }

    @Override
    public boolean isResourceCacheable(boolean isFromAlternateCacheKey, DataSource dataSource,
        EncodeStrategy encodeStrategy) {
      return dataSource != DataSource.RESOURCE_DISK_CACHE && dataSource != DataSource.MEMORY_CACHE;
    }

    @Override
    public boolean decodeCachedResource() {
      return true;
    }

    @Override
    public boolean decodeCachedData() {
      return true;
    }
  };

  /**
   * Saves no data to cache.
   */
  public static final DiskCacheStrategy NONE = new DiskCacheStrategy() {
    @Override
    public boolean isDataCacheable(DataSource dataSource) {
      return false;
    }

    @Override
    public boolean isResourceCacheable(boolean isFromAlternateCacheKey, DataSource dataSource,
        EncodeStrategy encodeStrategy) {
      return false;
    }

    @Override
    public boolean decodeCachedResource() {
      return false;
    }

    @Override
    public boolean decodeCachedData() {
      return false;
    }
  };

  /**
   * Writes retrieved data directly to the disk cache before it's decoded.
   */
  public static final DiskCacheStrategy DATA = new DiskCacheStrategy() {
    @Override
    public boolean isDataCacheable(DataSource dataSource) {
      return dataSource != DataSource.DATA_DISK_CACHE && dataSource != DataSource.MEMORY_CACHE;
    }

    @Override
    public boolean isResourceCacheable(boolean isFromAlternateCacheKey, DataSource dataSource,
        EncodeStrategy encodeStrategy) {
      return false;
    }

    @Override
    public boolean decodeCachedResource() {
      return false;
    }

    @Override
    public boolean decodeCachedData() {
      return true;
    }
  };

  /**
   * Writes resources to disk after they've been decoded.
   */
  public static final DiskCacheStrategy RESOURCE = new DiskCacheStrategy() {
    @Override
    public boolean isDataCacheable(DataSource dataSource) {
      return false;
    }

    @Override
    public boolean isResourceCacheable(boolean isFromAlternateCacheKey, DataSource dataSource,
        EncodeStrategy encodeStrategy) {
      return dataSource != DataSource.RESOURCE_DISK_CACHE && dataSource != DataSource.MEMORY_CACHE;
    }

    @Override
    public boolean decodeCachedResource() {
      return true;
    }

    @Override
    public boolean decodeCachedData() {
      return false;
    }
  };

  /**
   * Tries to intelligently choose a strategy based on the data source of the
   * {@link com.bumptech.glide.load.data.DataFetcher} and the
   * {@link com.bumptech.glide.load.EncodeStrategy} of the
   * {@link com.bumptech.glide.load.ResourceEncoder} (if an
   * {@link com.bumptech.glide.load.ResourceEncoder} is available).
   */
  public static final DiskCacheStrategy AUTOMATIC = new DiskCacheStrategy() {
    @Override
    public boolean isDataCacheable(DataSource dataSource) {
      return dataSource == DataSource.REMOTE;
    }

    @Override
    public boolean isResourceCacheable(boolean isFromAlternateCacheKey, DataSource dataSource,
        EncodeStrategy encodeStrategy) {
      return ((isFromAlternateCacheKey && dataSource == DataSource.DATA_DISK_CACHE)
          || dataSource == DataSource.LOCAL)
          && encodeStrategy == EncodeStrategy.TRANSFORMED;
    }

    @Override
    public boolean decodeCachedResource() {
      return true;
    }

    @Override
    public boolean decodeCachedData() {
      return true;
    }
  };

  /**
   * Returns true if this request should cache the original unmodified data.
   *
   * @param dataSource Indicates where the data was originally retrieved.
   */
  public abstract boolean isDataCacheable(DataSource dataSource);

  /**
   * Returns true if this request should cache the final transformed resource.
   *
   * @param isFromAlternateCacheKey {@code true} if the resource we've decoded was loaded using an
   *                                alternative, rather than the primary, cache key.
   * @param dataSource Indicates where the data used to decode the resource was originally
   *                   retrieved.
   * @param encodeStrategy The {@link EncodeStrategy} the {@link
   * com.bumptech.glide.load.ResourceEncoder} will use to encode the resource.
   */
  public abstract boolean isResourceCacheable(boolean isFromAlternateCacheKey,
      DataSource dataSource, EncodeStrategy encodeStrategy);

  /**
   * Returns true if this request should attempt to decode cached resource data.
   */
  public abstract boolean decodeCachedResource();

  /**
   * Returns true if this request should attempt to decode cached source data.
   */
  public abstract boolean decodeCachedData();
}
```

策略具体行为：

- DiskCacheStrategy.ALL
- DiskCacheStrategy.NONE
- DiskCacheStrategy.DATA
- DiskCacheStrategy.RESOURCE
- DiskCacheStrategy.AUTOMATIC


### # Glide

```java
/**
 * Begin a load with Glide that will tied to the give
 * {@link android.support.v4.app.FragmentActivity}'s lifecycle and that uses the given
 * {@link android.support.v4.app.FragmentActivity}'s default options.
 *
 * @param activity The activity to use.
 * @return A RequestManager for the given FragmentActivity that can be used to start a load.
 */
@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}
```

Glide

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
  Preconditions.checkNotNull(
      context,
      "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
          + "returns null (which usually occurs when getActivity() is called before the Fragment "
          + "is attached or after the Fragment is destroyed).");
  return Glide.get(context).getRequestManagerRetriever();
}
```

```java
@NonNull
public static Glide get(@NonNull Context context) {
  if (glide == null) {
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context);
      }
    }
  }

  return glide;
}
```

```java
@NonNull
public RequestManagerRetriever getRequestManagerRetriever() {
  return requestManagerRetriever;
}
```

## RequestManagerRetriever

```java
@NonNull
public RequestManager get(@NonNull Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }

  return getApplicationManager(context);
}

@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(
        activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

@NonNull
public RequestManager get(@NonNull Fragment fragment) {
  Preconditions.checkNotNull(fragment.getActivity(),
        "You cannot start a load on a fragment before it is attached or after it is destroyed");
  if (Util.isOnBackgroundThread()) {
    return get(fragment.getActivity().getApplicationContext());
  } else {
    FragmentManager fm = fragment.getChildFragmentManager();
    return supportFragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
  }
}

@SuppressWarnings("deprecation")
@NonNull
public RequestManager get(@NonNull Activity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    android.app.FragmentManager fm = activity.getFragmentManager();
    return fragmentGet(
        activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

@SuppressWarnings("deprecation")
@NonNull
public RequestManager get(@NonNull View view) {
  if (Util.isOnBackgroundThread()) {
    return get(view.getContext().getApplicationContext());
  }

  Preconditions.checkNotNull(view);
  Preconditions.checkNotNull(view.getContext(),
      "Unable to obtain a request manager for a view without a Context");
  Activity activity = findActivity(view.getContext());
  // The view might be somewhere else, like a service.
  if (activity == null) {
    return get(view.getContext().getApplicationContext());
  }

  // Support Fragments.
  // Although the user might have non-support Fragments attached to FragmentActivity, searching
  // for non-support Fragments is so expensive pre O and that should be rare enough that we
  // prefer to just fall back to the Activity directly.
  if (activity instanceof FragmentActivity) {
    Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
    return fragment != null ? get(fragment) : get(activity);
  }

  // Standard Fragments.
  android.app.Fragment fragment = findFragment(view, activity);
  if (fragment == null) {
    return get(activity);
  }
  return get(fragment);
}

```

从上面取出 __FragmentActivity__ 的方法进行解释

```java
@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(
        activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}
```

```java
@SuppressWarnings({"deprecation", "DeprecatedIsStillUsed"})
@Deprecated
@NonNull
private RequestManager fragmentGet(@NonNull Context context,
    @NonNull android.app.FragmentManager fm,
    @Nullable android.app.Fragment parentHint,
    boolean isParentVisible) {
  RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
  }
  return requestManager;
}

@NonNull
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
  SupportRequestManagerFragment current =
      getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
  }
  return requestManager;
}
```

```java
@NonNull
private RequestManager getApplicationManager(@NonNull Context context) {
  // Either an application context or we're on a background thread.
  if (applicationManager == null) {
    synchronized (this) {
      if (applicationManager == null) {
        // Normally pause/resume is taken care of by the fragment we add to the fragment or
        // activity. However, in this case since the manager attached to the application will not
        // receive lifecycle events, we must force the manager to start resumed using
        // ApplicationLifecycle.

        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context.getApplicationContext());
        applicationManager =
            factory.build(
                glide,
                new ApplicationLifecycle(),
                new EmptyRequestManagerTreeNode(),
                context.getApplicationContext());
      }
    }
  }

  return applicationManager;
}
```

```java
/**
 * An interface for listener to {@link android.app.Fragment} and {@link android.app.Activity}
 * lifecycle events.
 */
public interface LifecycleListener {

  /**
   * Callback for when {@link android.app.Fragment#onStart()}} or {@link
   * android.app.Activity#onStart()} is called.
   */
  void onStart();

  /**
   * Callback for when {@link android.app.Fragment#onStop()}} or {@link
   * android.app.Activity#onStop()}} is called.
   */
  void onStop();

  /**
   * Callback for when {@link android.app.Fragment#onDestroy()}} or {@link
   * android.app.Activity#onDestroy()} is called.
   */
  void onDestroy();
}
```

```java
/**
 * A class for tracking, canceling, and restarting in progress, completed, and failed requests.
 *
 * <p>This class is not thread safe and must be accessed on the main thread.
 */
public class RequestTracker {
  private static final String TAG = "RequestTracker";
  // Most requests will be for views and will therefore be held strongly (and safely) by the view
  // via the tag. However, a user can always pass in a different type of target which may end up not
  // being strongly referenced even though the user still would like the request to finish. Weak
  // references are therefore only really functional in this context for view targets. Despite the
  // side affects, WeakReferences are still essentially required. A user can always make repeated
  // requests into targets other than views, or use an activity manager in a fragment pager where
  // holding strong references would steadily leak bitmaps and/or views.
  private final Set<Request> requests =
      Collections.newSetFromMap(new WeakHashMap<Request, Boolean>());
  // A set of requests that have not completed and are queued to be run again. We use this list to
  // maintain hard references to these requests to ensure that they are not garbage collected
  // before they start running or while they are paused. See #346.
  @SuppressWarnings("MismatchedQueryAndUpdateOfCollection")
  private final List<Request> pendingRequests = new ArrayList<>();
  private boolean isPaused;

  /**
   * Starts tracking the given request.
   */
  public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
      request.begin();
    } else {
      request.clear();
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      pendingRequests.add(request);
    }
  }

  @VisibleForTesting
  void addRequest(Request request) {
    requests.add(request);
  }

  /**
   * Stops tracking the given request, clears, and recycles it, and returns {@code true} if the
   * request was removed or invalid or {@code false} if the request was not found.
   */
  public boolean clearRemoveAndRecycle(@Nullable Request request) {
    // It's safe for us to recycle because this is only called when the user is explicitly clearing
    // a Target so we know that there are no remaining references to the Request.
    return clearRemoveAndMaybeRecycle(request, /*isSafeToRecycle=*/ true);
  }

  private boolean clearRemoveAndMaybeRecycle(@Nullable Request request, boolean isSafeToRecycle) {
     if (request == null) {
       // If the Request is null, the request is already cleared and we don't need to search further
       // for its owner.
      return true;
    }
    boolean isOwnedByUs = requests.remove(request);
    // Avoid short circuiting.
    isOwnedByUs = pendingRequests.remove(request) || isOwnedByUs;
    if (isOwnedByUs) {
      request.clear();
      if (isSafeToRecycle) {
        request.recycle();
      }
    }
    return isOwnedByUs;
  }

  /**
   * Returns {@code true} if requests are currently paused, and {@code false} otherwise.
   */
  public boolean isPaused() {
    return isPaused;
  }

  /**
   * Stops any in progress requests.
   */
  public void pauseRequests() {
    isPaused = true;
    for (Request request : Util.getSnapshot(requests)) {
      if (request.isRunning()) {
        request.clear();
        pendingRequests.add(request);
      }
    }
  }

  /** Stops any in progress requests and releases bitmaps associated with completed requests. */
  public void pauseAllRequests() {
    isPaused = true;
    for (Request request : Util.getSnapshot(requests)) {
      if (request.isRunning() || request.isComplete()) {
        request.clear();
        pendingRequests.add(request);
      }
    }
  }

  /**
   * Starts any not yet completed or failed requests.
   */
  public void resumeRequests() {
    isPaused = false;
    for (Request request : Util.getSnapshot(requests)) {
      // We don't need to check for cleared here. Any explicit clear by a user will remove the
      // Request from the tracker, so the only way we'd find a cleared request here is if we cleared
      // it. As a result it should be safe for us to resume cleared requests.
      if (!request.isComplete() && !request.isRunning()) {
        request.begin();
      }
    }
    pendingRequests.clear();
  }

  /**
   * Cancels all requests and clears their resources.
   *
   * <p>After this call requests cannot be restarted.
   */
  public void clearRequests() {
    for (Request request : Util.getSnapshot(requests)) {
      // It's unsafe to recycle the Request here because we don't know who might else have a
      // reference to it.
      clearRemoveAndMaybeRecycle(request, /*isSafeToRecycle=*/ false);
    }
    pendingRequests.clear();
  }

  /**
   * Restarts failed requests and cancels and restarts in progress requests.
   */
  public void restartRequests() {
    for (Request request : Util.getSnapshot(requests)) {
      if (!request.isComplete() && !request.isCleared()) {
        request.clear();
        if (!isPaused) {
          request.begin();
        } else {
          // Ensure the request will be restarted in onResume.
          pendingRequests.add(request);
        }
      }
    }
  }

  @Override
  public String toString() {
    return super.toString() + "{numRequests=" + requests.size() + ", isPaused=" + isPaused + "}";
  }
}
```

## RequestManager


Bitmap、Drawable、String、Uri、File、Integer、URL等等，下面选择String


```java
/**
 * Equivalent to calling {@link #asDrawable()} and then {@link RequestBuilder#load(String)}.
 *
 * @return A new request builder for loading a {@link Drawable} using the given model.
 */
@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
```

```java
/**
 * Attempts to load the resource using any registered
 * {@link com.bumptech.glide.load.ResourceDecoder}s
 * that can decode the given resource class or any subclass of the given resource class.
 *
 * @param resourceClass The resource to decode.
 * @return A new request builder for loading the given resource class.
 */
@NonNull
@CheckResult
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
```



RequestBuilder

```java
/**
 * Returns a request builder to load the given {@link java.lang.String}.
 *
 * <p> Note - this method caches data using only the given String as the cache key. If the data is
 * a Uri outside of your control, or you otherwise expect the data represented by the given String
 * to change without the String identifier changing, Consider using
 * {@link com.bumptech.glide.request.RequestOptions#signature(com.bumptech.glide.load.Key)} to
 * mixin a signature you create that identifies the data currently at the given String that will
 * invalidate the cache if that data changes. Alternatively, using
 * {@link com.bumptech.glide.load.engine.DiskCacheStrategy#NONE} and/or
 * {@link com.bumptech.glide.request.RequestOptions#skipMemoryCache(boolean)} may be
 * appropriate.
 * </p>
 *
 * @see #load(Object)
 *
 * @param string A file path, or a uri or url handled by
 * {@link com.bumptech.glide.load.model.UriLoader}.
 */
@NonNull
@Override
@CheckResult
public RequestBuilder<TranscodeType> load(@Nullable String string) {
  return loadGeneric(string);
}
```

```java
@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```

