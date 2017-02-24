---
layout:     post
title:      "Android源码系列(1) -- View"
date:       2016-10-18
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---



# 前言

最近看View的事件分发源码，并做了一些笔记。笔记以注释的形式插入到代码中，请仔细阅读文章中给出的源码，或有错漏之处，欢迎指正。-- 源码来自Android 6.0

# 一、代码构建

### 1.1 自定义Button

为了能看见事件调用什么方法，先继承`Button`类并重载`dispatchTouchEvent()`和`onTouchEvent()`。而所有发送给View的事件，首先是由`dispatchTouchEvent()`接收。

```java
public class MyButton extends Button {

    private static final String TAG = "MyButton";

    public MyButton(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        int action = event.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                Log.e(TAG, "dispatchTouchEvent ACTION_DOWN");
                break;
            case MotionEvent.ACTION_MOVE:
                Log.e(TAG, "dispatchTouchEvent ACTION_MOVE");
                break;
            case MotionEvent.ACTION_UP:
                Log.e(TAG, "dispatchTouchEvent ACTION_UP");
                break;
            default:
                break;
        }
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int action = event.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                Log.e(TAG, "onTouchEvent ACTION_DOWN");
                break;
            case MotionEvent.ACTION_MOVE:
                Log.e(TAG, "onTouchEvent ACTION_MOVE");
                break;
            case MotionEvent.ACTION_UP:
                Log.e(TAG, "onTouchEvent ACTION_UP");
                break;
            default:
                break;
        } 
        return super.onTouchEvent(event);
    }
}
```

### 1.2 xml布局

在`main_activity.xml`中使用自定义的`Button`

```xml
<RelativeLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.corevk.demoproject.MyButton
        android:id="@+id/MyButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>    
</RelativeLayout>
```

### 1.3 MainActivity

绑定按钮并给按钮设置一个监听器`OnTouchListener`。后面我会说明这个监听器的用途。

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    private Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mButton = (Button) findViewById(R.id.MyButton);
        mButton.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                int action = event.getAction();
                switch (action) {
                    case MotionEvent.ACTION_DOWN:
                        Log.e(TAG, "onTouch ACTION_DOWN");
                        break;
                    case MotionEvent.ACTION_MOVE:
                        Log.e(TAG, "onTouch ACTION_MOVE");
                        break;
                    case MotionEvent.ACTION_UP:
                        Log.e(TAG, "onTouch ACTION_UP");
                        break;
                    default:
                        break;
                }
                return false; // 返回值会影响事件的分发行为
            }
        });
    }
}
```

### 1.4 梳理

一共设置三个东西：`dispatchTouchEvent()`、`onTouchEvent()`和`OnTouchListener()`。这三个被重写的方法都对`ACTION_DOWN`、`ACTION_MOVE`、`ACTION_UP`的动作显示信息。

# 二、运行结果

### 2.1 OnTouchListener false

`View.OnTouchListener`中返回`false`，点击按钮马上放开。如果手指一直在屏幕上滑动，Log的`ACTION_DOWN`和`ACTION_UP`之间会报告`ACTION_MOVE`的信息。

结果按照`dispatchTouchEvent` -> `onTouch` -> `onTouchEvent`出现

```
10-13 23:53:29.382 17840-17840/? E/MyButton: dispatchTouchEvent ACTION_DOWN
10-13 23:53:29.382 17840-17840/? E/MainActivity: onTouch ACTION_DOWN
10-13 23:53:29.382 17840-17840/? E/MyButton: onTouchEvent ACTION_DOWN

10-13 23:53:29.414 17840-17840/? E/MyButton: dispatchTouchEvent ACTION_UP
10-13 23:53:29.414 17840-17840/? E/MainActivity: onTouch ACTION_UP
10-13 23:53:29.414 17840-17840/? E/MyButton: onTouchEvent ACTION_UP
```

### 2.2 OnTouchListener true

在`View.OnTouchListener`的返回值中返回`true`，点击按钮马上放开:`dispatchTouchEvent（）` -> `onTouch`

```
10-13 23:55:32.523 18106-18106/? E/MyButton: dispatchTouchEvent ACTION_DOWN
10-13 23:55:32.523 18106-18106/? E/MainActivity: onTouch ACTION_DOWN

10-13 23:55:32.554 18106-18106/? E/MyButton: dispatchTouchEvent ACTION_UP
10-13 23:55:32.554 18106-18106/? E/MainActivity: onTouch ACTION_UP
```

当`View.OnTouchListener`的返回值中返回`true`，`onTouchEvent()`不会触发，说明事件没有分发到`onTouchEvent()`。

# 三、源码分析

### 3.1 dispatchTouchEvent

先看`dispatchTouchEvent`源码

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (event.isTargetAccessibilityFocus()) {
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false; // 默认为false

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        stopNestedScroll(); // 停止嵌套滚动
    }

    // 安全机制过滤触摸事件，被遮挡要过滤该事件
    if (onFilterTouchEventForSecurity(event)) {
        ListenerInfo li = mListenerInfo; // 这里获取mListenerInfo
        
        // 以下所有条件成立执行这个语句块并返回True:
        //    1. mListenerInfo不为空，且设置OnTouchListener监听器
        //    2. view为Enable，表明控件可用
        //    3. mOnTouchListener.onTouch(this, event)尝试消费事件
        if (li != null 
        		&& li.mOnTouchListener != null
        		&& (mViewFlags & ENABLED_MASK) == ENABLED
        		&& li.mOnTouchListener.onTouch(this, event)){ 
            // 若Button.OnClickListener返回false，则result为false
            result = true;
        }
        
        // result为false，交给onTouchEven处理
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

那么`li.mOnTouchListener`在哪里设定？上面的代码有一段可知：

```java
ListenerInfo li = mListenerInfo;

if (li != null
    && li.mOnTouchListener != null
    && (mViewFlags & ENABLED_MASK) == ENABLED
    && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
   }  
```

`li.mOnTouchListener`是依赖`mButton.setOnTouchListener`的。如果不调用`mButton.setOnTouchListener`，`li.mOnTouchListener`为空。

在`MainActivity.onCreate`中给`mButton.setOnTouchListener`创建一个`OnTouchListener()`实例的同时，这个实例会保存在`getListenerInfo().mOnTouchListener`。

```java
public void setOnTouchListener(OnTouchListener l) {
    getListenerInfo().mOnTouchListener = l;
}
```

而`getListenInfo`里面判断`mListenerInfo`是否为空，非空直接返回，否则创建新的`ListenerInfo`。

```java
ListenerInfo getListenerInfo() {
    if (mListenerInfo != null) {
        return mListenerInfo;
    }
    mListenerInfo = new ListenerInfo();
    return mListenerInfo;
}
```

### 3.2 onTouchEvent

如果设置了`View.OnTouchListener()`，其返回值决定了事件是否继续分发给`onTouchEvent`实例。假如`OnTouchListener()`返回`false`，`onTouchEvent`可以接收事件，那后面又会做什么？

```java
public boolean onTouchEvent(MotionEvent event) {
    // 获取动作点击屏幕的位置
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
    
    // View为Disable进入，否则跳过
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP
                && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }   
        // 可点击或长按的View可以消费事件，但没有实际动作
       	return (((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
    }  
 
    // 代理可以处理事件就返回true
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    
    // 如果view是可点击的，处理不同的点击操作后返回true
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) { 
            // 分支代码剖析在下一节
            case MotionEvent.ACTION_UP:
                .....
                break;
                
            case MotionEvent.ACTION_DOWN:
                ....
                break;
                
            case MotionEvent.ACTION_CANCEL:
                ....
                break;

            case MotionEvent.ACTION_MOVE:
                ....
                break;
        } 
        return true; // 处理完后返回true
    }
    return false; // 事件继续下发
}
```

# 四、MotionEvent详解

Android触摸消息中有三种监测类别：

1. **Prepressed**：轻触(tap)屏幕，时间小于TAP_TIMEOUT
2. **Pressed**：点击(press)屏幕，时间介于TAP_TIMEOUT和LONG_PRESS_TIMEOUT之间
3. **Long pressed**：长按(long press)屏幕，时间大于DEFAULT_LONG_PRESS_TIMEOUT或LONG_PRESS_TIMEOUT


![img](/img/android/pressed.png)

不同版本的Android API时间值可能不同，Android 6.0(23) `TAP_TIMEOUT`是100ms，部分旧版本是115ms，这里按照最新值解说。

```java
/**
 * Defines the duration in milliseconds of the pressed state in child
 * components.
 */
private static final int PRESSED_STATE_DURATION = 64;

/**
 * Defines the duration in milliseconds we will wait to see if a touch event
 * is a tap or a scroll. If the user does not move within this interval, it is
 * considered to be a tap.
 */
private static final int TAP_TIMEOUT = 100;


/**
 * Defines the default duration in milliseconds before a press turns into
 * a long press
 */
private static final int DEFAULT_LONG_PRESS_TIMEOUT = 500;
```

### 4.1 MotionEvent.ACTION_DOWN
```java
mHasPerformedLongPress = false; // 长按事件默认为false;

if (performButtonActionOnTouchDown(event)) {
    break;
}

//检查是否在滚动容器内
boolean isInScrollingContainer = isInScrollingContainer();

// 在可滚动的视图中会增加点击的检查
if (isInScrollingContainer) {
    mPrivateFlags |= PFLAG_PREPRESSED; // 增加PREPRESSED标志

    // 创建CheckForTap()实例 mPendingCheckForTap
    if (mPendingCheckForTap == null) {
        mPendingCheckForTap = new CheckForTap();
    }
    mPendingCheckForTap.x = event.getX();
    mPendingCheckForTap.y = event.getY();
    
    // 100ms后执行检查能否达到PRESSED状态
    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
} else {
    setPressed(true, x, y); // 不在滚动容器就设置PRESSED状态
    checkForLongClick(0);   // 开始检测长按动作
}
```

在上面的代码中，调用时先经过100ms的延时，再使用`CheckForTap`

```java
private final class CheckForTap implements Runnable {
    public float x; // mPendingCheckForTap.x
    public float y; // mPendingCheckForTap.y

    @Override
    public void run() {
        // 取消PFLAG_PREPRESSED标志
        mPrivateFlags &= ~PFLAG_PREPRESSED;
        
        // 检查点击位置是否移动，没有移动就增加PRESSED标志
        setPressed(true, x, y);
        
        // 接着开始长按动检查
        checkForLongClick(ViewConfiguration.getTapTimeout());
    }
}
```

**checkForLongClick**

```java
private void checkForLongClick(int delayOffset) {
    // 仅在view支持长按时执行，否则退出方法
    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
        mHasPerformedLongPress = false;

        if (mPendingCheckForLongPress == null) {
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        mPendingCheckForLongPress.rememberWindowAttachCount(); // 可能是多点触控数值
        
        // 调用方法前已经延迟100ms，需要减去这段时间
        postDelayed(mPendingCheckForLongPress,
            ViewConfiguration.getLongPressTimeout() - delayOffset);
    }
}
```

**CheckForLongPress**

```java
private final class CheckForLongPress implements Runnable {
    private int mOriginalWindowAttachCount;

    @Override
    public void run() {
        // 需要检查是否已经到达PRESSED状态
        if (isPressed() && (mParent != null)
                && mOriginalWindowAttachCount == mWindowAttachCount) {   
                if (performLongClick()) {
                    mHasPerformedLongPress = true;
            }
        }
    }
    
    // 疑似记录长按过程多点触控是否变化
    public void rememberWindowAttachCount() {
        mOriginalWindowAttachCount = mWindowAttachCount;
    }
}
```

在run里面调用的performLongClick。如果设置的长按回调会在下面调用。返回值与handled相同，直接控制**mHasPerformedLongPress**。


```java
public boolean performLongClick() {
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

    boolean handled = false;
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnLongClickListener != null) {
        handled = li.mOnLongClickListener.onLongClick(View.this);
    }
    if (!handled) {
        handled = showContextMenu();
    }
    if (handled) {
        performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
    }
    return handled;
}
```

### 4.2 MotionEvent.ACTION_MOVE

```java
drawableHotspotChanged(x, y); // 当前位置x,y

// 移动到按钮的范围外就会执行
if (!pointInView(x, y, mTouchSlop)) {
    // 在按钮外，移除PREPRESSED状态和对应回调
    removeTapCallback();
    
    // 如果是PRESSED，就移除长按检测，并移除PRESSED状态
    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
        removeLongPressCallback();

        setPressed(false);
    }
}
```

```java
public void setPressed(boolean pressed) {
	// 如果状态不是PRESSED，更新标志位并刷新背景
	final boolean needsRefresh =
	    pressed != ((mPrivateFlags & PFLAG_PRESSED) == PFLAG_PRESSED);

    if (pressed) {
        mPrivateFlags |= PFLAG_PRESSED;
    } else {
        mPrivateFlags &= ~PFLAG_PRESSED;
    }

    if (needsRefresh) {
        refreshDrawableState();
    }
    dispatchSetPressed(pressed);
}
```

当触发时间不到100ms，触点移到控件外，就会清除PREPRESSED标志。

```java
private void removeTapCallback() {
    if (mPendingCheckForTap != null) {
        mPrivateFlags &= ~PFLAG_PREPRESSED;
        removeCallbacks(mPendingCheckForTap);
    }
}
```

如果已经出发长按检测，就需要把长按检测移除。

```java
private void removeLongPressCallback() {
    if (mPendingCheckForLongPress != null) {
      removeCallbacks(mPendingCheckForLongPress);
    }
}
```


### 4.3 MotionEvent.ACTION_UP

```java
// 判断mPrivateFlags是否包含PREPRESSED
boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;

// PRESSED或PREPRESSED有一个就执行
if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
    // take focus if we don't have it already and we should in
    // touch mode.
    boolean focusTaken = false;
    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
        focusTaken = requestFocus();
    }

    if (prepressed) {
        // The button is being released before we actually
        // showed it as pressed.  Make it show the pressed
        // state now (before scheduling the click) to ensure
        // the user sees it.
        setPressed(true, x, y);
    }
    
    // 没有长按且不忽略下一个抬起事件，就移除长按     
    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {

        removeLongPressCallback();

        // 只有在按下的状态才执行点击操作
        if (!focusTaken) {
            // 用Runnable提交而不是直接执行是为了让其他可见View在点击操作开始之前更新
            if (mPerformClick == null) {
                mPerformClick = new PerformClick();
            }

            // 通过handler添加到消息队列尾部，失败就直接执行performClick()
            if (!post(mPerformClick)) {
                performClick();
            }
        }
    }

    if (mUnsetPressedState == null) {
        mUnsetPressedState = new UnsetPressedState();
    }
    
    if (prepressed) {
		postDelayed(mUnsetPressedState,
   	        ViewConfiguration.getPressedStateDuration()); // 64ms
    } else if (!post(mUnsetPressedState)) {
        mUnsetPressedState.run(); // Post失败就取消按下状态
    }

    removeTapCallback();
}
mIgnoreNextUpEvent = false;
```

`performClick()`在`ACTION_UP`的过程中被调用的，`onClick`事件是在这里被执行的。

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK); // 点击音效
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```

```java
private final class UnsetPressedState implements Runnable {
    @Override
    public void run() {
        setPressed(false);
    }
}
```

