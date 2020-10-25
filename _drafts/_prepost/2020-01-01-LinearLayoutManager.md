---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---

```java
public class LinearLayoutManager extends RecyclerView.LayoutManager implements
        ItemTouchHelper.ViewDropHandler, RecyclerView.SmoothScroller.ScrollVectorProvider
```



```java
/**
 * The magic functions :). Fills the given layout, defined by the layoutState. This is fairly
 * independent from the rest of the {@link LinearLayoutManager}
 * and with little change, can be made publicly available as a helper class.
 *
 * @param recycler        Current recycler that is attached to RecyclerView
 * @param layoutState     Configuration on how we should fill out the available space.
 * @param state           Context passed by the RecyclerView to control scroll steps.
 * @param stopOnFocusable If true, filling stops in the first focusable new child
 * @return Number of pixels that it added. Useful for scroll functions.
 */
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // max offset we should set is mFastScroll + available
    final int start = layoutState.mAvailable;
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        // TODO ugly bug fix. should not happen
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunkResult.resetInternal();
        if (RecyclerView.VERBOSE_TRACING) {
            TraceCompat.beginSection("LLM LayoutChunk");
        }
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        if (RecyclerView.VERBOSE_TRACING) {
            TraceCompat.endSection();
        }
        if (layoutChunkResult.mFinished) {
            break;
        }
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
        /**
         * Consume the available space if:
         * * layoutChunk did not request to be ignored
         * * OR we are laying out scrap children
         * * OR we are not doing pre-layout
         */
        if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                || !state.isPreLayout()) {
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            // we keep a separate remaining space because mAvailable is important for recycling
            remainingSpace -= layoutChunkResult.mConsumed;
        }

        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            recycleByLayoutState(recycler, layoutState);
        }
        if (stopOnFocusable && layoutChunkResult.mFocusable) {
            break;
        }
    }
    if (DEBUG) {
        validateChildOrder();
    }
    return start - layoutState.mAvailable;
}

void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    if (view == null) {
        if (DEBUG && layoutState.mScrapList == null) {
            throw new RuntimeException("received null view when unexpected");
        }
        // if we are laying out views in scrap, this may return null which means there is
        // no more items to layout.
        result.mFinished = true;
        return;
    }
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addDisappearingView(view);
        } else {
            addDisappearingView(view, 0);
        }
    }
    measureChildWithMargins(view, 0, 0);
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    int left, top, right, bottom;
    if (mOrientation == VERTICAL) {
        if (isLayoutRTL()) {
            right = getWidth() - getPaddingRight();
            left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
        } else {
            left = getPaddingLeft();
            right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            bottom = layoutState.mOffset;
            top = layoutState.mOffset - result.mConsumed;
        } else {
            top = layoutState.mOffset;
            bottom = layoutState.mOffset + result.mConsumed;
        }
    } else {
        top = getPaddingTop();
        bottom = top + mOrientationHelper.getDecoratedMeasurementInOther(view);

        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            right = layoutState.mOffset;
            left = layoutState.mOffset - result.mConsumed;
        } else {
            left = layoutState.mOffset;
            right = layoutState.mOffset + result.mConsumed;
        }
    }
    // We calculate everything with View's bounding box (which includes decor and margins)
    // To calculate correct layout position, we subtract margins.
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    if (DEBUG) {
        Log.d(TAG, "laid out child at position " + getPosition(view) + ", with l:"
                + (left + params.leftMargin) + ", t:" + (top + params.topMargin) + ", r:"
                + (right - params.rightMargin) + ", b:" + (bottom - params.bottomMargin));
    }
    // Consume the available space if the view is not removed OR changed
    if (params.isItemRemoved() || params.isItemChanged()) {
        result.mIgnoreConsumed = true;
    }
    result.mFocusable = view.hasFocusable();
}
```

