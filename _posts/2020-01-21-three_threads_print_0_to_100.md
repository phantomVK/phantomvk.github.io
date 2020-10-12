---
layout:     post
title:      "三个线程交替打印0到100"
date:       2020-01-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

### ReentrantLock + Condition

多个线程抢夺控制权，检查自己是否符合执行条件，若条件满足则执行输出，否则让出控制权唤醒其他线程。

这种方案有明显缺点：因 __ReentrantLock__ 默认非公平锁，若多个线程抢夺时间片，热点线程有机会更频繁取得执行权。其次，被唤醒的线程不一定是符合执行条件的线程，所以会再次挂起，造成此次唤醒浪费的开销。

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new Task(0));
        Thread t2 = new Thread(new Task(1));
        Thread t3 = new Thread(new Task(2));

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}

class Task implements Runnable {
    private static final ReentrantLock lock = new ReentrantLock();
    private static final Condition condition = lock.newCondition();
    private final int threadNum;
    private static int count;

    public Task(int num) {
        threadNum = num;
    }

    @Override
    public void run() {
        while (true) {
            lock.lock();
            if (count > 100) {
                lock.unlock();
                break;
            }

            if (count % 3 == threadNum) {
                System.out.println(threadNum + "-->" + count);
                count++;
            } else {
                try {
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            condition.signalAll();
            lock.unlock();
        }
    }
}
```

### Semaphore

信号量可避免上述方案的问题，每次本线程执行完毕，定向唤醒下一个符合条件的线程，保证执行顺序，且互斥锁允许一个线程释放另一个线程持有的锁。

```java
public class Main {
    private static final int THREAD_COUNT = 3;
    private static volatile int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Semaphore[] semaphores = new Semaphore[THREAD_COUNT];
        for (int i = 0; i < THREAD_COUNT; i++) {
            semaphores[i] = new Semaphore(1);

            // 只有最后一个Semaphore可用，因此线程0先执行
            if (i != THREAD_COUNT - 1) {
                semaphores[i].acquire();
            }
        }

        for (int i = 0; i < THREAD_COUNT; i++) {
            final int index = i;
            new Thread(new Runnable() {
                final Semaphore prevSemaphore = semaphores[index == 0 ? THREAD_COUNT - 1 : index - 1];
                final Semaphore currSemaphore = semaphores[index];

                @Override
                public void run() {
                    final String threadName = Thread.currentThread().getName();

                    while (true) {
                        try {
                            prevSemaphore.acquire();
                            if (count > 100) return;
                            System.out.println(threadName + count++);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } finally {
                            currSemaphore.release();
                        }
                    }
                }
            }, "线程" + i + ": ").start();
        }
    }
}
```

