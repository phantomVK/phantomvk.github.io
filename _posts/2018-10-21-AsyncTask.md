---
layout:     post
title:      "Android源码系列(15) -- AsyncTask"
date:       2018-10-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

#### 1.1 作用

AsyncTask令主线程的正确使用变得简单。无需维护线程或 __Handler__ ，即能让任务在后台线程运算，并把结果提交到主线程。

```java
public abstract class AsyncTask<Params, Progress, Result>
```

AsyncTask设计为围绕着 __Thread__ 和 __Handler__、无需构造普通线程框架的帮助类。合适执行(最多运算几秒的)短任务。如果任务导致线程长时间执行，强烈建议用由 __java.util.concurrent__ 包下 __Executor__、__ThreadPoolExecutor__ 和 __FutureTask__ 提供的APIs。

#### 1.2 组成

通过一个在后台线程执行的工作定义任务，执行结果发布到主线程。异步任务构成：
 - 3个类型： __Params__、 __Progress__、 __Result__ 
 - 4个步骤： __onPreExecute__、 __doInBackground__、 __onProgressUpdate__、 __onPostExecute__


AsyncTask由子类继承，至少重写方法 __doInBackground()__，通常也重写另一个方法 __onPostExecute()__。

用法示例：

```java
private class DownloadFilesTask extends AsyncTask(URL, Integer, Long) {
    protected Long doInBackground(URL... urls) {
        int count = urls.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(urls[i]);
            publishProgress((int) ((i / (float) count) * 100));
            // Escape early if cancel() is called
            if (isCancelled()) break;
        }
        return totalSize;
    }

    protected void onProgressUpdate(Integer... progress) {
        setProgressPercent(progress[0]);
    }

    protected void onPostExecute(Long result) {
        showDialog("Downloaded " + result + " bytes");
    }
}

// 启动创建完成的任务，用法非常简单:
new DownloadFilesTask().execute(url1, url2, url3);
```

#### 1.3 3个参数类型

有以下三个被异步任务使用的参数：
 - __Params：__ 送到任务执行参数的类型；
 - __Progress：__ 任务在后台计算时进度单元的类型，如Integer；
 - __Result：__ 后台计算结果返回类型；

 三个参数不需全部用上，不需要的用 __Void__ 代替。例如：

```java
private class MyTask extends AsyncTask<Void, Void, Void> { ... }
```

#### 1.4 4个方法

分别是 __onPreExecute__、 __doInBackground__ 、 __onProgressUpdate__、 __onPostExecute__

1. __onPreExecute()__ 在任务执行前于主线程调用，起配置任务的作用，如在界面上显示进度条；
2. 随后，在后台线程调用 __doInBackground__。本步骤用来执行很长时间的后台计算任务，参数在此步骤传递到异步任务。计算结果也在这步骤返给上一步骤。在计算子线程过程中，能通过方法 __publishProgress__ 传递进度到主线程；
3. 在子线程执行 __publishProgress__ 会触发主线程调用 __onProgressUpdate__，并向界面传递最新进度值；
4. 后台线程执行完毕后，计算结果作为参数在主线程传给方法 __onPostExecute__ ；

![AsyncTask_Execution](/img/android/images/AsyncTask_Execution.png)

#### 1.5 取消任务

任何时候都可通过 __cancel(boolean)__ 取消任务，方法会继续调起 __isCancelled()__ 并返回 __true__。

调用 __cancel(boolean)__ 后，__doInBackground(Object[])__返回后的下一个执行方法是 __onCancelled(Object)__，而不是 __onPostExecute(Object)__ 。

如果可以，为了保证任务能尽早被取消，需周期性地在 __doInBackground(Object[])__ 中检查 __isCancelled()__ 方法的返回值。

#### 1.6 线程规则

为保证类正常运行，有些线程规则需要遵守：

- __AsyncTask__ 必须在主线程载入。VERSION_CODES.JELLY_BEAN中此过程自动完成；
- 任务实例必须在主线程中创建；
- __execute__ 方法必须在主线程调用；
- 不得手动调用 __onPreExecute()__ 、__onPostExecute()__、 __doInBackground()__、 __onProgressUpdate()__ ；
- 每个任务仅能执行一次，任务尝试多次调用会抛出异常；

#### 1.7 内存可观察能力

__AsyncTask__ 保证所有回调通过以下安全、不需显式同步的方式调用：

- 在 __构造方法__ 或 __onPreExecute__ 设置成员变量，在 __doInBackground__ 引用；
- 在 __doInBackground__ 设置成员变量，在 __onProgressUpdate__ 、__onPostExecute__ 引用；

#### 1.8 执行的顺序

历史版本：

- 首次引入 __AsyncTasks__ 类时，任务在单个串行后台线程中执行；

- 从 __VERSION_CODES.DONUT__ 开始改为线程池，并在多线程同时并行执行；

- 因为避免并行计算导致错误，从 __VERSION_CODES.HONEYCOMB__ 始任务回到单后台线程执行；

如果确实需要并行执行，可以通过 __executeOnExecutor(java.util.concurrent.Executor, Object[])__ 达到使用 __THREAD_POOL_EXECUTOR__ 的目的。

#### 1.9 关于系统版本

本文介源码来自 __Android 28__ 。但不同历史版本源码实现方法差别非常大，也会出现不同结果。所以在实际运行过程中，务必关注运行时系统版本。

## 二、常量

#### 2.1 并行线程池

获取设备处理器核心数

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
```

线程池核心线程数最少2个线程、最多4个线程。在遵循此前提下，更倾向核心线程数比处理器实际核心数少1个，避免完全占用处理器而影其他后台任务的执行

```java
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
```

线程池最大线程数

```java
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
```

非核心线程存活时间

```java
private static final int KEEP_ALIVE_SECONDS = 30;
```

线程工厂，为并行任务的Executor构建线程

```java
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);  // 原子变量，从1开始递增

    public Thread newThread(Runnable r) { // 设置线程名称
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};
```

执行任务Executor的阻塞队列，队列长度128

```java
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);
```

并行任务Executor

```java
public static final Executor THREAD_POOL_EXECUTOR;
```

初始化并行线程池Executor

```java
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

#### 2.2 串行线程池

同一进程共用一个Executor顺序执行任务，任务每次执行一个

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

#### 2.3 消息类型

消息类型为任务执行结果

```java
private static final int MESSAGE_POST_RESULT = 0x1;
```

消息类型为主线程进度通知

```java
private static final int MESSAGE_POST_PROGRESS = 0x2;
```

## 三、数据成员

```java
// 顺序任务执行的Executor
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

// 保存InternalHandler，内部使用主线程Looper或自定义Looper
private static InternalHandler sHandler;

private final WorkerRunnable<Params, Result> mWorker;
private final FutureTask<Result> mFuture;

// 任务状态，任务构建后默认处于Status.PENDING
private volatile Status mStatus = Status.PENDING;

// 任务是否已被取消
private final AtomicBoolean mCancelled = new AtomicBoolean();

// 任务是否已被触发
private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

private final Handler mHandler;
```

## 四、SerialExecutor

```java
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                // 此任务运行完成后触发下一个任务执行
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        
        // 顺序执行线程池首次执行，mActive为空
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        // 从任务队列获取下一任务
        if ((mActive = mTasks.poll()) != null) {
            // 向并行执行线程池添加任务
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

## 五、状态枚举

任务的状态枚举，用于表示指定任务当前处理状态

```java
public enum Status {
    PENDING, // 任务正在尚未被执行，正在排队等待

    RUNNING, // 任务正在执行(运算)标志

    FINISHED, // 任务已经执行完毕：先调用onPostExecute()，再把状态置为此值
}
```

所有任务按照此生命周期单向前进，每个状态只允许设置一次。

![AsyncTask_Status](/img/android/images/AsyncTask_Status.png)


```java
// 获取主线程Handler
private static Handler getMainHandler() {
    // AsyncTask共用同一InternalHandler
    synchronized (AsyncTask.class) {
        // 初始化InternalHandler
        if (sHandler == null) {
            // 向InternalHandler传递主线程的Looper
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}

// 设置默认Executor
public static void setDefaultExecutor(Executor exec) {
    sDefaultExecutor = exec;
}
```

## 六、构造方法

构建新异步任务，所有构造方法必须在主线程调用

```java
public AsyncTask() {
    this((Looper) null);
}
```

通过指定Handler构建实例
```java
public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}
```
通过指定Looper构建实例

```java
public AsyncTask(@Nullable Looper callbackLooper) {
    // 若没有传入其他Looper，构造方法会自动获取主线程Looper，用获取的Looper创建Handler
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);

    // 构建WorkerRunnable
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            // 设置任务已被触发
            mTaskInvoked.set(true);
            // 任务执行结果
            Result result = null;
            try {
                // 设置线程优先级
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                // 把任务所需参数传入到后台线程执行并获取结果
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                // 任务执行出现异常，则取消任务
                mCancelled.set(true);
                throw tr;
            } finally {
                // 任务执行完成把结果传递给postResult
                postResult(result);
            }
            // 返回结果
            return result;
        }
    };

    // 把WorkerRunnable封装到FutureTask
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

## 七、成员方法

```java
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}
```

```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

```java
private Handler getHandler() {
    return mHandler;
}
```

```java
// 获取当前任务的执行状态
public final Status getStatus() {
    return mStatus;
}
```

重写方法实现后台线程的计算逻辑。任务调用者提供参数 __params__ 给 __execute()__，__execute()__ 传递给本方法。在方法内可调用 __publishProgress()__ 向主线程发布实时更新值

```java
@WorkerThread
protected abstract Result doInBackground(Params... params);
```

在 __doInBackground()__ 调用前，先在主线程执行此方法

```java
@MainThread
protected void onPreExecute() {
}
```

__doInBackground()__ 完成后在主线程调用此方法，__result__ 是 __doInBackground()__ 返回的结果。如果任务被取消，此方法不会触发

```java
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onPostExecute(Result result) {
}
```

__publishProgress()__ 运行后在主线程调用此方法，__value__ 表示任务处理的进度，参数 __values__ 是传递给__publishProgress()__ 的 __values__ 值

```java
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onProgressUpdate(Progress... values) {
}
```

__cancel(boolean)__ 被调用，且 __doInBackground(Object[])__ 结束后在主线程调用此方法。

方法默认简单回调 __onCancelled()__ 并忽略结果。如果要在子类重写其他实现，不要在重写方法内调用 __super.onCancelled(result)__。注意，__result__ 可为 __null__。

```java
@SuppressWarnings({"UnusedParameters"})
@MainThread
protected void onCancelled(Result result) {
    onCancelled();
}    
```

应用最好能重写本方法，以便在任务被取消后做出反应。此方法由 __onCancelled(Object)__ 的默认实现调用。 __cancel(boolean)__ 被调用且 __doInBackground(Object[])__ 已结束后，方法在主线程上调用。

```java
@MainThread
protected void onCancelled() {
}
```

若任务在正常完成前被取消，此方法返回true。

```java
public final boolean isCancelled() {
    return mCancelled.get();
}
```

尝试取消任务，如果任务已执行完毕、取消、由于其他原因不能取消的，则取消失败。任务取消成功，且在 __cancel()__ 调用是尚未开始，任务不会执行。

如果任务已经开始，参数 __mayInterruptIfRunning__ 决定任务执行线程是否该被中断。调用此方法，且 __doInBackground(Object[])__ 结束后，会在主线程调用 __onCancelled(Object)__。调用此方法能保证 __onPostExecute(Object)__ 不会执行。

此方法调用后，需从 __doInBackground(Object[])__ 周期性检查由 __isCancelled()__ 返回的值并尽快结束任务。参数 __mayInterruptIfRunning__ 为 __true__，执行任务线程需被中断，否则等任务执行直至完成。

```java
public final boolean cancel(boolean mayInterruptIfRunning) {
    mCancelled.set(true);
    return mFuture.cancel(mayInterruptIfRunning);
}
```

等待计算完毕后获取执行结果：

- 任务被取消后调用此方法，抛出CancellationException；

- 当前线程在等待结果过程被中断，抛出InterruptedException；

```java
public final Result get() throws InterruptedException, ExecutionException {
    return mFuture.get();
}
```

等待计算完毕后获取执行结果，设置timeout作为等待超时时间：

- 任务被取消后调用此方法，抛出CancellationException；

- 任务执行过程中出现异常，抛出ExecutionException；

- 当前线程等待结果过程中被中断，抛出InterruptedException；

- 等待结果超时，抛出TimeoutException；

```java
public final Result get(long timeout, TimeUnit unit) throws InterruptedException,
        ExecutionException, TimeoutException {
    return mFuture.get(timeout, unit);
}
```

用指定参数执行任务，任务返回自身以便调用者获取引用。功能在单个后台线程执行，或根据具体平台版本决定。

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

用指定参数执行任务，任务返回自身以便调用者获取引用。方法用 __THREAD_POOL_EXECUTOR__ 实现多任务并行处理，也可以用自定义Executor实现定制。并行执行不能保证任务运行顺序的先后，如果多个任务需有序执行，请使用顺序任务。

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    // 若任务默认状态不是PENDING状态，直接抛出异常
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                // 任务正在执行，不能再次开始任务
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                // 任务已经执行完毕，每个任务仅能被执行一次，不能再次开始
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

通过默认Executor执行单个Runnable

```java
@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}
```

__doInBackground()__ 在后台线程运行过程可(多次)调用此方法。每次调用方法都会在子线程向主线程发布更新进度的 __Message__，并在主线程触发 __onProgressUpdate()__。如果任务被取消，__onProgressUpdate__ 不会调用。

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

任务执行完成，根据完成状态执行对应分支逻辑

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
## 八、InternalHandler

```java
private static class InternalHandler extends Handler {
    // 构造AsyncTask实例时默认主线程Looper
    public InternalHandler(Looper looper) {
        super(looper);
    }

    // 消息通过handleMessage在主线程执行分发运行逻辑
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        // 从Message中获取异步任务执行结果
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        // 按照消息不同类型执行逻辑
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // 唯一结果传入finish()，内部再调用onPostExecute()
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                // 通知主线程更新进度
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

## 九、WorkerRunnable

WorkerRunnable实现Callable接口。相比Runnable接口，Callable会在运行成功完成后返回运行结果。此类是抽象类，子类需实现`call()`抽象方法

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

## 十、AsyncTaskResult

异步任务执行后结果

```java
@SuppressWarnings({"RawUseOfParameterizedType"})
private static class AsyncTaskResult<Data> {
    final AsyncTask mTask;
    final Data[] mData;

    AsyncTaskResult(AsyncTask task, Data... data) {
        mTask = task;
        mData = data;
    }
}
```
