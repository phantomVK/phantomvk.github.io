---
layout:     post
title:      "Android源码系列(2) -- ViewGroup事件分发"
date:       2016-11-07
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---


# 前言

上一篇文章 [Android View 事件分发源码剖析](http://phantomvk.coding.me/2016/10/18/Android_View/) 详细分析View事件分发的细节。接下来继续学习ViewGroup事件分发。此次源码同样基于Android SDK 23，即Android 6.0。不同Framework源码可能不一样，请自行斟酌。

# 一、 代码构建

继承LinearLayout重写**dispatchTouchEvent()**、**onInterceptTouchEvent()**、**onTouchEvent()**方法。

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

然后直接修改上次的xml布局文件，把RelativeLayout改为自定义ViewGroup:

```xml
<com.phantomvk.demoproject.MyLinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.phantomvk.demoproject.MyButton
        android:id="@+id/MyButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CLICK ME"/>

</com.phantomvk.demoproject.MyLinearLayout>
```

# 二、运行结果

事件按照以下顺序传递：

* MyLinearLayout: dispatchTouchEvent
* MyLinearLayout: onInterceptTouchEvent
* MyButton: dispatchTouchEvent
* MyButton: onTouchEvent

事件分发流程：

- `ViewGroup.dispatchTouchEvent()`交给`ViewGroup.onInterceptTouchEvent()`；


- 事件进入`ViewGroup.onInterceptTouchEvent()`，该方法返回`false`继续下发；
- 分发给子View的`dispatchTouchEvent()`，传递到`onTouchEvent.OnTouchListener`消费；
- 如果`OnTouchListener`不拦截事件，则交给`View.onTouchEvent()`消费.

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

流程图：

![ViewGroup_View_dispatchTouchEvent](/img/android/event/ViewGroup_View_dispatchTouchEvent.png)

# 三、源码剖析

#### 3.1 dispatchTouchEvent

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
            // 开始新触摸动作时先丢弃之前所有状态
            // 框架可能由于APP切换、ANR或其他状态改变，结束了先前抬起或取消事件
            cancelAndClearTouchTargets(ev); // 取消并清除触摸的Targets
            resetTouchState(); // 重置触摸状态
        }

        // 是否已拦截标志位
        final boolean intercepted;
        
        // 发生ACTION_DOWN事件或者ACTION_DOWN的后续事件，进入此方法
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            // ViewGroup自身是否允许拦截事件
            if (!disallowIntercept) {
                // ViewGroup自行拦截事件并发送到onInterceptTouchEvent(ev)
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); 
            } else {
                // 不允许拦截
                intercepted = false;
            }
        } else {
            // 没有FirstTouchTarget且不是down事件，ViewGroup拦截事件
            intercepted = true;
        }

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
        
        // 之前不允许拦截事件，或onInterceptTouchEvent(ev)返回false
        if (!canceled && !intercepted) {

            // 把事件分发给的子视图，寻找能获取焦点的视图
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;
            
            // ACTION_DOWN开始新的事件序列
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
                // 新触摸Target为空且有子视图
                if (newTouchTarget == null && childrenCount != 0) {
                    // 当前x、y坐标
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    
                    // 从视图最上层到底层，获取所有能接收该事件的子视图
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                    // 从最底层的父视图开始遍历找寻newTouchTarget
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // child的索引
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        // 从children列表获取索引的child
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // 若当前视图无法获取用户焦点，跳过
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        
                        // view不能接收坐标事件或者触摸坐标点不在view的范围内，跳过
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        // 找到子View，封装为TouchTarget
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
                        
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
                                // 最近一个消费事件的childIndex
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

                // 没有找到接收事件的View
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
            // 有子View接收事件，则后续操作事件也分发给它
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

#### 3.2 onFilterTouchEventForSecurity

隐私策略过滤触摸事件返回状态值。true表示继续分发事件，false表示事件被过滤掉不再分发。

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

#### 3.3 requestDisallowInterceptTouchEvent

禁止 __ViewGroup__ 拦截事件

```java
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    // 如果disallowIntercept和ViewGroup的值一致，则不需要改变该值
    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        return;
    }

    // 改变该ViewGroup对应标志位
    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    // 通知该ViewGroup的父ViewGroup逐层向上修改disallowIntercept
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```


#### 3.4 buildOrderedChildList

建立一个视图组的列表，通过虚拟的Z轴来进行排序。

```java
ArrayList<View> buildOrderedChildList() {
    final int count = mChildrenCount;
    if (count <= 1 || !hasChildWithZ()) return null;

    if (mPreSortedChildren == null) {
        // 创建新列表
        mPreSortedChildren = new ArrayList<View>(count);
    } else {
        // 检查已有列表容量是否足够，容量不足则进行扩容
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

#### 3.5 dispatchTransformedTouchEvent

分发已变换的点击事件

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
            // 传入View为空，调用父类dispatchTouchEvent()
            handled = super.dispatchTouchEvent(event);
        } else {
            // 调用子View的dispatchTouchEvent()，事件进入到子View
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


#### 3.6 addTouchTarget

调用该方法获取了TouchTarget。同时mFirstTouchTarget被赋予相同对象。
```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

#### 3.7 onInterceptTouchEvent

方法默认返回false，表示继续执行事件分发。如果该方法被重写并返回true，事件被拦截并不再分发。

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```

#### 3.8 cancelAndClearTouchTargets 

取消并移除所有触控目标

```java
private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        if (event == null) {
            final long now = SystemClock.uptimeMillis();
            event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            syntheticEvent = true;
        }

        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            resetCancelNextUpFlag(target.child);
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
        }
        clearTouchTargets();

        if (syntheticEvent) {
            event.recycle();
        }
    }
}
```

#### 3.9 resetTouchState

```java
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
```

#### 3.10 findChildWithAccessibilityFocus

查找拥有焦点的子视图

```java
private View findChildWithAccessibilityFocus() {
    // 获取ViewRoot
    ViewRootImpl viewRoot = getViewRootImpl();
    if (viewRoot == null) {
        return null;
    }

    // 获取当前焦点的View
    View current = viewRoot.getAccessibilityFocusedHost();
    if (current == null) {
        return null;
    }

    // 查找View的父类，看其直接或间接父类视为被文本ViewGroup
    ViewParent parent = current.getParent();
    while (parent instanceof View) {
        if (parent == this) {
            return current;
        }
        current = (View) parent;
        parent = current.getParent();
    }

    // 焦点View的父类不是本ViewGroup，返回null
    return null;
}
```




