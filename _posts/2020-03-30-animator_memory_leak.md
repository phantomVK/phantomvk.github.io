---
layout:     post
title:      "Android动画内存泄漏原理"
date:       2020-03-30
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

### 一、示例

**ValueAnimator** 持续输出从0到1000的整形值，然后反向输出1000到0数值，如此循环往复。该整形值转换为字符串设置到 __TextView__。

```kotlin
class LeakActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_leak)

        val a = ValueAnimator.ofInt(0, 1000)
        a.duration = 1000
        a.repeatMode = ValueAnimator.REVERSE
        a.repeatCount = ValueAnimator.INFINITE
        // 传入AnimatorUpdateListener实现，此实现隐式持有Activity引用
        a.addUpdateListener { l -> textView.text = l.animatedValue.toString() }
        a.start()
    }
}
```

### 二、源码

#### 2.1 类签名

首先关注 **ValueAnimator** 的父类，这里尤其关注类继承了接口 __AnimationHandler.AnimationFrameCallback__。

```java
public class ValueAnimator extends Animator implements AnimationHandler.AnimationFrameCallback
```

#### 2.2 start()方法

然后看 __start()__ 方法做了什么

```java
@Override
public void start() {
    start(false);
}

// 初始化很多状态相关的布尔值，这些值都不是我们关心的，直接看方法第36行
private void start(boolean playBackwards) {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    mReversing = playBackwards;
    mSelfPulse = !mSuppressSelfPulseRequested;
    // Special case: reversing from seek-to-0 should act as if not seeked at all.
    if (playBackwards && mSeekFraction != -1 && mSeekFraction != 0) {
        if (mRepeatCount == INFINITE) {
            // Calculate the fraction of the current iteration.
            float fraction = (float) (mSeekFraction - Math.floor(mSeekFraction));
            mSeekFraction = 1 - fraction;
        } else {
            mSeekFraction = 1 + mRepeatCount - mSeekFraction;
        }
    }
    mStarted = true;
    mPaused = false;
    mRunning = false;
    mAnimationEndRequested = false;
    // Resets mLastFrameTime when start() is called, so that if the animation was running,
    // calling start() would put the animation in the
    // started-but-not-yet-reached-the-first-frame phase.
    mLastFrameTime = -1;
    mFirstFrameTime = -1;
    mStartTime = -1;
    
    // 注意这里调用addAnimationCallback()方法
    // 此方法调用完，下面几行代码是实际开始动画的逻辑
    addAnimationCallback(0);

    if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
        // If there's no start delay, init the animation and notify start listeners right away
        // to be consistent with the previous behavior. Otherwise, postpone this until the first
        // frame after the start delay.
        startAnimation();
        if (mSeekFraction == -1) {
            // No seek, start at play time 0. Note that the reason we are not using fraction 0
            // is because for animations with 0 duration, we want to be consistent with pre-N
            // behavior: skip to the final value immediately.
            setCurrentPlayTime(0);
        } else {
            setCurrentFraction(mSeekFraction);
        }
    }
}
```

上面提到的 __addAnimationCallback()__ 方法看一下。

方法实参 **this** 就是 **ValueAnimator** 实例，作为 __AnimationHandler.AnimationFrameCallback__ 接口的实现类，添加到 __AnimationHandler.addAnimationFrameCallback()__ 方法内。

```java
private void addAnimationCallback(long delay) {
    if (!mSelfPulse) {
        return;
    }
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```

由于在下文这个 __AnimationHandler__ 会再次出现。

```java
public class AnimationHandler {
    // Internal per-thread collections used to avoid set collisions as animations start and end
    // while being processed.
    private final ArrayMap<AnimationFrameCallback, Long> mDelayedCallbackStartTime =
            new ArrayMap<>();
    private final ArrayList<AnimationFrameCallback> mAnimationCallbacks =
            new ArrayList<>();
    private final ArrayList<AnimationFrameCallback> mCommitCallbacks =
            new ArrayList<>();

    // callback放入到列表中
    public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        if (mAnimationCallbacks.size() == 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
        if (!mAnimationCallbacks.contains(callback)) {
            mAnimationCallbacks.add(callback);
        }

        if (delay > 0) {
            mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
        }
    }
}
```

#### 2.2 cancel()方法

文处实例代码没有调用动画的取消方法，所以直接看 __ValueAnimator.cancel()__ 是如何释放资源的

```java
@Override
public void cancel() {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }

    // 如果动画没启动，不需要取消而直接返回
    if (mAnimationEndRequested) {
        return;
    }

    // 只停止正在执行的动画监听器
    if ((mStarted || mRunning) && mListeners != null) {
        if (!mRunning) {
            // If it's not yet running, then start listeners weren't called. Call them now.
            notifyStartListeners();
        }
        
        // 回调所有注册的监听器
        ArrayList<AnimatorListener> tmpListeners =
                (ArrayList<AnimatorListener>) mListeners.clone();
        for (AnimatorListener listener : tmpListeners) {
            listener.onAnimationCancel(this);
        }
    }

    // 这里终止动画
    endAnimation();
}
```

__endAnimation()__ 方法由 __ValueAnimator__ 内部调用，结束动画时会从动画列表移除该动画，即 __removeAnimationCallback()__

```java
private void endAnimation() {
    if (mAnimationEndRequested) {
        return;
    }
  
    // 关注移除动画回调
    removeAnimationCallback();

    mAnimationEndRequested = true;
    mPaused = false;
    boolean notify = (mStarted || mRunning) && mListeners != null;
    if (notify && !mRunning) {
        // If it's not yet running, then start listeners weren't called. Call them now.
        notifyStartListeners();
    }
    mRunning = false;
    mStarted = false;
    mStartListenersCalled = false;
    mLastFrameTime = -1;
    mFirstFrameTime = -1;
    mStartTime = -1;
    if (notify && mListeners != null) {
        ArrayList<AnimatorListener> tmpListeners =
                (ArrayList<AnimatorListener>) mListeners.clone();
        int numListeners = tmpListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            // 这里面会移除示例设置的AnimatorUpdateListener
            // 就是这个AnimatorUpdateListener持有的Activity引用
            // 只有本方法不调用，这个监听器就不会移除
            tmpListeners.get(i).onAnimationEnd(this, mReversing);
        }
    }
    // mReversing needs to be reset *after* notifying the listeners for the end callbacks.
    mReversing = false;
    if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
        Trace.asyncTraceEnd(Trace.TRACE_TAG_VIEW, getNameForTrace(),
                System.identityHashCode(this));
    }
}
```

__endAnimation()__ 方法调用以下方法，前文 __start()__ 方法分析出现的 __AnimationHandler__ 再次出现。

```java
private void removeAnimationCallback() {
    if (!mSelfPulse) {
        return;
    }
    getAnimationHandler().removeCallback(this);
}
```

继续跳到这里获取单例

```java
public AnimationHandler getAnimationHandler() {
    return AnimationHandler.getInstance();
}
```

可见 __AnimationHandler__ 存放在 __ThreadLocal__ 里面，就是主线程的 __ThreadLocal__ 区域内。而 __AnimationFrameCallback__ 保存在 __AnimationHandler__ 中。

```java
public class AnimationHandler {

    private final ArrayMap<AnimationFrameCallback, Long> mDelayedCallbackStartTime = new ArrayMap<>();
    private final ArrayList<AnimationFrameCallback> mAnimationCallbacks = new ArrayList<>();
    private final ArrayList<AnimationFrameCallback> mCommitCallbacks = new ArrayList<>();

    // 所有注册的AnimationHandler按照线程分类，存放在线程的ThreadLocal区域内
    // 因为动画在主线程创建和播放，所以保存在主线程ThreadLocal
    // 而AnimationHandler实例包含的数据就是上面三个列表
    public final static ThreadLocal<AnimationHandler> sAnimatorHandler = new ThreadLocal<>();
    private boolean mListDirty = false;
    
    // AnimationHandler实例被获取之后
    // 会被调用后面的removeCallback(AnimationFrameCallback)方法
    public static AnimationHandler getInstance() {
        if (sAnimatorHandler.get() == null) {
            sAnimatorHandler.set(new AnimationHandler());
        }
        return sAnimatorHandler.get();
    }
    
    // 从三个列表移除回调
    public void removeCallback(AnimationFrameCallback callback) {
        // 如果页面退出但动画没有取消
        // 则callback隐式持有的Activity引用并一直保存在列表中
        mCommitCallbacks.remove(callback);
        mDelayedCallbackStartTime.remove(callback);
        int id = mAnimationCallbacks.indexOf(callback);
        if (id >= 0) {
            mAnimationCallbacks.set(id, null);
            mListDirty = true;
        }
    }
    
    .....
}
```

### 三、总结

总结内存泄漏路径：

- __ThreadLocal__ 在主线程持有 __AnimationHandler__ 对象；
- __AnimationHandler__ 持有多个 __AnimationFrameCallback__ 列表，列表负责不同功能；
- __AnimationFrameCallback__ 接口由 __ValueAnimator__ 类实现，就是我们示例代码创建的 __ValueAnimator__；
- 创建 __ValueAnimator__ 实例时，我们设置了监听器修改 __textView__ 的值，该监听器隐式持有 __Activity__ 实例；

若不结束或清除 __ThreadLocal__ 中 __AnimationHandler__ 的 __AnimationFrameCallback__ 回调，__AnimationFrameCallback__ 就是 __ValueAnimator__，而 __Activity__ 引用就被 __ValueAnimator__ 的监听器永久间接持有造成内存泄漏；

