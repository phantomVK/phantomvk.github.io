---
layout:     post
title:      "Android源码系列(9) -- Activity事件分发"
date:       2018-03-25
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 前言

之前介绍了 [View事件分发](https://phantomvk.github.io/2016/10/18/Android_View/) 和 [ViewGroup事件分发](https://phantomvk.github.io/2016/11/07/android-viewgroup/) ，了解点击事件如何在ViewGroup和View内部流动。如果把两者联系起来，容易知道ViewGroup把事件分发给View，当View不拦截事件时又把事件返回给ViewGroup。

本文研究Activity事件分发，探究事件如何从Activity分发到ViewGroup，ViewGroup不拦截事件Activity又如何处理。文章最终解释Activity、ViewGroup、Group三者事件分发行为如何形成闭环。

![Activity ViewGroup View](/img/android/event/Activity_ViewGroup_View.png)

上图只是事件分发最基本的示意，实际分发细节要复杂得多。Activity事件分发源码基于Android 27。另外两篇文章已经说明ViewGroup和View，Activity再遇到相关知识不深究，请自行翻阅前文。

## 一、Activity事件分发

#### 1.1 dispatchTouchEvent() 

点击事件最先被分发到Activity上，首个处理事件方法是dispatchTouchEvent()，重写方法可在所有事件分发到`window`前就进行拦截。方法参数`MotionEvent ev`是点击的事件类。

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ACTION_DOWN事件触发onUserInteraction()
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }

    // 事件交给window处理
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }

    // window没有消费该事件，交给Activity.onTouchEvent()处理
    return onTouchEvent(ev);
}
```

#### 1.2 onUserInteraction()

无论是按钮、触摸还是轨迹球事件都会分发给Activity。可以重写此方法，在activity运行过程中捕获用户与设备的交互事件。方法相当于一个回调，和`onUserLeaveHint()`一样，是为了帮助activity智能地管理状态栏的通知，尤其是在合适时间点取消一个与之相关的通知。

所有对`onUserLeaveHint()`的调用会同时伴随着对`onUserInteraction()`的调用，确保activity在一些关于用户操作，如向下拉并点击了通知时得到告知。方法只在`ACTION_DOWN`才会触发。

```java
public void onUserInteraction() {
}
```

#### 1.3 onUserLeaveHint()

和`onUserInteraction()`有关的方法。作为activity生命周期的一部分，用户把activity推到后台的时候调用方法。例如用户点击Home键`onUserLeaveHint()`就会被调用。但显示来电导致activity被中断并推到后台时`onUserLeaveHint()`不会调用。在`onPause()`生命周期调用前先触发此方法。

```java
protected void onUserLeaveHint() {
}
```

#### 1.4 performUserLeaving()

`onUserInteraction()`和`onUserLeaveHint()`都在用户离开时被回调，通过此方法串联起来：

```java
final void performUserLeaving() {
    onUserInteraction();
    onUserLeaveHint();
}
```

#### 1.5 小结

![Activity ViewGroup View](/img/android/event/activity_dispatchTouchEvent.png)

## 二、window.superDispatchTouchEvent()

#### 2.1 activity.getWindow()

getWindow()返回Window抽象类，`superDispatchTouchEvent()`是window的抽象方法。由自定义的windows调用，透过视图层级传递屏幕点击事件，例如Dialog。

```java
public abstract boolean superDispatchTouchEvent(MotionEvent event);
```

#### 2.2 PhoneWindow.superDispatchTouchEvent()
window的实现类是PhoneWindow，实际调用`PhoneWindow.superDispatchTouchEvent()`，进而调用`mDecor.superDispatchTouchEvent(event)`。

DecorView是一个保存在PhoneWindow的成员变量。有很多文章提到DecorView是PhoneWindow内部类。但从`Android27`看来，DecorView是独立的类而不是一个内部类。

```java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;

// Constructor for main window of an activity.
public PhoneWindow(Context context, Window preservedWindow,
        ActivityConfigCallback activityConfigCallback) {
    this(context);
    // Only main activity windows use decor context, all the other windows depend on whatever
    // context that was given to them.
    mUseDecorContext = true;
    if (preservedWindow != null) {
        mDecor = (DecorView) preservedWindow.getDecorView();
        mElevation = preservedWindow.getElevation();
        mLoadElevation = false;
        mForceDecorInstall = true;
        // If we're preserving window, carry over the app token from the preserved
        // window, as we'll be skipping the addView in handleResumeActivity(), and
        // the token will not be updated as for a new window.
        getAttributes().token = preservedWindow.getAttributes().token;
    }
    // Even though the device doesn't support picture-in-picture mode,
    // an user can force using it through developer options.
    boolean forceResizable = Settings.Global.getInt(context.getContentResolver(),
            DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
    mSupportsPictureInPicture = forceResizable || context.getPackageManager().hasSystemFeature(
            PackageManager.FEATURE_PICTURE_IN_PICTURE);
    mActivityConfigCallback = activityConfigCallback;
}

@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

在Activity类中，在onAttach()方法创建PhoneWindow实例

```java
final void attach(Context context, ....){
    ....
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ....
}
```

#### 2.3 DecorView.superDispatchTouchEvent()

调用super.dispatchTouchEvent()

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
#### 2.4 FrameLayout.dispatchTouchEvent()
由DecorView父类FrameLayout可知：

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks
```
调用super.dispatchTouchEvent(event)即是FrameLayout.dispatchTouchEvent(event)

#### 2.5 小结

![superDispatchTouchEvent](/img/android/event/superDispatchTouchEvent.png)

### 三、 Activity.onTouchEvent() 

#### 3.1 onTouchEvent()

ViewGroup和View都没有消费事件，该事件最终回到activity，交给activity.onTouchEvent()。

```java
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    // activity也不消费事件，事件的分发流程终结
    return false;
}
```
#### 3.2 Windos.shouldCloseOnTouch()
事件点击在DecorVIew外，且点击事件没有被其他组件消费时，支持关闭activity
```java
public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
    final boolean isOutside =
            event.getAction() == MotionEvent.ACTION_DOWN && isOutOfBounds(context, event)
            || event.getAction() == MotionEvent.ACTION_OUTSIDE;
    if (mCloseOnTouchOutside && peekDecorView() != null && isOutside) {
        return true;
    }
    return false;
}

private boolean mCloseOnTouchOutside = false;
private boolean mSetCloseOnTouchOutside = false;

public void setCloseOnTouchOutside(boolean close) {
    mCloseOnTouchOutside = close;
    mSetCloseOnTouchOutside = true;
}

public void setCloseOnTouchOutsideIfNotSet(boolean close) {
    if (!mSetCloseOnTouchOutside) {
        mCloseOnTouchOutside = close;
        mSetCloseOnTouchOutside = true;
    }
}
```
检查点击事件是否落在decorView外
```java
private boolean isOutOfBounds(Context context, MotionEvent event) {
    final int x = (int) event.getX();
    final int y = (int) event.getY();
    final int slop = ViewConfiguration.get(context).getScaledWindowTouchSlop();
    final View decorView = getDecorView();
    return (x < -slop) || (y < -slop)
            || (x > (decorView.getWidth()+slop))
            || (y > (decorView.getHeight()+slop));
}
```
#### 3.3 window.peekDecorView()
获取已经当前已经创建的顶层decorView，否则返回null。已知Window是抽象类，所以看实现类PhoneWindow。

```java
public abstract View peekDecorView();
```

#### 3.4 PhoneWindow.peekDecorView()
仅仅是返回持有的DecorView
```java
private DecorView mDecor;

@Override
public final View peekDecorView() {
    return mDecor;
}
```

## 四、总结

Activity仅有`dispatchTouchEvent()`和`onTouchEvent()`两个主要方法，和View的方法一样没有`ViewGroup.onInterceptTouchEvent()`。

因为Activity本身默认不处理任何点击事件，只在ViewGroup和View都不处理事件时才尝试消费事件。最终也可能返回false，表示activity也不消费事件。