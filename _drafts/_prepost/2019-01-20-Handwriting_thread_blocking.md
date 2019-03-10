---
layout:     post
title:      "手写线程阻塞"
date:       2019-03-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

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
