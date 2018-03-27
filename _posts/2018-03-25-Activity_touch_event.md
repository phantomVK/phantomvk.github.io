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

之前介绍了 [View事件分发](https://phantomvk.github.io/2016/10/18/Android_View/) 和 [ViewGroup事件分发](https://phantomvk.github.io/2016/11/07/android-viewgroup/) ，了解点击事件如何在ViewGroup和View内部流动。如果把两者联系起来，容易知道ViewGroup把事件分发给View，当View不拦截事件时把事件又返回给ViewGroup。

本文研究Activity事件分发，探究事件如何从Activity分发到ViewGroup，ViewGroup不拦截事件时Activity会如何处理。本篇文章最终解释Activity、ViewGroup、Group三者事件分发行为如何形成闭环。

![Activity ViewGroup View](/img/android/Activity_ViewGroup_View.png)

上图只是事件分发最基本的示意，实际分发细节要复杂得多。Activity事件分发源码基于Android 27。另外两篇文章已经说明ViewGroup和View，Activity再遇到相关知识不深究，请自行翻阅前文。

## 一、Activity事件分发

#### 1.1 dispatchTouchEvent() 

发生的点击事件最先被分发到Activity上，首个处理事件的方法是dispatchTouchEvent()，重写方法可实现所有事件分发到window前就进行拦截的目的。如果不正确重写这个方法，会影响后续所有事件的分发处理逻辑。方法参数MotionEvent是点击的事件类。

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ACTION_DOWN事件触发onUserInteraction()
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    // 事件交给Activity的Window处理
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    // Window没有消费该事件，交由Activity本身的onTouchEvent()消费
    return onTouchEvent(ev);
}
```

#### 1.2 onUserInteraction()

无论key、touch、还是trackball事件都会触发此方法。可以重写此方法令activity在运行过程中捕获用户与设备交互的事件。此方法相当于一个回调，和onUserLeaveHint()一样，是为了帮助activity智能管理状态栏的系统通知，尤其是在合适时间点取消一个通知。

所有对onUserLeaveHint()的调用会同时伴随着对onUserInteraction()的调用，确保你的activity在一些关于的用户活动，如向下拉并点击了通知时得到告知。方法只在ACTION_DOWN才会触发。

```java
public void onUserInteraction() {
}
```

#### 1.3 onUserLeaveHint()

和onUserInteraction()有关的方法。作为activity生命周期的一部分，用户把activity推到后台的时候调用方法。例如用户点击Home键onUserLeaveHint()就会被调用。但显示来电导致activity被中断并推到后台时，onUserLeaveHint()不会调用。在activity.onPause()生命周期回调调用前先触发此方法。

```java
protected void onUserLeaveHint() {
}
```

#### 1.4 performUserLeaving()

onUserInteraction()和onUserLeaveHint()都在用户离开时被回调，通过此方法串联起来：

```java
final void performUserLeaving() {
    onUserInteraction();
    onUserLeaveHint();
}
```

#### 1.5 小结

![Activity ViewGroup View](/img/android/activity_dispatchTouchEvent.png)

## 二、window.superDispatchTouchEvent()

#### 2.1 activity.getWindow()

activity.getWindow()返回Window抽象类。window.superDispatchTouchEvent()同样是抽象方法，由自定义的windows调用，透过视图层级传递屏幕点击事件，例如Dialog。

```java
public abstract boolean superDispatchTouchEvent(MotionEvent event);
```

#### 2.2 PhoneWindows.superDispatchTouchEvent()
window的实现类是PhoneWindows，即调用PhoneWindows.superDispatchTouchEvent()，也就是调用mDecor.superDispatchTouchEvent(event)。而DecorView是保存在PhoneWindows的成员变量。有很多文章提到DecorView是PhoneWindows内部类，但从Android27看来，DecorView是独立的类而不是一个内部类。

```java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;

@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
#### 2.3 DecorView.superDispatchTouchEvent()

调用super.dispatchTouchEvent()。

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
#### 2.4 FrameLayout.dispatchTouchEvent()
根据DecorView的父类FrameLayout可知，super.dispatchTouchEvent(event)等价FrameLayout.dispatchTouchEvent()。

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks
```
#### 2.5 小结
![superDispatchTouchEvent](/img/android/superDispatchTouchEvent.png)

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
获取已经当前已经创建的顶层decorView，否则返回null。已知Window是抽象类，所以看实现类PhoneWindow
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

activity仅有dispatchTouchEvent() 和 onTouchEvent()两个主要方法，和View的方法一致，没有ViewGroup.onInterceptTouchEvent()，因为activity本身默认不处理任何点击事件，只在ViewGroup和View都不处理事件时才尝试消费事件。最终也可能返回false，表示activity也不消费事件。