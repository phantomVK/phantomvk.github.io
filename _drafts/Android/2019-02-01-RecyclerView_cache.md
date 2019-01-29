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

第一级缓存

```java
final ArrayList<RecyclerView.ViewHolder> mAttachedScrap = new ArrayList();
ArrayList<RecyclerView.ViewHolder> mChangedScrap = null;
final ArrayList<RecyclerView.ViewHolder> mCachedViews = new ArrayList();
```

第二级缓存

```java
private RecyclerView.ViewCacheExtension mViewCacheExtension;
```

第三级缓存

```java
RecyclerView.RecycledViewPool mRecyclerPool;
```

RecyclerView的内部类Recycler

```java
@Nullable
RecyclerView.ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    if (position >= 0 && position < RecyclerView.this.mState.getItemCount()) {
        boolean fromScrapOrHiddenOrCache = false;
        RecyclerView.ViewHolder holder = null;
        if (RecyclerView.this.mState.isPreLayout()) {
            holder = this.getChangedScrapViewForPosition(position);
            fromScrapOrHiddenOrCache = holder != null;
        }

        if (holder == null) {
            holder = this.getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
            if (holder != null) {
                if (!this.validateViewHolderForOffsetPosition(holder)) {
                    if (!dryRun) {
                        holder.addFlags(4);
                        if (holder.isScrap()) {
                            RecyclerView.this.removeDetachedView(holder.itemView, false);
                            holder.unScrap();
                        } else if (holder.wasReturnedFromScrap()) {
                            holder.clearReturnedFromScrapFlag();
                        }

                        this.recycleViewHolderInternal(holder);
                    }

                    holder = null;
                } else {
                    fromScrapOrHiddenOrCache = true;
                }
            }
        }

        int offsetPosition;
        int type;
        if (holder == null) {
            offsetPosition = RecyclerView.this.mAdapterHelper.findPositionOffset(position);
            if (offsetPosition < 0 || offsetPosition >= RecyclerView.this.mAdapter.getItemCount()) {
                throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item position " + position + "(offset:" + offsetPosition + ")." + "state:" + RecyclerView.this.mState.getItemCount() + RecyclerView.this.exceptionLabel());
            }

            type = RecyclerView.this.mAdapter.getItemViewType(offsetPosition);
            if (RecyclerView.this.mAdapter.hasStableIds()) {
                holder = this.getScrapOrCachedViewForId(RecyclerView.this.mAdapter.getItemId(offsetPosition), type, dryRun);
                if (holder != null) {
                    holder.mPosition = offsetPosition;
                    fromScrapOrHiddenOrCache = true;
                }
            }

            if (holder == null && this.mViewCacheExtension != null) {
                View view = this.mViewCacheExtension.getViewForPositionAndType(this, position, type);
                if (view != null) {
                    holder = RecyclerView.this.getChildViewHolder(view);
                    if (holder == null) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned a view which does not have a ViewHolder" + RecyclerView.this.exceptionLabel());
                    }

                    if (holder.shouldIgnore()) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned a view that is ignored. You must call stopIgnoring before returning this view." + RecyclerView.this.exceptionLabel());
                    }
                }
            }

            if (holder == null) {
                holder = this.getRecycledViewPool().getRecycledView(type);
                if (holder != null) {
                    holder.resetInternal();
                    if (RecyclerView.FORCE_INVALIDATE_DISPLAY_LIST) {
                        this.invalidateDisplayListInt(holder);
                    }
                }
            }

            if (holder == null) {
                long start = RecyclerView.this.getNanoTime();
                if (deadlineNs != 9223372036854775807L && !this.mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                    return null;
                }

                holder = RecyclerView.this.mAdapter.createViewHolder(RecyclerView.this, type);
                if (RecyclerView.ALLOW_THREAD_GAP_WORK) {
                    RecyclerView innerView = RecyclerView.findNestedRecyclerView(holder.itemView);
                    if (innerView != null) {
                        holder.mNestedRecyclerView = new WeakReference(innerView);
                    }
                }

                long end = RecyclerView.this.getNanoTime();
                this.mRecyclerPool.factorInCreateTime(type, end - start);
            }
        }

        if (fromScrapOrHiddenOrCache && !RecyclerView.this.mState.isPreLayout() && holder.hasAnyOfTheFlags(8192)) {
            holder.setFlags(0, 8192);
            if (RecyclerView.this.mState.mRunSimpleAnimations) {
                offsetPosition = RecyclerView.ItemAnimator.buildAdapterChangeFlagsForAnimations(holder);
                offsetPosition |= 4096;
                RecyclerView.ItemAnimator.ItemHolderInfo info = RecyclerView.this.mItemAnimator.recordPreLayoutInformation(RecyclerView.this.mState, holder, offsetPosition, holder.getUnmodifiedPayloads());
                RecyclerView.this.recordAnimationInfoIfBouncedHiddenView(holder, info);
            }
        }

        boolean bound = false;
        if (RecyclerView.this.mState.isPreLayout() && holder.isBound()) {
            holder.mPreLayoutPosition = position;
        } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
            type = RecyclerView.this.mAdapterHelper.findPositionOffset(position);
            bound = this.tryBindViewHolderByDeadline(holder, type, position, deadlineNs);
        }

        android.view.ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
        RecyclerView.LayoutParams rvLayoutParams;
        if (lp == null) {
            rvLayoutParams = (RecyclerView.LayoutParams)RecyclerView.this.generateDefaultLayoutParams();
            holder.itemView.setLayoutParams(rvLayoutParams);
        } else if (!RecyclerView.this.checkLayoutParams(lp)) {
            rvLayoutParams = (RecyclerView.LayoutParams)RecyclerView.this.generateLayoutParams(lp);
            holder.itemView.setLayoutParams(rvLayoutParams);
        } else {
            rvLayoutParams = (RecyclerView.LayoutParams)lp;
        }

        rvLayoutParams.mViewHolder = holder;
        rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
        return holder;
    } else {
        throw new IndexOutOfBoundsException("Invalid item position " + position + "(" + position + "). Item count:" + RecyclerView.this.mState.getItemCount() + RecyclerView.this.exceptionLabel());
    }
}
```

内部类RecycledViewPool

```java
public static class RecycledViewPool {
    private static final int DEFAULT_MAX_SCRAP = 5;
    SparseArray<RecyclerView.RecycledViewPool.ScrapData> mScrap = new SparseArray();
    private int mAttachCount = 0;

    public RecycledViewPool() {
    }

    public void clear() {
        for(int i = 0; i < this.mScrap.size(); ++i) {
            RecyclerView.RecycledViewPool.ScrapData data = (RecyclerView.RecycledViewPool.ScrapData)this.mScrap.valueAt(i);
            data.mScrapHeap.clear();
        }

    }

    public void setMaxRecycledViews(int viewType, int max) {
        RecyclerView.RecycledViewPool.ScrapData scrapData = this.getScrapDataForType(viewType);
        scrapData.mMaxScrap = max;
        ArrayList scrapHeap = scrapData.mScrapHeap;

        while(scrapHeap.size() > max) {
            scrapHeap.remove(scrapHeap.size() - 1);
        }

    }

    public int getRecycledViewCount(int viewType) {
        return this.getScrapDataForType(viewType).mScrapHeap.size();
    }

    @Nullable
    public RecyclerView.ViewHolder getRecycledView(int viewType) {
        RecyclerView.RecycledViewPool.ScrapData scrapData = (RecyclerView.RecycledViewPool.ScrapData)this.mScrap.get(viewType);
        if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
            ArrayList<RecyclerView.ViewHolder> scrapHeap = scrapData.mScrapHeap;
            return (RecyclerView.ViewHolder)scrapHeap.remove(scrapHeap.size() - 1);
        } else {
            return null;
        }
    }

    int size() {
        int count = 0;

        for(int i = 0; i < this.mScrap.size(); ++i) {
            ArrayList<RecyclerView.ViewHolder> viewHolders = ((RecyclerView.RecycledViewPool.ScrapData)this.mScrap.valueAt(i)).mScrapHeap;
            if (viewHolders != null) {
                count += viewHolders.size();
            }
        }

        return count;
    }

    public void putRecycledView(RecyclerView.ViewHolder scrap) {
        int viewType = scrap.getItemViewType();
        ArrayList<RecyclerView.ViewHolder> scrapHeap = this.getScrapDataForType(viewType).mScrapHeap;
        if (((RecyclerView.RecycledViewPool.ScrapData)this.mScrap.get(viewType)).mMaxScrap > scrapHeap.size()) {
            scrap.resetInternal();
            scrapHeap.add(scrap);
        }
    }

    long runningAverage(long oldAverage, long newValue) {
        return oldAverage == 0L ? newValue : oldAverage / 4L * 3L + newValue / 4L;
    }

    void factorInCreateTime(int viewType, long createTimeNs) {
        RecyclerView.RecycledViewPool.ScrapData scrapData = this.getScrapDataForType(viewType);
        scrapData.mCreateRunningAverageNs = this.runningAverage(scrapData.mCreateRunningAverageNs, createTimeNs);
    }

    void factorInBindTime(int viewType, long bindTimeNs) {
        RecyclerView.RecycledViewPool.ScrapData scrapData = this.getScrapDataForType(viewType);
        scrapData.mBindRunningAverageNs = this.runningAverage(scrapData.mBindRunningAverageNs, bindTimeNs);
    }

    boolean willCreateInTime(int viewType, long approxCurrentNs, long deadlineNs) {
        long expectedDurationNs = this.getScrapDataForType(viewType).mCreateRunningAverageNs;
        return expectedDurationNs == 0L || approxCurrentNs + expectedDurationNs < deadlineNs;
    }

    boolean willBindInTime(int viewType, long approxCurrentNs, long deadlineNs) {
        long expectedDurationNs = this.getScrapDataForType(viewType).mBindRunningAverageNs;
        return expectedDurationNs == 0L || approxCurrentNs + expectedDurationNs < deadlineNs;
    }

    void attach() {
        ++this.mAttachCount;
    }

    void detach() {
        --this.mAttachCount;
    }

    void onAdapterChanged(RecyclerView.Adapter oldAdapter, RecyclerView.Adapter newAdapter, boolean compatibleWithPrevious) {
        if (oldAdapter != null) {
            this.detach();
        }

        if (!compatibleWithPrevious && this.mAttachCount == 0) {
            this.clear();
        }

        if (newAdapter != null) {
            this.attach();
        }

    }

    private RecyclerView.RecycledViewPool.ScrapData getScrapDataForType(int viewType) {
        RecyclerView.RecycledViewPool.ScrapData scrapData = (RecyclerView.RecycledViewPool.ScrapData)this.mScrap.get(viewType);
        if (scrapData == null) {
            scrapData = new RecyclerView.RecycledViewPool.ScrapData();
            this.mScrap.put(viewType, scrapData);
        }

        return scrapData;
    }

    static class ScrapData {
        final ArrayList<RecyclerView.ViewHolder> mScrapHeap = new ArrayList();
        int mMaxScrap = 5;
        long mCreateRunningAverageNs = 0L;
        long mBindRunningAverageNs = 0L;

        ScrapData() {
        }
    }
}
```

