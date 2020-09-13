---
layout:     post
title:      "手写Java线程死锁"
date:       2019-04-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

如果运行结果没有同时出现 __o1没出现死锁__ 和 __o2没出现死锁__，表示两个线程已死锁

```java
public class DeadLockClass {
    public static void main(String[] args) {
        new Thread(new DeadLock(true)).start();
        new Thread(new DeadLock(false)).start();
    }

    private static class DeadLock implements Runnable {
        private boolean mFlag;
        private static final Object LOCK_0 = new Object();
        private static final Object LOCK_1 = new Object();

        public DeadLock(boolean flag) { mFlag = flag; }

        public void run() {
            synchronized (mFlag ? LOCK_0 : LOCK_1) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ignore) {
                }

                synchronized (mFlag ? LOCK_1 : LOCK_0) {
                    System.out.println("o1没出现死锁");
                }
            }
        }
    }
}
```
