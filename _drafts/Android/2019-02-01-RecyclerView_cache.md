---
layout:     post
title:      "RecyclerView缓存机制"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

Android 27.1.1

第一级缓存

```java
final ArrayList<RecyclerView.ViewHolder> mAttachedScrap = new ArrayList();
```

```java
ArrayList<RecyclerView.ViewHolder> mChangedScrap = null;
```

mCachedViews目的用于解决滑动抖动的问题，默认容量为2

```java
final ArrayList<RecyclerView.ViewHolder> mCachedViews = new ArrayList();
```

第二级缓存

开发者自定义的外部缓存，需实现ViewCacheExtension唯一抽象方法`getViewForPositionAndType()`。若没有定义的话本级缓存默认为null。

```java
private RecyclerView.ViewCacheExtension mViewCacheExtension;
```

第三级缓存

```java
RecyclerView.RecycledViewPool mRecyclerPool;
```

RecyclerView的内部类Recycler

```java
/**
 * Attempts to get the ViewHolder for the given position, either from the Recycler scrap,
 * cache, the RecycledViewPool, or creating it directly.
 * <p>
 * If a deadlineNs other than {@link #FOREVER_NS} is passed, this method early return
 * rather than constructing or binding a ViewHolder if it doesn't think it has time.
 * If a ViewHolder must be constructed and not enough time remains, null is returned. If a
 * ViewHolder is aquired and must be bound but not enough time remains, an unbound holder is
 * returned. Use {@link ViewHolder#isBound()} on the returned object to check for this.
 *
 * @param position Position of ViewHolder to be returned.
 * @param dryRun True if the ViewHolder should not be removed from scrap/cache/
 * @param deadlineNs Time, relative to getNanoTime(), by which bind/create work should
 *                   complete. If FOREVER_NS is passed, this method will not fail to
 *                   create/bind the holder if needed.
 *
 * @return ViewHolder for requested position
 */
@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException("Invalid item position " + position
                + "(" + position + "). Item count:" + mState.getItemCount()
                + exceptionLabel());
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;
    // 0) If there is a changed scrap, try to find from there
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    if (holder == null) {
        // 从第一级缓存中获取
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
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }

        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) Find from scrap/cache via stable ids, if exists
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        
        // 从第二级缓存ViewCacheExtension中获取缓存内容
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
        
        // 从第三级缓存RecycledViewPool中获取缓存内容
        // 获取ViewHolder内layout可重用，但是数据需要重新绑定
        if (holder == null) { // fallback to pool
            if (DEBUG) {
                Log.d(TAG, "tryGetViewHolderForPositionByDeadline("
                        + position + ") fetching from shared pool");
            }
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
            if (DEBUG) {
                Log.d(TAG, "tryGetViewHolderForPositionByDeadline created new ViewHolder");
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

第一级缓存获取

```java
/**
 * Returns a view for the position either from attach scrap, hidden children, or cache.
 *
 * @param position Item position
 * @param dryRun  Does a dry run, finds the ViewHolder but does not remove
 * @return a ViewHolder that can be re-used for this position.
 */
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

    // Search in our first-level recycled view cache.
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        // invalid view holders may be in cache if adapter has stable ids as they can be
        // retrieved via getScrapOrCachedViewForId
        if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            if (DEBUG) {
                Log.d(TAG, "getScrapOrHiddenOrCachedHolderForPosition(" + position
                        + ") found match in cache: " + holder);
            }
            return holder;
        }
    }
    return null;
}
```

内部类RecycledViewPool

```java
/**
 * RecycledViewPool lets you share Views between multiple RecyclerViews.
 * <p>
 * If you want to recycle views across RecyclerViews, create an instance of RecycledViewPool
 * and use {@link RecyclerView#setRecycledViewPool(RecycledViewPool)}.
 * <p>
 * RecyclerView automatically creates a pool for itself if you don't provide one.
 */
public static class RecycledViewPool {
    private static final int DEFAULT_MAX_SCRAP = 5;

    /**
     * Tracks both pooled holders, as well as create/bind timing metadata for the given type.
     *
     * Note that this tracks running averages of create/bind time across all RecyclerViews
     * (and, indirectly, Adapters) that use this pool.
     *
     * 1) This enables us to track average create and bind times across multiple adapters. Even
     * though create (and especially bind) may behave differently for different Adapter
     * subclasses, sharing the pool is a strong signal that they'll perform similarly, per type.
     *
     * 2) If {@link #willBindInTime(int, long, long)} returns false for one view, it will return
     * false for all other views of its type for the same deadline. This prevents items
     * constructed by {@link GapWorker} prefetch from being bound to a lower priority prefetch.
     */
    static class ScrapData {
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;
        long mBindRunningAverageNs = 0;
    }
    SparseArray<ScrapData> mScrap = new SparseArray<>();

    private int mAttachCount = 0;

    /**
     * Discard all ViewHolders.
     */
    public void clear() {
        for (int i = 0; i < mScrap.size(); i++) {
            ScrapData data = mScrap.valueAt(i);
            data.mScrapHeap.clear();
        }
    }

    /**
     * Sets the maximum number of ViewHolders to hold in the pool before discarding.
     *
     * @param viewType ViewHolder Type
     * @param max Maximum number
     */
    public void setMaxRecycledViews(int viewType, int max) {
        ScrapData scrapData = getScrapDataForType(viewType);
        scrapData.mMaxScrap = max;
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        while (scrapHeap.size() > max) {
            scrapHeap.remove(scrapHeap.size() - 1);
        }
    }

    /**
     * Returns the current number of Views held by the RecycledViewPool of the given view type.
     */
    public int getRecycledViewCount(int viewType) {
        return getScrapDataForType(viewType).mScrapHeap.size();
    }

    /**
     * Acquire a ViewHolder of the specified type from the pool, or {@code null} if none are
     * present.
     *
     * @param viewType ViewHolder type.
     * @return ViewHolder of the specified type acquired from the pool, or {@code null} if none
     * are present.
     */
    @Nullable
    public ViewHolder getRecycledView(int viewType) {
        final ScrapData scrapData = mScrap.get(viewType);
        if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
            final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
            return scrapHeap.remove(scrapHeap.size() - 1);
        }
        return null;
    }

    /**
     * Total number of ViewHolders held by the pool.
     *
     * @return Number of ViewHolders held by the pool.
     */
    int size() {
        int count = 0;
        for (int i = 0; i < mScrap.size(); i++) {
            ArrayList<ViewHolder> viewHolders = mScrap.valueAt(i).mScrapHeap;
            if (viewHolders != null) {
                count += viewHolders.size();
            }
        }
        return count;
    }

    /**
     * Add a scrap ViewHolder to the pool.
     * <p>
     * If the pool is already full for that ViewHolder's type, it will be immediately discarded.
     *
     * @param scrap ViewHolder to be added to the pool.
     */
    public void putRecycledView(ViewHolder scrap) {
        final int viewType = scrap.getItemViewType();
        final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
        if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
            return;
        }
        if (DEBUG && scrapHeap.contains(scrap)) {
            throw new IllegalArgumentException("this scrap item already exists");
        }
        scrap.resetInternal();
        scrapHeap.add(scrap);
    }

    long runningAverage(long oldAverage, long newValue) {
        if (oldAverage == 0) {
            return newValue;
        }
        return (oldAverage / 4 * 3) + (newValue / 4);
    }

    void factorInCreateTime(int viewType, long createTimeNs) {
        ScrapData scrapData = getScrapDataForType(viewType);
        scrapData.mCreateRunningAverageNs = runningAverage(
                scrapData.mCreateRunningAverageNs, createTimeNs);
    }

    void factorInBindTime(int viewType, long bindTimeNs) {
        ScrapData scrapData = getScrapDataForType(viewType);
        scrapData.mBindRunningAverageNs = runningAverage(
                scrapData.mBindRunningAverageNs, bindTimeNs);
    }

    boolean willCreateInTime(int viewType, long approxCurrentNs, long deadlineNs) {
        long expectedDurationNs = getScrapDataForType(viewType).mCreateRunningAverageNs;
        return expectedDurationNs == 0 || (approxCurrentNs + expectedDurationNs < deadlineNs);
    }

    boolean willBindInTime(int viewType, long approxCurrentNs, long deadlineNs) {
        long expectedDurationNs = getScrapDataForType(viewType).mBindRunningAverageNs;
        return expectedDurationNs == 0 || (approxCurrentNs + expectedDurationNs < deadlineNs);
    }

    void attach(Adapter adapter) {
        mAttachCount++;
    }

    void detach() {
        mAttachCount--;
    }


    /**
     * Detaches the old adapter and attaches the new one.
     * <p>
     * RecycledViewPool will clear its cache if it has only one adapter attached and the new
     * adapter uses a different ViewHolder than the oldAdapter.
     *
     * @param oldAdapter The previous adapter instance. Will be detached.
     * @param newAdapter The new adapter instance. Will be attached.
     * @param compatibleWithPrevious True if both oldAdapter and newAdapter are using the same
     *                               ViewHolder and view types.
     */
    void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,
            boolean compatibleWithPrevious) {
        if (oldAdapter != null) {
            detach();
        }
        if (!compatibleWithPrevious && mAttachCount == 0) {
            clear();
        }
        if (newAdapter != null) {
            attach(newAdapter);
        }
    }

    private ScrapData getScrapDataForType(int viewType) {
        ScrapData scrapData = mScrap.get(viewType);
        if (scrapData == null) {
            scrapData = new ScrapData();
            mScrap.put(viewType, scrapData);
        }
        return scrapData;
    }
}
```

