---
layout:     post
title:      "HeightListView - 自适应TextView高度"
date:       2017-08-27
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

使用`Adapter`可能需要在一个`item`里包含`ListView`控件动态展示多个`TextView`。普通的`ListView`会出现所有的`TextView`只显示第一行而不是多行内容的问题。

问题的根源是`ListView`没有正确计算出每个`TextView`文字需要的高度，仅显示一行文字的高度。

重写的的类中，把计算高度的`onItemsChanged()`通过`DataSetObserver`绑定到数据对应的`adapter`，且重写`onMeasure(int, int)`。数据改变时回调`DataSetObserver`，新数据对应的`TextView`高度得到计算。

`DataSetObserver`在初始化`ListView`的时候已经创建完成，在`setAdapter(ListAdapter adapter)`中绑定到`adapter`。

经过上面的处理，令`HeightListView`与`ListView`用法无异。

```java
/**
 * Auto height calculating ListView.
 * Calculates items' total height which includes some TextViews.
 * Use it just like a common ListView.
 * Author: PhantomVK
 */
public final class HeightListView extends ListView {

    private DataSetObserver mDataSetObserver; // The custom data set observer.

    public HeightListView(Context context) {
        this(context, null);
    }

    public HeightListView(Context context, AttributeSet attrs) {
        super(context, attrs);
        createObserver();
    }

    public HeightListView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        createObserver();
    }

    /**
     * Overrides to set a suitable measure spec.
     *
     * @param widthMeasureSpec  widthMeasureSpec, not changed.
     * @param heightMeasureSpec heightMeasureSpec replaced with a custom one.
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int spec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
        super.onMeasure(widthMeasureSpec, spec);
    }

    /**
     * Sets the data behind this ListView.
     * <p>
     * Unregisters the observer of the pre adapter before which will be replaced.
     * After the new adapter is setted, registers a observer.
     *
     * @param adapter Set the new listView adapter
     */
    @Override
    public void setAdapter(ListAdapter adapter) {
        if (adapter == null) throw new NullPointerException();

        if (getAdapter() != null) {
            getAdapter().unregisterDataSetObserver(mDataSetObserver);
        }

        adapter.registerDataSetObserver(mDataSetObserver);
        super.setAdapter(adapter);
    }

    /**
     * Creates a custom data set change observer.
     */
    private void createObserver() {
        if (mDataSetObserver != null) return;
        mDataSetObserver = new DataSetObserver() {
            @Override
            public void onChanged() {
                super.onChanged();
                onItemsChanged();
            }
        };
    }

    /**
     * Calculates the height of all sub items, includes divider height of each item.
     */
    private void onItemsChanged() {
        int count = getAdapter().getCount();
        if (count == 0) return;

        int height = 0;
        for (int i = 0; i < count; i++) {
            View v = getAdapter().getView(i, null, this);
            v.measure(0, 0);
            height += v.getMeasuredHeight();
        }

        ViewGroup.LayoutParams params = getLayoutParams();
        params.height = height + getDividerHeight() * (count - 1);
        setLayoutParams(params);
    }

    @Override
    public boolean isClickable() {
        return super.isClickable();
    }

    @Override
    public boolean isEnabled() {
        return super.isEnabled();
    }
}
```

最后还把`isClickable()`和`isEnabled()`显式调用父类方法：`item`中`ListView`展示的所有`subitem`都是可以点击的。要`subitem`把点击事件传递给`item`处理，则需要重写或设置上述两个方法的值为`false`。

