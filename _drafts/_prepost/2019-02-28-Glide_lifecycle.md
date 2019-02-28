---
layout:     post
title:      "Glide生命周期"
date:       2019-02-28
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Glide
---

图片加载时只需调用以下代码，__Glide__ 就会全自动完成下载、缓存、缩放、展示等流程。

```java
Glide.with(this)
        .load("http://www.abc.com")
        .into(imageView)
```

其中包含应用进入后台，图片会暂停加载的策略。通过这篇文章，探究 __Glide__ 是如何实现开发者不主动触发逻辑，就能达到任务生命周期自动管理的奥秘。

## Glide

上面的`Glide.with(this)`根据传入实参类型，例如：__FragmentActivity__、__Fragment__、__Context__ 等不同选择目标重载方法。这里选择 __FragmentActivity__ 进行解析，因为 __AppCompatActivity__ 的父类是 __FragmentActivity__。

```java
public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}
```

先获取 __Glide__ 单例，然后从这个单例里获取 __RequestManagerRetriever__ 的实例。

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

忽略 __Glide__ 首次初始化流程，终于从 __RequestManagerRetriever__ 获取 __RequestManager__ 实例。

```java
public RequestManager get(@NonNull FragmentActivity activity) {
  // 前后台判断
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    // 从Activity实例获取FragmentManager
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(activity, fm, null, isActivityVisible(activity));
  }
}
```

从上面可见，条件判断语句根据应用是否在前台走分支。因为本文关心是应用在前台的逻辑，所以直接走 __else__。

进入分支后可见从 __Activity__ 实例获取 __FragmentManager__。 

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
  SupportRequestManagerFragment current =
      (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
  // 判断实例为空则需构建Fragment实例并加入到Activity
  if (current == null) {
    // 用FragmentManager获取SupportRequestManagerFragment实例
    current = pendingSupportRequestManagerFragments.get(fm);
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

所有创建的 __SupportRequestManagerFragment__ 都保存在以下哈希表中

```java
final Map<FragmentManager, SupportRequestManagerFragment> pendingSupportRequestManagerFragments = new HashMap<>();
```

## SupportRequestManagerFragment

接下来看看上面提及的 __SupportRequestManagerFragment__。从签名可知继承父类 __Fragment__，但不像一般 __Fragment__ 包含填充的UI界面，而是仅包含生命周期的相关回调操作。说白了，就是借用 __Fragment__ 达到监听 __Activity__ 生命周期的目标。

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

最后进入 __ActivityFragmentLifecycle__ 实现类，这个类其实就是观察者模式的具体实现。所有需要加载图片的任务，自行创建 __LifecycleListener__ 放入 __ActivityFragmentLifecycle__ 中。

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

## 总结

通过上述流程可知，__Glide__ 的图片加载生命周期管理依赖传入的实例。

如果实例类型为 __Activity__，__Glide__ 就能向里面注册一个没有界面的 __Fragment__ 达到监听生命周期变化的目的。所以，该参数应尽可能传入 __Activity__ 或 __Fragment__ 对象，最少也得是 __View__ 持有的 __Context__，而不是 __ApplicationContext__。

![Glide_lifecycle_hierarchy](/img/Glide/Glide_lifecycle_hierarchy.png)

最后，__Glide__ 监听界面生命周期变化没有什么奥秘，就是把定制的 __Fragment__ 添加到图片所在的 __Activity__ 中，利用系统机制监听周期变化，在不同周期的不同加载操作。