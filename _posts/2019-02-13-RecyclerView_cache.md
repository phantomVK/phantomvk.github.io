---
layout:     post
title:      "RecyclerView缓存机制"
date:       2019-02-13
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## 一、RecyclerView与性能

使用 __RecyclerView__ 的难度可大可小，若仅展示单类型列表，只要优化视图绘制性能、降低布局复杂度，就能保证性能。

若列表分类多、样式差异大，类似微信聊天消息界面，遇到问题的难度将大幅增加。需要在预加载、复用上做进一步调优，单纯实现 __onCreateViewHolder()__ 和 __onBindViewHolder()__ 并不能满足需求。总的来说，要以追求视图出现在屏幕前耗费最少时间为目标。

__RecyclerView__ 缓存分为3级，每级有各自的缓存数量和策略。

![RecyclerView_cache_level](/img/android/RecyclerView/RecyclerView_cache_level.png)

当所有缓存层均没有所需实例，最后由 __onCreateViewHolder()__ 创建并绑定数据。源码版本：Android 27.1.1

## 二、一级缓存

一级缓存包含三个容器实例：__mAttachedScrap__、__mChangedScrap__、__mCachedViews__。根据不同场景 __ViewHolder__ 缓存到不同容器。

__mAttachedScrap__ 保存依附于 __RecyclerView__ 的 __ViewHolder__。包含移出屏幕但未从 __RecyclerView__ 移除的 __ViewHolder__。

```java
final ArrayList<RecyclerView.ViewHolder> mAttachedScrap = new ArrayList();
```

__mChangedScrap__ 保存数据发生改变的 __ViewHolder__，即调用 __notifyDataSetChanged()__ 等系列方法后需要更新的 __ViewHolder__。

```java
ArrayList<RecyclerView.ViewHolder> mChangedScrap = null;
```

__mCachedViews__ 用于解决滑动抖动的问题，默认容量为2，可根据需要调优。

```java
final ArrayList<RecyclerView.ViewHolder> mCachedViews = new ArrayList();
```

## 三、二级缓存

开发者自定义的缓存，需实现 __ViewCacheExtension__ 抽象类。若没有定义此缓存默认为null。

```java
private RecyclerView.ViewCacheExtension mViewCacheExtension;
```

## 四、三级缓存

__mCachedViews__ 无法保存屏幕上所有移除的 __ViewHolder__ 时，剩余的 __ViewHolder__ 根据 __type__ 分类放入缓存池中。

```java
RecyclerView.RecycledViewPool mRecyclerPool;
```

## 五、Recycler

#### 5.1 tryGetViewHolderForPositionByDeadline()

以下是 __RecyclerView__ 的内部类 __Recycler__ 去除类签名的源码。__Adapter__ 利用position获取 __ViewHolder__：若一级缓存命失、__mViewCacheExtension__ 为空，则从缓存池查找对象。

```java
@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    // 越界检查
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException("Invalid item position " + position
                + "(" + position + "). Item count:" + mState.getItemCount()
                + exceptionLabel());
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;

    // 0) 从mChangedScrap获取ViewHolder
    if (mState.isPreLayout()) {
        // 用position作为参数
        holder = getChangedScrapViewForPosition(position);
        // from scrap.
        fromScrapOrHiddenOrCache = holder != null;
    }

    // 1) 用position从scrap、hidden list、cache中获取
    if (holder == null) {
        // 从第一级缓存中获取，从mAttachedScrap和mCachedViews中获取ViewHolder
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                // recycle holder (and unscrap if relevant) since it can't be used
                if (!dryRun) {
                    // we would like to recycle this but need to make sure it is not used by
                    // animation logic etc.
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    
    // 用posotion查找缓存失败，尝试使用stable ids获取缓存
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }

        // 用offsetPosition获取ViewType
        final int type = mAdapter.getItemViewType(offsetPosition);
        if (mAdapter.hasStableIds()) {
            // 2) 通过stable ids和type从scrap/cache查找
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // 更新position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        
        // 从第二级缓存ViewCacheExtension中获取缓存内容，即mViewCacheExtension
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view which does not have a ViewHolder"
                            + exceptionLabel());
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view that is ignored. You must call stopIgnoring before"
                            + " returning this view." + exceptionLabel());
                }
            }
        }
        
        // 从三级缓存RecycledViewPool中获取缓存内容
        // ViewHolder内layout可重用，但是数据需重新绑定
        if (holder == null) {
            // 根据目标类型获取ViewHolder
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        
        // 从所有缓存中均没法获取对应数据，只能通过Adapter构造新的ViewHolder
        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                    && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            // 这里调用继承Adapter后实现的createViewHolder方法
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            if (ALLOW_THREAD_GAP_WORK) {
                // only bother finding nested RV if prefetching
                RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                if (innerView != null) {
                    holder.mNestedRecyclerView = new WeakReference<>(innerView);
                }
            }

            long end = getNanoTime();
            mRecyclerPool.factorInCreateTime(type, end - start);
            }
        }
    }

    // This is very ugly but the only place we can grab this information
    // before the View is rebound and returned to the LayoutManager for post layout ops.
    // We don't need this in pre-layout since the VH is not updated by the LM.
    if (fromScrapOrHiddenOrCache && !mState.isPreLayout() && holder
            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST)) {
        holder.setFlags(0, ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
        if (mState.mRunSimpleAnimations) {
            int changeFlags = ItemAnimator
                    .buildAdapterChangeFlagsForAnimations(holder);
            changeFlags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
            final ItemHolderInfo info = mItemAnimator.recordPreLayoutInformation(mState,
                    holder, changeFlags, holder.getUnmodifiedPayloads());
            recordAnimationInfoIfBouncedHiddenView(holder, info);
        }
    }

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        // do not update unless we absolutely have to.
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        if (DEBUG && holder.isRemoved()) {
            throw new IllegalStateException("Removed holder should be bound and it should"
                    + " come here only in pre-layout. Holder: " + holder
                    + exceptionLabel());
        }
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        // 以下方法调用mAdapter.bindViewHolder把内容绑定到ViewHolder
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }

    final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
    final LayoutParams rvLayoutParams;
    if (lp == null) {
        rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else if (!checkLayoutParams(lp)) {
        rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else {
        rvLayoutParams = (LayoutParams) lp;
    }
    rvLayoutParams.mViewHolder = holder;
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
    return holder;
}
```

#### 5.2 getScrapOrHiddenOrCachedHolderForPosition()

从 __attach scrap__、__hidden children__ 或 __cache__ 根据 __position__ 返回 __ViewHolder__

```java
// @param position 条目位置
// @param dryRun   空转，只查找ViewHolder而不移除
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();

    // Try first for an exact, non-invalid match from scrap.
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }

    if (!dryRun) {
        View view = mChildHelper.findHiddenNonRemovedView(position);
        if (view != null) {
            // This View is good to be used. We just need to unhide, detach and move to the
            // scrap list.
            final ViewHolder vh = getChildViewHolderInt(view);
            mChildHelper.unhide(view);
            int layoutIndex = mChildHelper.indexOfChild(view);
            if (layoutIndex == RecyclerView.NO_POSITION) {
                throw new IllegalStateException("layout index should not be -1 after "
                        + "unhiding a view:" + vh + exceptionLabel());
            }
            mChildHelper.detachViewFromParent(layoutIndex);
            scrapView(view);
            vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                    | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
            return vh;
        }
    }

    // 从一级缓存查找已回收的视图缓存
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        // invalid view holders may be in cache if adapter has stable ids as they can be
        // retrieved via getScrapOrCachedViewForId
        if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}
```

#### 5.3 tryBindViewHolderByDeadline()

```java
private boolean tryBindViewHolderByDeadline(ViewHolder holder, int offsetPosition,
        int position, long deadlineNs) {
    // 设置holder的RecyclerView
    holder.mOwnerRecyclerView = RecyclerView.this;
    // holder的viewType
    final int viewType = holder.getItemViewType();
    long startBindNs = getNanoTime();
    if (deadlineNs != FOREVER_NS
            && !mRecyclerPool.willBindInTime(viewType, startBindNs, deadlineNs)) {
        // abort - we have a deadline we can't meet
        return false;
    }
    // 绑定视图数据
    mAdapter.bindViewHolder(holder, offsetPosition);
    long endBindNs = getNanoTime();
    mRecyclerPool.factorInBindTime(holder.getItemViewType(), endBindNs - startBindNs);
    attachAccessibilityDelegateOnBind(holder);
    if (mState.isPreLayout()) {
        holder.mPreLayoutPosition = position;
    }
    return true;
}
```

## 六、RecycledViewPool

若要把 __RecycledViewPool__ 在多个 __RecyclerViews__ 间共享，只需主动创建该实例，通过 __RecyclerView.setRecycledViewPool(RecycledViewPool)__ 绑定到 __RecyclerView__ 上。如果没有给 __RecyclerView__ 指定任何 __RecycledViewPool__，则会自行创建该实例。

每个 __type__ 默认缓存5个 __ViewHolder__，可针对不同 __type__ 修改需缓存数量。例如：增加显示面积较小 __ViewHolder__ 缓存的数量，保证缓存对象足够填满屏幕又无需创建新对象。

```java
private static final int DEFAULT_MAX_SCRAP = 5;
```

__ScrapData__ 是 __RecycledViewPool__ 的内部类
```java
static class ScrapData {
    // 保存ViewHolder的列表
    final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
    // 本类型最多可保留多少ViewHolder
    int mMaxScrap = DEFAULT_MAX_SCRAP;
    // 记录创建视图的平均时长
    long mCreateRunningAverageNs = 0;
    // 记录视图绑定的平均时长
    long mBindRunningAverageNs = 0;
}
```

根据分类缓存 __ScrapData__

```java
SparseArray<ScrapData> mScrap = new SparseArray<>();
```

调整指定 __viewType__ 视图缓存的最大值

```java
public void setMaxRecycledViews(int viewType, int max) {
    ScrapData scrapData = getScrapDataForType(viewType);
    // 修改mMaxScrap的值
    scrapData.mMaxScrap = max;
    final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
    // 裁剪多余缓存实例
    while (scrapHeap.size() > max) {
        scrapHeap.remove(scrapHeap.size() - 1);
    }
}
```

例如：下图样式的 __ViewHolder__ 仅缓存5个，多余视图移出屏幕后会销毁。下次需要该 __ViewHolder__ 又要重新构建，所以提高缓存数量可减少这种情况发生次数。

![RecyclerView_demo](/img/android/RecyclerView/RecyclerView_demo_30.png)

根据 __viewType__ 从缓存池获取 __ScrapData__，再从 __ScrapData__ 取出有效 __ViewHolder__。如果缓存池内没有缓存该实例则返回null。

```java
@Nullable
public ViewHolder getRecycledView(int viewType) {
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        // 从列表取出一个ViewHolder
        return scrapHeap.remove(scrapHeap.size() - 1);
    }
    return null;
}
```

读取 __ViewHolder__ 的 __viewType__ 并找到对应 __scrapHeap__ 列表，把 __ViewHolder__ 缓存到该列表

```java
public void putRecycledView(ViewHolder scrap) {
    final int viewType = scrap.getItemViewType();
    final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
    if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
        return;
    }
    scrap.resetInternal();
    scrapHeap.add(scrap);
}
```

根据 __viewType__ 获取 __ScrapData__

```java
private ScrapData getScrapDataForType(int viewType) {
    ScrapData scrapData = mScrap.get(viewType);
    if (scrapData == null) {
        scrapData = new ScrapData();
        mScrap.put(viewType, scrapData);
    }
    return scrapData;
}
```

## 七、参考链接

- [RecyclerView缓存原理，有图有真相](https://juejin.im/post/5b79a0b851882542b13d204b)