---
layout:     post
title:      "为什么ViewStub不能填充两次"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

```java
/**
 * A ViewStub is an invisible, zero-sized View that can be used to lazily inflate
 * layout resources at runtime.
 *
 * When a ViewStub is made visible, or when {@link #inflate()}  is invoked, the layout resource 
 * is inflated. The ViewStub then replaces itself in its parent with the inflated View or Views.
 * Therefore, the ViewStub exists in the view hierarchy until {@link #setVisibility(int)} or
 * {@link #inflate()} is invoked.
 *
 * The inflated View is added to the ViewStub's parent with the ViewStub's layout
 * parameters. Similarly, you can define/override the inflate View's id by using the
 * ViewStub's inflatedId property. For instance:
 *
 * <pre>
 *     &lt;ViewStub android:id="@+id/stub"
 *               android:inflatedId="@+id/subTree"
 *               android:layout="@layout/mySubTree"
 *               android:layout_width="120dip"
 *               android:layout_height="40dip" /&gt;
 * </pre>
 *
 * The ViewStub thus defined can be found using the id "stub." After inflation of
 * the layout resource "mySubTree," the ViewStub is removed from its parent. The
 * View created by inflating the layout resource "mySubTree" can be found using the
 * id "subTree," specified by the inflatedId property. The inflated View is finally
 * assigned a width of 120dip and a height of 40dip.
 *
 * The preferred way to perform the inflation of the layout resource is the following:
 *
 * <pre>
 *     ViewStub stub = findViewById(R.id.stub);
 *     View inflated = stub.inflate();
 * </pre>
 *
 * When {@link #inflate()} is invoked, the ViewStub is replaced by the inflated View
 * and the inflated View is returned. This lets applications get a reference to the
 * inflated View without executing an extra findViewById().
 *
 * @attr ref android.R.styleable#ViewStub_inflatedId
 * @attr ref android.R.styleable#ViewStub_layout
 */
public final class ViewStub extends View {
```

```java
private int mInflatedId;
private int mLayoutResource;

private WeakReference<View> mInflatedViewRef;
```

```java
public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context);

    final TypedArray a = context.obtainStyledAttributes(attrs,
            R.styleable.ViewStub, defStyleAttr, defStyleRes);
    mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
    mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
    mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
    a.recycle();

    setVisibility(GONE);
    setWillNotDraw(true);
}
```

```java
/**
 * Inflates the layout resource identified by {@link #getLayoutResource()}
 * and replaces this StubbedView in its parent by the inflated layout resource.
 *
 * @return The inflated layout resource.
 *
 */
public View inflate() {
    final ViewParent viewParent = getParent();

    if (viewParent != null && viewParent instanceof ViewGroup) {
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
            final View view = inflateViewNoAdd(parent);
            replaceSelfWithView(view, parent);

            mInflatedViewRef = new WeakReference<>(view);
            if (mInflateListener != null) {
                mInflateListener.onInflate(this, view);
            }

            return view;
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

```java
private View inflateViewNoAdd(ViewGroup parent) {
    final LayoutInflater factory;
    if (mInflater != null) {
        factory = mInflater;
    } else {
        factory = LayoutInflater.from(mContext);
    }
    final View view = factory.inflate(mLayoutResource, parent, false);

    if (mInflatedId != NO_ID) {
        view.setId(mInflatedId);
    }
    return view;
}
```

```java
private void replaceSelfWithView(View view, ViewGroup parent) {
    final int index = parent.indexOfChild(this);
    parent.removeViewInLayout(this);

    final ViewGroup.LayoutParams layoutParams = getLayoutParams();
    if (layoutParams != null) {
        parent.addView(view, index, layoutParams);
    } else {
        parent.addView(view, index);
    }
}
```



[](/2018/03/03/LayoutInflater/#六视图创建)

```java
// 创建类实例，args是自定义主题相关变量
final View view = constructor.newInstance(args);

// 类型是ViewStub
if (view instanceof ViewStub) {
    // 使用同一个Context给ViewStub设置LayoutInflater
    // 后面View填充是会使用此LayoutInflater
    final ViewStub viewStub = (ViewStub) view;
    viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
}
mConstructorArgs[0] = lastContext;
return view;
```



https://stackoverflow.com/questions/23783101/how-to-check-if-a-viewstub-is-already-inflated