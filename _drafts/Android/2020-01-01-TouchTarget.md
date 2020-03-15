---
layout:     post
title:      "TouchTarget"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---

Android29 ViewGroup.TouchTarget

```java
/* Describes a touched view and the ids of the pointers that it has captured.
 *
 * This code assumes that pointer ids are always in the range 0..31 such that
 * it can use a bitfield to track which pointer ids are present.
 * As it happens, the lower layers of the input dispatch pipeline also use the
 * same trick so the assumption should be safe here...
 */
private static final class TouchTarget {
    private static final int MAX_RECYCLED = 32;
    private static final Object sRecycleLock = new Object[0];
    private static TouchTarget sRecycleBin;
    private static int sRecycledCount;

    public static final int ALL_POINTER_IDS = -1; // all ones

    // The touched child view.
    @UnsupportedAppUsage
    public View child;

    // The combined bit mask of pointer ids for all pointers captured by the target.
    public int pointerIdBits;

    // The next target in the target list.
    public TouchTarget next;

    @UnsupportedAppUsage
    private TouchTarget() {
    }

    public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
        if (child == null) {
            throw new IllegalArgumentException("child must be non-null");
        }

        final TouchTarget target;
        synchronized (sRecycleLock) {
            if (sRecycleBin == null) {
                target = new TouchTarget();
            } else {
                target = sRecycleBin;
                sRecycleBin = target.next;
                 sRecycledCount--;
                target.next = null;
            }
        }
        target.child = child;
        target.pointerIdBits = pointerIdBits;
        return target;
    }

    public void recycle() {
        if (child == null) {
            throw new IllegalStateException("already recycled once");
        }

        synchronized (sRecycleLock) {
            if (sRecycledCount < MAX_RECYCLED) {
                next = sRecycleBin;
                sRecycleBin = this;
                sRecycledCount += 1;
            } else {
                next = null;
            }
            child = null;
        }
    }
}
```

