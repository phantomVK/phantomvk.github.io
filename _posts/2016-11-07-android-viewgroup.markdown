---
layout:     post
title:      "Android源码系列(2) -- ViewGroup"
date:       2016-11-07
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---


# 前言

上一篇文章 [Android View 事件分发源码剖析](http://phantomvk.coding.me/2016/10/18/Android_View/) 我们详细介绍了View事件分发的细节。接下来我们继续学习ViewGroup事件分发的内容。此次源码同样基于Android SDK 23，即Android 6.0。如果你看的是以前版本的源码，可能会有明显的不一样，请自行斟酌。

# 一、 代码构建

继承LinearLayout类，重写**dispatchTouchEvent**、**onInterceptTouchEvent**、**onTouchEvent**三个方法。

```java
public class MyLinearLayout extends LinearLayout {

    private static final String TAG = "MyLinearLayout";

    public MyLinearLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int action = ev.getAction();
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
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        int action = ev.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                Log.e(TAG, "onInterceptTouchEvent ACTION_DOWN");
                break;
            case MotionEvent.ACTION_MOVE:
                Log.e(TAG, "onInterceptTouchEvent ACTION_MOVE");
                break;
            case MotionEvent.ACTION_UP:
                Log.e(TAG, "onInterceptTouchEvent ACTION_UP");
                break;
            default:
                break;
        }
        return super.onInterceptTouchEvent(ev);
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

然后直接修改上次的xml布局文件，把`RelativeLayout`改为自定义的**com.corevk.demoproject.MyLinearLayout**。

```xml
<com.corevk.demoproject.MyLinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.corevk.demoproject.MyButton
        android:id="@+id/MyButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CLICK ME"/>
</com.corevk.demoproject.MyLinearLayout>
```

# 二、运行结果

事件按照一定规律传递:

* MyLinearLayout: dispatchTouchEvent
* MyLinearLayout: onInterceptTouchEvent
* MyButton: dispatchTouchEvent
* MyButton: onTouchEvent

事件首先分发到`ViewGroup`中，然后`ViewGroup`分发到`View`。由于`View`是一个设置了`OnClickListener`的`Button`，所以`Button.onTouchEvent()`在父类中返回`true`终止分发。

```
demoproject E/MyLinearLayout: dispatchTouchEvent ACTION_DOWN
demoproject E/MyLinearLayout: onInterceptTouchEvent ACTION_DOWN
demoproject E/MyButton: dispatchTouchEvent ACTION_DOWN
demoproject E/MyButton: onTouchEvent ACTION_DOWN

demoproject E/MyLinearLayout: dispatchTouchEvent ACTION_MOVE
demoproject E/MyLinearLayout: onInterceptTouchEvent ACTION_MOVE
demoproject E/MyButton: dispatchTouchEvent ACTION_MOVE
demoproject E/MyButton: onTouchEvent ACTION_MOVE

demoproject E/MyLinearLayout: dispatchTouchEvent ACTION_UP
demoproject E/MyLinearLayout: onInterceptTouchEvent ACTION_UP
demoproject E/MyButton: dispatchTouchEvent ACTION_UP
demoproject E/MyButton: onTouchEvent ACTION_UP
```

# 三、源码剖析

## 3.1 ViewGroup 

### 3.1.1 dispatchTouchEvent

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {

    // 通知verifier检查Touch事件
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }

    // 是否已经处理标志位
    boolean handled = false;
    
    // 检查是否被其他View覆盖
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 有MotionEvent.ACTION_DOWN事件
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 开始一个新的触摸动作时先丢弃之前所有的状态
            // 框架可能由于APP切换、ANR或其他状态改变，结束了先前抬起或取消事件
            cancelAndClearTouchTargets(ev); // 取消并清除触摸的Targets
            resetTouchState(); // 重置触摸状态
        }

        // 检查拦截器
        final boolean intercepted;
        
        // 发生ACTION_DOWN事件或者已经发生ACTION_DOWN，进入此方法
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            // 检查是否允许调用拦截器
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); 
            } else {
                intercepted = false;
            }
        } else {
            // 如果没有触摸targets，且这个操作不是down事件，这个viewgroup会继续拦截
            intercepted = true;
        }

        // 如果已拦截或正在处理gesture，开始正常的事件分发
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }
        
        // 检查取消
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // 更新触摸目标的列表
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        // 没有取消且没有拦截就执行
        if (!canceled && !intercepted) {

            // 把事件分发给的子视图，寻找能获取焦点的视图
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;
            
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                //清空之前的触摸对象
                removePointersFromTouchTargets(idBitsToAssign);
                // 统计子视图数目
                final int childrenCount = mChildrenCount;
                // 新的触摸Target不为空且有子视图
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    
                    // 从视图最上层到底层，获取所有能接收该事件的子视图
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                    // 从最底层的父视图开始遍历找寻newTouchTarget
                    // 如果已经存在找寻newTouchTarget，说明正在接收触摸事件，则跳出循环。
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // 若当前视图无法获取用户焦点，跳过本次循环
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        
                        // view不能接收坐标事件或者触摸坐标点不在view的范围内，跳过本次循环
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        // 已经开始接收触摸事件，结束整个循环
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
                        
                        // 重置取消下一个抬起标志位
                        resetCancelNextUpFlag(child);
                        
                        // 如果触摸位置在child的区域内，则把事件分发给对应的子View或ViewGroup
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            mLastTouchDownTime = ev.getDownTime();
                            // 获取TouchDown的下标
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            // 获取TouchDown的坐标
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 添加TouchTarget，mFirstTouchTarget不为null
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 已经分发给NewTouchTarget，退出
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                        ev.setTargetAccessibilityFocus(false);
                    }
                    // 清除视图列表
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 将mFirstTouchTarget链表最后的touchTarget赋给newTouchTarget
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
        
        // 只有处理ACTION_DOWN事件才会进入addTouchTarget方法。
        if (mFirstTouchTarget == null) {
        // 没有触摸target，交给当前ViewGroup来处理
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
        } else {
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // 当发生抬起或取消事件，重置触摸targets
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }
    
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

### 3.1.3 onFilterTouchEventForSecurity

隐私策略过滤触摸事件返回状态值。true表示继续分发事件，而false表示该事件应该被过滤掉不再进行任何分发。

```java
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
            && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // Window is obscured, drop this touch.
        return false;
    }
    return true;
}
```

### 3.1.3 requestDisallowInterceptTouchEvent

```java
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```


### 3.1.4 buildOrderedChildList()

建立一个视图组的列表，通过虚拟的Z轴来进行排序。

```java
ArrayList<View> buildOrderedChildList() {
    final int count = mChildrenCount;
    if (count <= 1 || !hasChildWithZ()) return null;

    if (mPreSortedChildren == null) {
        mPreSortedChildren = new ArrayList<View>(count);
    } else {
        mPreSortedChildren.ensureCapacity(count);
    }

    final boolean useCustomOrder = isChildrenDrawingOrderEnabled();
    for (int i = 0; i < mChildrenCount; i++) {
        // 添加下一个子视图到列表尾
        int childIndex = useCustomOrder ? getChildDrawingOrder(mChildrenCount, i) : i;
        View nextChild = mChildren[childIndex];
        float currentZ = nextChild.getZ();

        int insertIndex = i;
        while (insertIndex > 0 && mPreSortedChildren.get(insertIndex - 1).getZ() > currentZ) {
            insertIndex--;
        }
        mPreSortedChildren.add(insertIndex, nextChild);
    }
    return mPreSortedChildren;
}
```

### 3.1.5 dispatchTransformedTouchEvent

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    // 发生取消操作时不再执行后续的任何操作
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    // 由于一些原因发生不一致的操作时抛弃该事件
    if (newPointerIdBits == 0) {
        return false;
    }

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    // 判断新pointer id与旧pointer id是否相等
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                // 不存在子视图时，ViewGroup就调用View.dispatchTouchEvent分发事件
                // 再调用ViewGroup.onTouchEvent来处理事件
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);
                // 将触摸事件分发给子ViewGroup或View;
                handled = child.dispatchTouchEvent(event);
                
                // 调整该事件的位置
                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        // 分离事件，获取包含newPointerIdBits的MotionEvent
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    // 不存在子视图时，ViewGroup调用View.dispatchTouchEvent分发事件，再调用ViewGroup.onTouchEvent来处理事件
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        
        //将触摸事件分发给子ViewGroup或View;
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // 结束，回收transformedEvent
    transformedEvent.recycle();
    return handled;
}
```
    
    
### 3.1.6 addTouchTarget

调用该方法获取了TouchTarget。同时mFirstTouchTarget被赋予相同对象。
    
```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

## 3.2 ViewGroup - onInterceptTouchEvent

方法默认返回false，表示继续执行事件分发。如果该方法被重写并放回true，事件被拦截并不再分发。

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```





