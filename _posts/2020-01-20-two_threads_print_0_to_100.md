---
layout:     post
title:      "两个线程交替打印0到100"
date:       2020-01-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

### synchronized + MONITOR

两个线程都在同一个 **MONITOR** 上等待和唤醒。如果只要求两个线程轮流打印，则只需调用 **MONITOR.notify();** 唤醒另一个线程。

```java
public class Main {
    public static final Object MONITOR = new Object();
    public static volatile int count = 0;

    private static final Runnable task = () -> {
        String name = Thread.currentThread().getName();
        while (true) {
            synchronized (MONITOR) {
                // 本线程退出时要唤醒另一个线程
                // 否则另一个线程等不到唤醒且阻塞无法退出
                if (count > 100) {
                    MONITOR.notify();
                    return;
                }

                System.out.println(name + ": " + count++);
                MONITOR.notify();

                try {
                    MONITOR.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    };

    public static void main(String[] args) {
        new Thread(task, "奇数线程").start();
        new Thread(task, "偶数线程").start();
    }
}
```

运行效果：

```
奇数线程: 0
偶数线程: 1
奇数线程: 2
偶数线程: 3
.....
奇数线程: 98
偶数线程: 99
奇数线程: 100

Process finished with exit code 0
```



### Flag + while

此方法没有使用同步，而是使用两个 **volatile** 修饰的布尔值作为信号量。

执行流程：

- 启动时 **Odd线程** 和 **Even线程** 都直接开始忙循环，等待 **Main线程** 发出信号；
- **Main线程** 修改信号通知两个子线程，让其一离开忙循环。示例让 **Odd线程** 离开循环先起步；
- **Odd线程** 离开循环输出结果，修改信号量通知 **Even线程** 离开循环，自己开始忙循环；
- **Even线程** 离开循环输出结果，修改信号量通知 **Odd线程** 离开循环，自己开始忙循环；
- 两个线程交替执行直到结束；

```java
public class Main {
    // 两个线程启动后都会进入循环，都需要使用volatile修饰
    private static volatile boolean loopForOdd = true;
    private static volatile boolean loopForEven = true;
    private static volatile int counter = 1;

    public static void main(String[] args) {
        // Odd线程，即奇数线程
        new Thread(() -> {
            while (counter < 100) {
                while (loopForOdd) Thread.onSpinWait();
                System.out.println("奇数线程：" + counter++);
                loopForEven = false;
                loopForOdd = true;
            }
        }).start();

        // Even线程，即偶数线程
        new Thread(() -> {
            while (counter < 100) {
                while (loopForEven) Thread.onSpinWait();
                System.out.println("偶数线程：" + counter++);
                loopForOdd = false;
                loopForEven = true;
            }
        }).start();

        // 先启动奇数线程
        loopForOdd = false;
    }
}
```

运行效果：

```
奇数线程: 0
偶数线程: 1
奇数线程: 2
偶数线程: 3
.....
奇数线程: 98
偶数线程: 99
奇数线程: 100

Process finished with exit code 0
```



### ReentrantLock + Condition

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new Task(0));
        Thread t2 = new Thread(new Task(1));
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}

class Task implements Runnable {
    private static final ReentrantLock lock = new ReentrantLock();
    private static final Condition condition = lock.newCondition();
    private static volatile int count = 0;
    private final int taskName;

    public Task(int taskName) {
        this.taskName = taskName;
    }

    @Override
    public void run() {
        while (true) {
            lock.lock();

            if (count > 100) {
                lock.unlock();
                break;
            }

            if (count % 2 == taskName) {
                System.out.println(taskName + ": " + (count++));
            } else {
                try {
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            condition.signal();
            lock.unlock();
        }
    }
}
```

运行效果：

```
0: 0
1: 1
0: 2
1: 3
....
0: 98
1: 99
0: 100

Process finished with exit code 0
```