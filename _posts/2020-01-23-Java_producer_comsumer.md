---
layout:     post
title:      "Java手写生产者消费者"
date:       2020-01-23
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

### BlockingQueue

```java
public class Main {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(5);
        new Thread(new Producer(queue), "Producer1").start();
        new Thread(new Producer(queue), "Producer2").start();
        new Thread(new Consumer(queue), "Consumer1").start();
    }
}

class Producer implements Runnable {
    private final BlockingQueue<String> queue;

    public Producer(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        final String name = Thread.currentThread().getName();
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(name + " produces message " + i);
                queue.put("Message " + i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class Consumer implements Runnable {
    private final BlockingQueue<String> queue;

    public Consumer(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        final String name = Thread.currentThread().getName();
        while (true) {
            try {
                System.out.println(name + " consumes " + queue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```



### Synchronized-wait-notifyAll

```java
public class Main {
    private static final int THREAD_COUNT = 2;
    private static final int THREAD_PRODUCTS = 5;
    private static final Buffer<Integer> buffer = new Buffer<>(3);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(() -> {
                for (int j = 0; j < THREAD_PRODUCTS; j++) {
                    try {
                        buffer.add(j);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }

        new Thread(() -> {
            while (true) {
                try {
                    System.out.println("消费数值：" + buffer.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

class Buffer<V> {
    private final List<V> list = new LinkedList<>();
    private final int size;

    public Buffer(int size) {
        this.size = size;
    }

    public void add(V v) throws InterruptedException {
        synchronized (this) {
            while (list.size() == size) {
                wait();
            }

            list.add(v);
            notifyAll();
        }
    }

    public V take() throws InterruptedException {
        synchronized (this) {
            while (list.size() == 0) {
                wait();
            }

            V v = list.remove(0);
            notifyAll();
            return v;
        }
    }
}
```



### ReentrantLock-Condition

```java
public class Main {
    private static final int THREAD_COUNT = 2;
    private static final int THREAD_PRODUCTS = 5;
    private static final Buffer<Integer> buffer = new Buffer<>(2);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(() -> {
                for (int j = 0; j < THREAD_PRODUCTS; j++) {
                    try {
                        buffer.add(j);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }

        new Thread(() -> {
            while (true) {
                try {
                    System.out.println("消费数值：" + buffer.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

class Buffer<V> {
    private final List<V> list = new LinkedList<V>();
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final int size;

    public Buffer(int size) {
        this.size = size;
    }

    public void add(V v) throws InterruptedException {
        lock.lock();
        try {
            while (list.size() == size) {
                notFull.await();
            }

            list.add(v);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public V take() throws InterruptedException {
        lock.lock();
        try {
            while (list.isEmpty()) {
                notEmpty.await();
            }

            V v = list.remove(0);
            notFull.signal();
            return v;
        } finally {
            lock.unlock();
        }
    }
}
```



### Semaphore

```java
public class Main {
    private static final int THREAD_COUNT = 2;
    private static final int THREAD_PRODUCTS = 5;
    private static final Buffer<Integer> buffer = new Buffer<>(2);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(() -> {
                for (int j = 0; j < THREAD_PRODUCTS; j++) {
                    try {
                        buffer.add(j);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }

        new Thread(() -> {
            while (true) {
                try {
                    System.out.println("消费数值：" + buffer.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

class Buffer<V> {
    private final List<V> list = new LinkedList<>();
    private final Semaphore mutex = new Semaphore(1);
    private final Semaphore notEmpty = new Semaphore(0);
    private final Semaphore notFull;

    public Buffer(int size) {
        notFull = new Semaphore(size);
    }

    public void add(V v) throws InterruptedException {
        notFull.acquire();

        try {
            mutex.acquire();
            list.add(v);
        } catch (InterruptedException e) {
            notFull.release();
        } finally {
            mutex.release();
            notEmpty.release();
        }
    }

    public V take() throws InterruptedException {
        notEmpty.acquire();

        try {
            mutex.acquire();
            return list.remove(0);
        } catch (InterruptedException e) {
            notEmpty.release();
            throw e;
        } finally {
            mutex.release();
            notFull.release();
        }
    }
}
```

