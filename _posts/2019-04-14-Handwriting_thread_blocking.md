---
layout:     post
title:      "手写Java线程阻塞"
date:       2019-04-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Programming Language
---

如果运行结果没有同时出现 __o1没出现死锁__ 和 __o2没出现死锁__，表示两个线程已死锁

```java
public class DeadLockClass {
    public static void main(String[] args) {
        Thread t1 = new Thread(new DeadLock(true));
        Thread t2 = new Thread(new DeadLock(false));
        t1.start();
        t2.start();
    }

    private static class DeadLock implements Runnable {
        private boolean mFlag;
        private static final Object OBJ_1 = new Object();
        private static final Object OBJ_2 = new Object();

        DeadLock(boolean flag) {
            mFlag = flag;
        }

        public void run() {
            if (mFlag) {
                synchronized (OBJ_1) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException ignore) {
                    }

                    synchronized (OBJ_2) {
                        System.out.println("o1没出现死锁");
                    }
                }
            } else {
                synchronized (OBJ_2) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException ignore) {
                    }

                    synchronized (OBJ_1) {
                        System.out.println("o2没出现死锁");
                    }
                }
            }
        }
    }
}
```
