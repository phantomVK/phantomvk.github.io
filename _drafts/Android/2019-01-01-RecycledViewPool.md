---
layout:     post
title:      "RecycledViewPool"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

Android 28

__RecycledViewPool__ 是 __RecyclerView__ 的静态内部类

```java
/**
 * RecycledViewPool lets you share Views between multiple RecyclerViews.
 * <p>
 * If you want to recycle views across RecyclerViews, create an instance of RecycledViewPool
 * and use {@link RecyclerView#setRecycledViewPool(RecycledViewPool)}.
 * <p>
 * RecyclerView automatically creates a pool for itself if you don't provide one.
 *
 */
public static class RecycledViewPool
```

```java
private static final int DEFAULT_MAX_SCRAP = 5;
```

```java
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
    ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
    int mMaxScrap = DEFAULT_MAX_SCRAP;
    long mCreateRunningAverageNs = 0;
    long mBindRunningAverageNs = 0;
}
```

```java
SparseArray<ScrapData> mScrap = new SparseArray<>();

private int mAttachCount = 0;

public void clear() {
    for (int i = 0; i < mScrap.size(); i++) {
        ScrapData data = mScrap.valueAt(i);
        data.mScrapHeap.clear();
    }
}

public void setMaxRecycledViews(int viewType, int max) {
    ScrapData scrapData = getScrapDataForType(viewType);
    scrapData.mMaxScrap = max;
    final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
    if (scrapHeap != null) {
        while (scrapHeap.size() > max) {
            scrapHeap.remove(scrapHeap.size() - 1);
        }
    }
}

/**
 * Returns the current number of Views held by the RecycledViewPool of the given view type.
 */
public int getRecycledViewCount(int viewType) {
    return getScrapDataForType(viewType).mScrapHeap.size();
}

public ViewHolder getRecycledView(int viewType) {
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        return scrapHeap.remove(scrapHeap.size() - 1);
    }
    return null;
}

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

public void putRecycledView(ViewHolder scrap) {
    final int viewType = scrap.getItemViewType();
    final ArrayList scrapHeap = getScrapDataForType(viewType).mScrapHeap;
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
```

