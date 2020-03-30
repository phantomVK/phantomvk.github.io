---
layout:     post
title:      "动画内存泄漏原理"
date:       2020-03-30
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

以下示例动画持续修改 __TextView__ 的文本数值

```kotlin
class LeakActivity : AppCompatActivity() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_leak)

    val a = ValueAnimator.ofInt(0, 1000)
    a.duration = 1000
    a.repeatMode = ValueAnimator.REVERSE
    a.repeatCount = ValueAnimator.INFINITE
    a.addUpdateListener { l -> textView.text = l.animatedValue.toString() }
    a.start()
  }
}
```

直接看 __ValueAnimator.cancel()__ 如何释放资源

```java
@Override
public void cancel() {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }

    // 如果动画没启动，不需要取消
    if (mAnimationEndRequested) {
        return;
    }

    // 只停止正在执行的动画监听器
    if ((mStarted || mRunning) && mListeners != null) {
        if (!mRunning) {
            // If it's not yet running, then start listeners weren't called. Call them now.
            notifyStartListeners();
        }
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

__endAnimation()__ 注释提到，此方法由 __ValueAnimator__ 内部调用，结束动画时要从动画列表移除该动画

```java
private void endAnimation() {
    if (mAnimationEndRequested) {
        return;
    }
  
    // 移除动画回调
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

上述方法调用以下方法

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
    public final static ThreadLocal<AnimationHandler> sAnimatorHandler = new ThreadLocal<>();

    public static AnimationHandler getInstance() {
        if (sAnimatorHandler.get() == null) {
            sAnimatorHandler.set(new AnimationHandler());
        }
        return sAnimatorHandler.get();
    }

    public void removeCallback(AnimationFrameCallback callback) {
        mCommitCallbacks.remove(callback);
        mDelayedCallbackStartTime.remove(callback);
        int id = mAnimationCallbacks.indexOf(callback);
        if (id >= 0) {
            mAnimationCallbacks.set(id, null);
            mListDirty = true;
        }
    }
}
```

总结内存泄漏路径：

- __ThreadLocal__ 在主线程持有 __AnimationHandler__；
- __AnimationHandler__ 持有多个 __AnimationFrameCallback__ 列表；
- __AnimationFrameCallback__ 接口由 __ValueAnimator__ 类实现；
- 创建的 __ValueAnimator__ 实例又隐式持有 __Activity__ 实例；
- 所以不结束或清除 __ThreadLocal__，__Activity__ 引用被 __AnimationHandler__ 间接持有；