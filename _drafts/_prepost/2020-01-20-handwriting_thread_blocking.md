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
class DeadLock implements Runnable {
    private boolean flag;
    private static final Object o1 = new Object();
    private static final Object o2 = new Object();

    DeadLock(boolean flag) {
        this.flag = flag;
    }

    public void run() {
        if (flag) {
            synchronized (o1) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ignore) {
                }

                synchronized (o2) {
                    System.out.println("o1没出现死锁");
                }
            }
        } else {
            synchronized (o2) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ignore) {
                }

                synchronized (o1) {
                    System.out.println("o2没出现死锁");
                }
            }
        }
    }
}

public class DeadLockClass {
    public static void main(String[] args) {
        Thread t1 = new Thread(new DeadLock(true));
        Thread t2 = new Thread(new DeadLock(false));
        t1.start();
        t2.start();
    }
}
```