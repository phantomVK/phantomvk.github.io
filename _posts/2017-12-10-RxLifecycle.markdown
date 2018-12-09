---
layout:     post
title:      "Android源码系列(7) -- RxLifecycle"
date:       2017-12-10
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

# 一、前言

## 1.1 简介

`RxJava`带来编码流畅性，`map`、`flatmap`、`filter`等通过链式操作，既避免回调地狱又解决线程频繁切换的问题。应用最多的场景如网络访问后回调主线程显示结果，如果界面已经退出但订阅没有解除，那`Activity`或`Fragment`句柄会被`Observer`长期持有导致内存泄漏。

## 1.2 传统方案

传统解决办法是构造`RxJava`时管理`Disposable`，`onDestroy`集中解除绑定：

```java
private CompositeDisposable mCompositeDisposable = new CompositeDisposable();

@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // This is an example that do something useless.
    Disposable disposable = Observable.just(1)
            .map(integer -> "Map to String: " + integer)
            .filter(s -> !s.isEmpty())
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(s -> Toast.show(this, s));
    
    mCompositeDisposable.add(disposable); // 集中管理
}

@Override
protected void onDestroy() {
    super.onDestroy();
    
    if (mCompositeDisposable != null) { // 解除所有订阅
        mCompositeDisposable.dispose();
    }
}
```

上述方案缺点模板代码多不简洁。

## 1.3 新办法

为了避免写模板代码引入`RxLifecycle`，代码增加`.compose(bindToLifecycle())`。

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // This is an example that do something useless.
    Observable.just(1)
            .compose(bindToLifecycle())
            .map(integer -> "Map to string: " + integer)
            .filter(s -> !s.isEmpty())
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(s -> ToastUtils.show(this, s));
}
```

`RxLifecycle`缺点这篇文章不会详细说，有兴趣的可以阅读原作者写的文章(英文)：[为什么不要用RxLifecycle](http://blog.danlew.net/2017/08/02/why-not-rxlifecycle/)。

# 二、源码

## 2.1 类签名

抽象类的父类是`AppCompatActivity`，实现`LifecycleProvider<ActivityEvent>`接口。

```java
public abstract class RxAppCompatActivity extends AppCompatActivity
    implements LifecycleProvider<ActivityEvent>
```

实现`LifecycleProvider<ActivityEvent>`接口增加了三个方法，并维护了一个`BehaviorSubject`实例，该实例是`Subject`的子类，实现在`RxJava`中，`ActivityEvent`放在文章最后简略展示。

```java
private final BehaviorSubject<ActivityEvent> lifecycleSubject = BehaviorSubject.create();
```


## 2.2 抽象接口

`LifecycleProvider`是`RxLifecycle`功能的抽象接口。`RxLifecycle`只对几个基本组件的实现了`LifecycleProvider`，没覆盖自定义组件的部分。所以我们通过实现这个接口，给自己的组件增加`RxLifecycle`的能力。

```java
// activity 和 fragment lifecycle providers公共的基本接口
public interface LifecycleProvider<E> {

    // @return lifecycle事件序列
    @Nonnull
    @CheckReturnValue
    Observable<E> lifecycle();

    // 绑定一个源，直到一个特定的事件发生（该事件就是手动绑定需结束事件的生命周期）
    @Nonnull
    @CheckReturnValue
    <T> LifecycleTransformer<T> bindUntilEvent(@Nonnull E event);

    // 绑定一个源，直到下一个合理的事件发生
    @Nonnull
    @CheckReturnValue
    <T> LifecycleTransformer<T> bindToLifecycle();
}
```

除此之外，把抽象和实现分离，通过抽象指导实现也是常用的设计模式思维之一。

## 2.3 实现接口

### 2.3.1 lifecycle

一共实现了三个接口，主要看后面两个。

```java
private final BehaviorSubject<ActivityEvent> lifecycleSubject = BehaviorSubject.create();

@Override
@NonNull
@CheckResult
public final Observable<ActivityEvent> lifecycle() {
    return lifecycleSubject.hide();
}
```

`bindUntilEvent`和`bindToLifecycle`实现生命周期绑定

```java
@Nonnull
@CheckReturnValue
<T> LifecycleTransformer<T> bindUntilEvent(@Nonnull E event);

@Nonnull
@CheckReturnValue
<T> LifecycleTransformer<T> bindToLifecycle();
```


### 2.3.2 bindUntilEvent

`bindUntilEvent()`是手动指定观察者取消订阅时的生命周期，不关心所在生命周期的状态。

```java
@Override
@NonNull
@CheckResult
public final <T> LifecycleTransformer<T> bindUntilEvent(@NonNull ActivityEvent event) {
    return RxLifecycle.bindUntilEvent(lifecycleSubject, event);
}
```

例如在`onDestroy`中解除绑定。

```java
.bindUntilEvent(ActivityEvent.DESTROY)
```

### 2.3.3 bindToLifecycle

`bindToLifecycle()`根据绑定时所处生命周期，自动在合理的生命周期中解除订阅。

```java
@Override
@NonNull
@CheckResult
public final <T> LifecycleTransformer<T> bindToLifecycle() {
    return RxLifecycleAndroid.bindActivity(lifecycleSubject);
}
```

## 2.4 生命周期绑定

`Activity`每个生命周期的`lifecycleSubject`发射相应所在生命周期的事件。同理`onStart`到`onDestroy`也完全一样。例如在`Activity`中：

* `onCreate()`绑定 --> `onDestroy()`解除订阅
* `onStart()`绑定 --> `onStop()`解除订阅
* `onResume()`绑定 --> `onPause()`解除订阅

下面仅取`onCreate`作为实例：

```java
@Override
@CallSuper
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    lifecycleSubject.onNext(ActivityEvent.CREATE);
}
```

### 2.4.1 LifecycleTransformer

`LifecycleTransformer`类实现了多个接口，成员方法利用`takeUntil`的特性，当第二个Observable发射了一项数据或者终止时，丢弃原始Observable发射的任何数据。

```java
// Transformer that continues a subscription until a second Observable emits an event.
@ParametersAreNonnullByDefault
public final class LifecycleTransformer<T> implements ObservableTransformer<T, T>,
                                                      FlowableTransformer<T, T>,
                                                      SingleTransformer<T, T>,
                                                      MaybeTransformer<T, T>,
                                                      CompletableTransformer
{
    final Observable<?> observable;

    LifecycleTransformer(Observable<?> observable) {
        checkNotNull(observable, "observable == null");
        this.observable = observable;
    }

    @Override
    public ObservableSource<T> apply(Observable<T> upstream) {
        return upstream.takeUntil(observable);
    }

    @Override
    public Publisher<T> apply(Flowable<T> upstream) {
        return upstream.takeUntil(observable.toFlowable(BackpressureStrategy.LATEST));
    }

    @Override
    public SingleSource<T> apply(Single<T> upstream) {
        return upstream.takeUntil(observable.firstOrError());
    }

    @Override
    public MaybeSource<T> apply(Maybe<T> upstream) {
        return upstream.takeUntil(observable.firstElement());
    }

    @Override
    public CompletableSource apply(Completable upstream) {
        return Completable.ambArray(upstream,
            observable.flatMapCompletable(Functions.CANCEL_COMPLETABLE));
    }
}
```

### 2.4.2 RxLifecycle

`RxLifeCycle`利用`BehaviorSubject`来观察生命周期的变化，通过`takeUtil`，令原`Observable`监视`Subject`发射的数据，当`Subject`发出信号时，`Observable`后续的数据就会无法发射。

```java
public class RxLifecycle {

    private RxLifecycle() {
        throw new AssertionError("No instances");
    }

    // 绑定给定的事件到lifecycle
    @Nonnull
    @CheckReturnValue
    public static <T, R> LifecycleTransformer<T> bindUntilEvent(@Nonnull final Observable<R> lifecycle,
                                                                @Nonnull final R event) {
        checkNotNull(lifecycle, "lifecycle == null");
        checkNotNull(event, "event == null");
        return bind(takeUntilEvent(lifecycle, event));
    }

    private static <R> Observable<R> takeUntilEvent(final Observable<R> lifecycle, final R event) {
        return lifecycle.filter(new Predicate<R>() {
            @Override
            public boolean test(R lifecycleEvent) throws Exception {
                // 生命周期和指定事件匹配返回true
                return lifecycleEvent.equals(event);
            }
        });
    }

    // 绑定给定源到lifecycle
    @Nonnull
    @CheckReturnValue
    public static <T, R> LifecycleTransformer<T> bind(@Nonnull final Observable<R> lifecycle) {
        return new LifecycleTransformer<>(lifecycle);
    }

    // 绑定给定源到lifecycle
    @Nonnull
    @CheckReturnValue
    public static <T, R> LifecycleTransformer<T> bind(@Nonnull Observable<R> lifecycle,
                                                      @Nonnull final Function<R, R> correspondingEvents) {
        checkNotNull(lifecycle, "lifecycle == null");
        checkNotNull(correspondingEvents, "correspondingEvents == null");
        return bind(takeUntilCorrespondingEvent(lifecycle.share(), correspondingEvents));
    }

    private static <R> Observable<Boolean> takeUntilCorrespondingEvent(final Observable<R> lifecycle,
                                                                       final Function<R, R> correspondingEvents) {
        return Observable.combineLatest(
            lifecycle.take(1).map(correspondingEvents),
            lifecycle.skip(1),
            new BiFunction<R, R, Boolean>() {
                @Override
                public Boolean apply(R bindUntilEvent, R lifecycleEvent) throws Exception {
                    return lifecycleEvent.equals(bindUntilEvent);
                }
            })
            .onErrorReturn(Functions.RESUME_FUNCTION)
            .filter(Functions.SHOULD_COMPLETE);
    }
}
```

## 2.5 Subject

上面还出现了`BehaviorSubject`，只看看他继承的父类`Subject`。这个类直接继承了`Observable<T>`，表明是被观察者；又实现`Observer<T>`，所以也能作为一名观察者。作为两个角色都兼顾的类，对上级的被观察者身份是观察者，对下级观察者身份是被观察者。

[官方文档](http://reactivex.io/RxJava/2.x/javadoc/)

```java
public abstract class Subject<T> extends Observable<T> implements Observer<T> {

    public abstract boolean hasObservers();

    public abstract boolean hasThrowable();

    public abstract boolean hasComplete();

    @Nullable
    public abstract Throwable getThrowable();

    @NonNull
    public final Subject<T> toSerialized() {
        if (this instanceof SerializedSubject) {
            return this;
        }
        return new SerializedSubject<T>(this);
    }
}
```

## 2.6 Event分类

`ActivityEvent`的生命周期枚举值

```java
public enum ActivityEvent {
    CREATE,
    START,
    RESUME,
    PAUSE,
    STOP,
    DESTROY
}
```

`FragmentEvent`的生命周期枚举值，里面没有`onActivityCreated`，所以建议用`CREATE_VIEW`代替。

```java
public enum FragmentEvent {
    ATTACH,
    CREATE,
    CREATE_VIEW,
    START,
    RESUME,
    PAUSE,
    STOP,
    DESTROY_VIEW,
    DESTROY,
    DETACH
}
```


# 三、参考链接

* [RxJava文档中文版](https://www.gitbook.com/book/mcxiaoke/rxdocs/details)

* [Why not RxLifecycle](http://blog.danlew.net/2017/08/02/why-not-rxlifecycle/)

