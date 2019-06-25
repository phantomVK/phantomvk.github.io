---
layout:     post
title:      "Glide生命周期管理"
date:       2019-03-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Glide
---

## 前言

图片加载时只需调用以下代码，__Glide__ 会自动完成下载、缓存、缩放、展示等流程。

```java
Glide.with(this)
        .load("http://www.abc.com")
        .into(imageView)
```

其中包含应用进入后台，图片会暂停加载的策略。通过这篇文章，探究 __Glide__ 是如何实现开发者不主动触发逻辑，就能实现任务生命周期自动管理的奥秘。__Glide 4.9.0__

## Glide

上面的`Glide.with(this)`根据传入实参类型，例如：__FragmentActivity__、__Fragment__、__Context__ 等不同选择目标重载方法。这里通过 __FragmentActivity__ 进行解析，常用 __AppCompatActivity__ 的父类是 __FragmentActivity__，便于理解。

```java
public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}
```

检查参数提供的 __Context__ 是否为空：当 __Fragment__ 与页面绑定前或解除绑定后，这些生命周期内获得宿主 __Activity__ 为空。获取 __Glide__ 单例并提取里面的 __RequestManagerRetriever__ 实例。

```java
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

## RequestManagerRetriever

忽略 __Glide__ 首次初始化流程，来到 __RequestManagerRetriever__ 获取 __RequestManager__ 实例的流程。

```java
public RequestManager get(@NonNull FragmentActivity activity) {
  // 前后台判断
  if (Util.isOnBackgroundThread()) {
    // 页面在后台时获取ApplicationContext
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    // 从Activity实例获取FragmentManager
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(activity, fm, null, isActivityVisible(activity));
  }
}
```

从上面可见条件判断根据应用是否在前台走分支，而本文仅关心应用在前台的逻辑。进入分支后可见从 __Activity__ 实例获取 __FragmentManager__。 

```java
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
  // 看下文
  SupportRequestManagerFragment current =
      getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
  }
  return requestManager;
}
```

以下方法从 __Activity__ 的 __FragmentManager__ 中查找是否存在名为 __FRAGMENT_TAG__ 的 __Fragment__ 实例。这个实例相当于一个隐形的钩子，挂在 __Activity__ 监听界面生命周期变化，并回调 __Glide__ 图片加载策略。

```java
static final String FRAGMENT_TAG = "com.bumptech.glide.manager";
```

实例不存在就创建 __Fragment__ 实例，这个实例类型为 __SupportRequestManagerFragment__。

```java
private SupportRequestManagerFragment getSupportRequestManagerFragment(
    @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
  // 首先用FRAGMENT_TAG字符串的方式从fm里面找Fragment
  SupportRequestManagerFragment current =
      (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
  
  // 字符串的方式找不到
  if (current == null) {
    // 每个FragmentActivity只有一个FragmentManager;
    // 每个FragmentManager只存一个SupportRequestManagerFragment;
    // 所以试着用FragmentManager获取SupportRequestManagerFragment实例
    current = pendingSupportRequestManagerFragments.get(fm);
    // 判断实例为空则需构建Fragment实例并加入到Activity
    if (current == null) {
      // 创建新实例
      current = new SupportRequestManagerFragment();
      current.setParentFragmentHint(parentHint);
      // Activity可见，触发Fragment的onStart()逻辑调用
      if (isParentVisible) {
        current.getGlideLifecycle().onStart();
      }
      pendingSupportRequestManagerFragments.put(fm, current);
      // 把SupportRequestManagerFragment加入到Activity的FragmentManager中
      fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
      handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
    }
  }
  return current;
}
```

所有创建的 __SupportRequestManagerFragment__ 都保存在以下哈希表中，即自行保存一份关系列表

```java
final Map<FragmentManager, SupportRequestManagerFragment> pendingSupportRequestManagerFragments = new HashMap<>();
```

## SupportRequestManagerFragment

接下来看看上面提及的 __SupportRequestManagerFragment__。

从签名可知此类继承父类为 __Fragment__。但不像一般 __Fragment__ 包含UI界面，这个实例仅包含生命周期的相关回调操作。说白了，就是借用 __Fragment__ 当钩子达到监听 __Activity__ 生命周期的目的。

```java
public class SupportRequestManagerFragment extends Fragment {
  // 生命周期回调
  private final ActivityFragmentLifecycle lifecycle;
  private final RequestManagerTreeNode requestManagerTreeNode =
      new SupportFragmentRequestManagerTreeNode();
  private final Set<SupportRequestManagerFragment> childRequestManagerFragments = new HashSet<>();

  @Nullable private SupportRequestManagerFragment rootRequestManagerFragment;
  @Nullable private RequestManager requestManager;
  @Nullable private Fragment parentFragmentHint;

  public SupportRequestManagerFragment() {
    this(new ActivityFragmentLifecycle());
  }

  @VisibleForTesting
  @SuppressLint("ValidFragment")
  public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
    this.lifecycle = lifecycle;
  }
    
  .....

  @Override
  public void onAttach(Context context) {
    super.onAttach(context);
    try {
      registerFragmentWithRoot(getActivity());
    } catch (IllegalStateException e) {
      // OnAttach can be called after the activity is destroyed, see #497.
    }
  }
    
  // 注意下面不同生命周期对lifecycle的调用

  @Override
  public void onDetach() {
    super.onDetach();
    parentFragmentHint = null;
    unregisterFragmentWithRoot();
  }

  @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }
}
```

## ActivityFragmentLifecycle

最后进入 __ActivityFragmentLifecycle__ 实现类，这个类其实就是观察者模式的具体实现。所有需要加载图片的任务，会创建 __LifecycleListener__ 监听器放入 __ActivityFragmentLifecycle__。

```java
class ActivityFragmentLifecycle implements Lifecycle {
  // 监听器集合
  private final Set<LifecycleListener> lifecycleListeners =
      Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());

  private boolean isStarted;
  private boolean isDestroyed;

  // 把给定的监听器添加到监听器列表，以便接收生命周期事件的通知
  // 最新的生命周期事件会在所有注册的监听器上触发
  // 若activity或fragment被stop，会调用LifecycleListener.onStop()
  // onStart和onDestroy生命周期有类似操作
  @Override
  public void addListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.add(listener);

    // 把最新界面状态通知给新来的LifecycleListener
    if (isDestroyed) {
      listener.onDestroy();
    } else if (isStarted) {
      listener.onStart();
    } else {
      listener.onStop();
    }
  }

  @Override
  public void removeListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.remove(listener);
  }

  // 下发生命周期通知，观察者模式
  void onStart() {
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
  }

  void onStop() {
    isStarted = false;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStop();
    }
  }

  void onDestroy() {
    isDestroyed = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onDestroy();
    }
  }
}
```

最后，定制的 __Fragment__ 监听 __Activity__ 生命周期变化，告知 __ActivityFragmentLifecycle__ ，随后 __ActivityFragmentLifecycle__ 回调所有注册的监听器，实现应用进入后台暂停加载等功能。

下面就是 __ActivityFragmentLifecycle__ 回调的逻辑类，负责请求的跟踪、取消、重启、完成和失败的管理。

```java
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
  // 保存未完成或准备运行请求的列表，用于持有这些请求的强引用
  // 以保证请求在开始执行之前或暂停的时候不会被垃圾回收
  @SuppressWarnings("MismatchedQueryAndUpdateOfCollection")
  private final List<Request> pendingRequests = new ArrayList<>();
  private boolean isPaused;

  // 开始跟踪指定请求
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

  // 停止跟踪指定请求，并进行清除和回收工作
  // 请求已被移除、失效返回true；请求找不到时返回false
  public boolean clearRemoveAndRecycle(@Nullable Request request) {
    // 此时回收请求是安全的，因为这个操作由用户主动清除时触发，因此可知没有其他持有请求的引用
    return clearRemoveAndMaybeRecycle(request, /*isSafeToRecycle=*/ true);
  }

  private boolean clearRemoveAndMaybeRecycle(@Nullable Request request, boolean isSafeToRecycle) {
     if (request == null) {
      // 请求为空表示请求已被清除，不再需要查找请求的所有者
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

  // 暂停所有正在进行的请求
  public void pauseRequests() {
    isPaused = true;
    for (Request request : Util.getSnapshot(requests)) {
      if (request.isRunning()) {
        request.clear();
        pendingRequests.add(request);
      }
    }
  }

  // 暂停所有正在进行的请求，释放已完成任务的关联bitmaps
  public void pauseAllRequests() {
    isPaused = true;
    for (Request request : Util.getSnapshot(requests)) {
      if (request.isRunning() || request.isComplete()) {
        request.clear();
        pendingRequests.add(request);
      }
    }
  }

  // 重启所有未完成或曾经失败的任务
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

  // 取消所有请求并清理任务持有的资源，取消的请求将不能再次重启
  public void clearRequests() {
    for (Request request : Util.getSnapshot(requests)) {
      // 在这里回收请求的不安全的，因为不知道别的地方是否持有请求的引用
      // 方法形参isSafeToRecycle赋值为false
      clearRemoveAndMaybeRecycle(request, false);
    }
    pendingRequests.clear();
  }

  // 从起失败的、取消的、正在进行的请求
  public void restartRequests() {
    for (Request request : Util.getSnapshot(requests)) {
      if (!request.isComplete() && !request.isCleared()) {
        request.clear();
        if (!isPaused) {
          request.begin();
        } else {
          // 确保请求在onResume生命周期正常重启
          pendingRequests.add(request);
        }
      }
    }
  }
}
```

## 总结

通过上述流程可知，__Glide__ 的图片加载生命周期管理依赖传入的 __Context__ 实例。如果实例类型为 __Activity__，__Glide__ 就能注册一个没有界面的 __Fragment__ 到 __Activity__，达到监听生命周期变化的目的。

因此，该参数应尽可能传入 __Activity__ 或 __Fragment__ 对象，最少也得是 __View__ 持有的 __Context__，而不是 __ApplicationContext__。不然图片的生命周期会超过界面实际的生命周期。

![Glide_lifecycle_hierarchy](/img/Glide/Glide_lifecycle_hierarchy.png)

最后，__Glide__ 监听界面生命周期变化没有什么奥秘，就是把定制的 __Fragment__ 添加到图片所在的 __Activity__ 中，利用系统机制监听周期变化，在不同周期通知图片进行不同操作