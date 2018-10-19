---
layout:     post
title:      "Android源码系列 -- AsyncTask"
date:       2018-10-16
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

Android 28

## 一、类签名

```java
/**
 * <p>AsyncTask enables proper and easy use of the UI thread. This class allows you
 * to perform background operations and publish results on the UI thread without
 * having to manipulate threads and/or handlers.</p>
 *
 * <p>AsyncTask is designed to be a helper class around {@link Thread} and {@link Handler}
 * and does not constitute a generic threading framework. AsyncTasks should ideally be
 * used for short operations (a few seconds at the most.) If you need to keep threads
 * running for long periods of time, it is highly recommended you use the various APIs
 * provided by the <code>java.util.concurrent</code> package such as {@link Executor},
 * {@link ThreadPoolExecutor} and {@link FutureTask}.</p>
 *
 * <p>An asynchronous task is defined by a computation that runs on a background thread and
 * whose result is published on the UI thread. An asynchronous task is defined by 3 generic
 * types, called <code>Params</code>, <code>Progress</code> and <code>Result</code>,
 * and 4 steps, called <code>onPreExecute</code>, <code>doInBackground</code>,
 * <code>onProgressUpdate</code> and <code>onPostExecute</code>.</p>
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 * <p>For more information about using tasks and threads, read the
 * <a href="{@docRoot}guide/components/processes-and-threads.html">Processes and
 * Threads</a> developer guide.</p>
 * </div>
 *
 * <h2>Usage</h2>
 * <p>AsyncTask must be subclassed to be used. The subclass will override at least
 * one method ({@link #doInBackground}), and most often will override a
 * second one ({@link #onPostExecute}.)</p>
 *
 * <p>Here is an example of subclassing:</p>
 * <pre class="prettyprint">
 * private class DownloadFilesTask extends AsyncTask&lt;URL, Integer, Long&gt; {
 *     protected Long doInBackground(URL... urls) {
 *         int count = urls.length;
 *         long totalSize = 0;
 *         for (int i = 0; i < count; i++) {
 *             totalSize += Downloader.downloadFile(urls[i]);
 *             publishProgress((int) ((i / (float) count) * 100));
 *             // Escape early if cancel() is called
 *             if (isCancelled()) break;
 *         }
 *         return totalSize;
 *     }
 *
 *     protected void onProgressUpdate(Integer... progress) {
 *         setProgressPercent(progress[0]);
 *     }
 *
 *     protected void onPostExecute(Long result) {
 *         showDialog("Downloaded " + result + " bytes");
 *     }
 * }
 * </pre>
 *
 * <p>Once created, a task is executed very simply:</p>
 * <pre class="prettyprint">
 * new DownloadFilesTask().execute(url1, url2, url3);
 * </pre>
 *
 * <h2>AsyncTask's generic types</h2>
 * <p>The three types used by an asynchronous task are the following:</p>
 * <ol>
 *     <li><code>Params</code>, the type of the parameters sent to the task upon
 *     execution.</li>
 *     <li><code>Progress</code>, the type of the progress units published during
 *     the background computation.</li>
 *     <li><code>Result</code>, the type of the result of the background
 *     computation.</li>
 * </ol>
 * <p>Not all types are always used by an asynchronous task. To mark a type as unused,
 * simply use the type {@link Void}:</p>
 * <pre>
 * private class MyTask extends AsyncTask&lt;Void, Void, Void&gt; { ... }
 * </pre>
 *
 * <h2>The 4 steps</h2>
 * <p>When an asynchronous task is executed, the task goes through 4 steps:</p>
 * <ol>
 *     <li>{@link #onPreExecute()}, invoked on the UI thread before the task
 *     is executed. This step is normally used to setup the task, for instance by
 *     showing a progress bar in the user interface.</li>
 *     <li>{@link #doInBackground}, invoked on the background thread
 *     immediately after {@link #onPreExecute()} finishes executing. This step is used
 *     to perform background computation that can take a long time. The parameters
 *     of the asynchronous task are passed to this step. The result of the computation must
 *     be returned by this step and will be passed back to the last step. This step
 *     can also use {@link #publishProgress} to publish one or more units
 *     of progress. These values are published on the UI thread, in the
 *     {@link #onProgressUpdate} step.</li>
 *     <li>{@link #onProgressUpdate}, invoked on the UI thread after a
 *     call to {@link #publishProgress}. The timing of the execution is
 *     undefined. This method is used to display any form of progress in the user
 *     interface while the background computation is still executing. For instance,
 *     it can be used to animate a progress bar or show logs in a text field.</li>
 *     <li>{@link #onPostExecute}, invoked on the UI thread after the background
 *     computation finishes. The result of the background computation is passed to
 *     this step as a parameter.</li>
 * </ol>
 * 
 * <h2>Cancelling a task</h2>
 * <p>A task can be cancelled at any time by invoking {@link #cancel(boolean)}. Invoking
 * this method will cause subsequent calls to {@link #isCancelled()} to return true.
 * After invoking this method, {@link #onCancelled(Object)}, instead of
 * {@link #onPostExecute(Object)} will be invoked after {@link #doInBackground(Object[])}
 * returns. To ensure that a task is cancelled as quickly as possible, you should always
 * check the return value of {@link #isCancelled()} periodically from
 * {@link #doInBackground(Object[])}, if possible (inside a loop for instance.)</p>
 *
 * <h2>Threading rules</h2>
 * <p>There are a few threading rules that must be followed for this class to
 * work properly:</p>
 * <ul>
 *     <li>The AsyncTask class must be loaded on the UI thread. This is done
 *     automatically as of {@link android.os.Build.VERSION_CODES#JELLY_BEAN}.</li>
 *     <li>The task instance must be created on the UI thread.</li>
 *     <li>{@link #execute} must be invoked on the UI thread.</li>
 *     <li>Do not call {@link #onPreExecute()}, {@link #onPostExecute},
 *     {@link #doInBackground}, {@link #onProgressUpdate} manually.</li>
 *     <li>The task can be executed only once (an exception will be thrown if
 *     a second execution is attempted.)</li>
 * </ul>
 *
 * <h2>Memory observability</h2>
 * <p>AsyncTask guarantees that all callback calls are synchronized in such a way that the following
 * operations are safe without explicit synchronizations.</p>
 * <ul>
 *     <li>Set member fields in the constructor or {@link #onPreExecute}, and refer to them
 *     in {@link #doInBackground}.
 *     <li>Set member fields in {@link #doInBackground}, and refer to them in
 *     {@link #onProgressUpdate} and {@link #onPostExecute}.
 * </ul>
 *
 * <h2>Order of execution</h2>
 * <p>When first introduced, AsyncTasks were executed serially on a single background
 * thread. Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
 * to a pool of threads allowing multiple tasks to operate in parallel. Starting with
 * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are executed on a single
 * thread to avoid common application errors caused by parallel execution.</p>
 * <p>If you truly want parallel execution, you can invoke
 * {@link #executeOnExecutor(java.util.concurrent.Executor, Object[])} with
 * {@link #THREAD_POOL_EXECUTOR}.</p>
 */
public abstract class AsyncTask<Params, Progress, Result>
```

## 二、常量

#### 2.1 并行线程池

```java
// 获取设备处理器核心数
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

// 线程池核心线程数最少2个线程，最多4个线程，在遵循此条件前提下更倾向核心
// 线程数比处理器实际核心数少1个，避免影响处理器其他后台任务的执行
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));

// 线程池最大线程数
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;

// 非核心线程存活时间
private static final int KEEP_ALIVE_SECONDS = 30;

// 线程工厂，为并行执行任务的Executor构建线程
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        // 设置线程名称，按照线程历史总数递增
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

// 执行任务的Executor的阻塞队列，队列长度128
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

// 并行执行任务的Executor
public static final Executor THREAD_POOL_EXECUTOR;

// 初始化并行执行任务的Executor
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

#### 2.2 串行线程池

```java
// 同一进程共用一个Executor顺序执行任务，任务每次执行一个
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
````

#### 2.3 消息类型

```java
// 消息类型为任务执行结果
private static final int MESSAGE_POST_RESULT = 0x1;

// 消息类型为主线程进度通知
private static final int MESSAGE_POST_PROGRESS = 0x2;
````

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
    // 任务正在尚未被执行，正在排队等待
    PENDING,

    // 任务正在执行(运算)标志
    RUNNING,

    // 任务已经执行完毕：先调用onPostExecute()，再把状态置为此值
    FINISHED,
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

```java
// 构建新异步任务，此构造方法必须在主线程调用
public AsyncTask() {
    this((Looper) null);
}

// 构建新异步任务，此构造方法必须在主线程调用
public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}

// 构建新异步任务，此构造方法必须在主线程调用
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

private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}

private Handler getHandler() {
    return mHandler;
}

// 获取当前任务的执行状态
public final Status getStatus() {
    return mStatus;
}

// 重写方法实现在后台线程中的计算逻辑
// 参数params就是传给execute()的参数，execute()的参数由任务调用者提供
// 在此方法内可调用publishProgress()，会自动在主线程上发布实时更新值
@WorkerThread
protected abstract Result doInBackground(Params... params);

// doInBackground调用前在主线程执行
@MainThread
protected void onPreExecute() {
}

// doInBackground()完成后会在主线程调用此方法，result是doInBackground()的返回结果
// 如果任务被取消了，此方法不会触发
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onPostExecute(Result result) {
}

// publishProgress()调用后会在主线程调用此方法，value表示任务处理的进度
// 此方法参数values即是传递给publishProgress()的values
@SuppressWarnings({"UnusedDeclaration"})
@MainThread
protected void onProgressUpdate(Progress... values) {
}

// cancel(boolean)被调用，且doInBackground(Object[])已结束后，会在主线程调用此方法
// 此方法默认只是简单回调onCancelled()并忽略结果
// 如果需在子类重写其他实现，无需在重写方法内调用super.onCancelled(result)，result可为null
@SuppressWarnings({"UnusedParameters"})
@MainThread
protected void onCancelled(Result result) {
    onCancelled();
}    

// 应用最好能重写方法，以便在任务被取消后做出反应
// 此方法由onCancelled(Object)的默认实现调用
// cancel(boolean)被调用且doInBackground(Object[])已结束后，方法在主线程上调用
@MainThread
protected void onCancelled() {
}

/**
 * Returns <tt>true</tt> if this task was cancelled before it completed
 * normally. If you are calling {@link #cancel(boolean)} on the task,
 * the value returned by this method should be checked periodically from
 * {@link #doInBackground(Object[])} to end the task as soon as possible.
 *
 * @return <tt>true</tt> if task was cancelled before it completed
 *
 * @see #cancel(boolean)
 */
// 若任务在正常完成前被取消，此方法返回true
public final boolean isCancelled() {
    return mCancelled.get();
}

/**
 * If successful,
 * and this task has not started when <tt>cancel</tt> is called,
 * this task should never run. If the task has already started,
 * then the <tt>mayInterruptIfRunning</tt> parameter determines
 * whether the thread executing this task should be interrupted in
 * an attempt to stop the task.</p>
 * 
 * <p>Calling this method will result in {@link #onCancelled(Object)} being
 * invoked on the UI thread after {@link #doInBackground(Object[])}
 * returns. Calling this method guarantees that {@link #onPostExecute(Object)}
 * is never invoked. After invoking this method, you should check the
 * value returned by {@link #isCancelled()} periodically from
 * {@link #doInBackground(Object[])} to finish the task as early as
 * possible.</p>
 *
 * @param mayInterruptIfRunning <tt>true</tt> if the thread executing this
 *        task should be interrupted; otherwise, in-progress tasks are allowed
 *        to complete.
 *
 * @return <tt>false</tt> if the task could not be cancelled,
 *         typically because it has already completed normally;
 *         <tt>true</tt> otherwise
 *
 * @see #isCancelled()
 * @see #onCancelled(Object)
 */
// 尝试取消此任务，若任务已执行完毕、取消、由于其他原因不能取消，则取消失败
// 任务取消成功，且在cancel()调用是尚未开始，任务不会执行
// 如果任务已经开始，参数mayInterruptIfRunning决定任务执行线程是否该被中断
// 调用此方法，且doInBackground(Object[])结束后，会在主线程调用onCancelled(Object)
// 调用此方法能保证onPostExecute(Object)永远不会执行
// 此方法调用后，需从doInBackground(Object[])周期性检查由isCancelled()返回的值以尽快结束任务
// 参数mayInterruptIfRunning为true，执行任务的线程应该被中断，否则允许任务执行直至完成
// 方法返回false便是任务违背取消，也就是任务已经完全正常执行
public final boolean cancel(boolean mayInterruptIfRunning) {
    mCancelled.set(true);
    return mFuture.cancel(mayInterruptIfRunning);
}

// 等待计算完毕后获取执行结果
// 任务被取消后调用此方法，抛出CancellationException
// 当前线程在等待结果过程被中断，抛出InterruptedException
public final Result get() throws InterruptedException, ExecutionException {
    return mFuture.get();
}

// 等待计算完毕后获取执行结果，设置timeout作为等待超时时间
// 任务被取消后调用此方法，抛出CancellationException
// 任务执行过程中出现异常，抛出ExecutionException
// 当前线程在等待结果过程被中断，抛出InterruptedException
// 等待结果超时，抛出TimeoutException
public final Result get(long timeout, TimeUnit unit) throws InterruptedException,
        ExecutionException, TimeoutException {
    return mFuture.get(timeout, unit);
}

/**
 * Executes the task with the specified parameters. The task returns
 * itself (this) so that the caller can keep a reference to it.
 * 
 * <p>Note: this function schedules the task on a queue for a single background
 * thread or pool of threads depending on the platform version.  When first
 * introduced, AsyncTasks were executed serially on a single background thread.
 * Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
 * to a pool of threads allowing multiple tasks to operate in parallel. Starting
 * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are back to being
 * executed on a single thread to avoid common application errors caused
 * by parallel execution.  If you truly want parallel execution, you can use
 * the {@link #executeOnExecutor} version of this method
 * with {@link #THREAD_POOL_EXECUTOR}; however, see commentary there for warnings
 * on its use.
 *
 * <p>This method must be invoked on the UI thread.
 *
 * @param params The parameters of the task.
 *
 * @return This instance of AsyncTask.
 *
 * @throws IllegalStateException If {@link #getStatus()} returns either
 *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
 *
 * @see #executeOnExecutor(java.util.concurrent.Executor, Object[])
 * @see #execute(Runnable)
 */
// 用指定参数执行任务，任务返回自身以便调用者获取引用
// 此功能在单个后台线程执行，或根据具体平台版本决定
// AsyncTask刚出来时，任务在一个顺序单后台线程执行；
// 从VERSION_CODES#DONUT开始改为线程池以允许任务并行执行；
// 从VERSION_CODES#HONEYCOMB回到单后台线程方式执行，已避免多线程并行执行错误；
// 如果明确需要并行执行，可以是用executeOnExecutor()
// getStatus()返回AsyncTask.Status#RUNNING或AsyncTask.Status#FINISHED时抛出IllegalStateException
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

/**
 * Executes the task with the specified parameters. The task returns
 * itself (this) so that the caller can keep a reference to it.
 * 
 * <p>This method is typically used with {@link #THREAD_POOL_EXECUTOR} to
 * allow multiple tasks to run in parallel on a pool of threads managed by
 * AsyncTask, however you can also use your own {@link Executor} for custom
 * behavior.
 * 
 * <p><em>Warning:</em> Allowing multiple tasks to run in parallel from
 * a thread pool is generally <em>not</em> what one wants, because the order
 * of their operation is not defined.  For example, if these tasks are used
 * to modify any state in common (such as writing a file due to a button click),
 * there are no guarantees on the order of the modifications.
 * Without careful work it is possible in rare cases for the newer version
 * of the data to be over-written by an older one, leading to obscure data
 * loss and stability issues.  Such changes are best
 * executed in serial; to guarantee such work is serialized regardless of
 * platform version you can use this function with {@link #SERIAL_EXECUTOR}.
 *
 * <p>This method must be invoked on the UI thread.
 *
 * @param exec The executor to use.  {@link #THREAD_POOL_EXECUTOR} is available as a
 *              convenient process-wide thread pool for tasks that are loosely coupled.
 * @param params The parameters of the task.
 *
 * @return This instance of AsyncTask.
 *
 * @throws IllegalStateException If {@link #getStatus()} returns either
 *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
 *
 * @see #execute(Object[])
 */
// 用指定参数执行任务，任务返回自身以便调用者获取引用
// 此方法用THREAD_POOL_EXECUTOR实现若任务并行处理，也可以用自定义Executor实现指定定制行为；
// 并行任务执行不能保证任务键执行顺序的先后，如果多个任务需有序修改同一个变量，请使用顺序任务
// getStatus()返回AsyncTask.Status#RUNNING或AsyncTask.Status#FINISHED时抛出IllegalStateException
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

// 通过默认Executor执行单个Runnable
@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}

// doInBackground()在后台线程运行过程中可(多次)调用此方法
// 每次调用此方法，都会从子线程向主线程发布更新进度的Message，并在主线程触发onProgressUpdate()
// 若任务已被取消，onProgressUpdate不会被调用
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}

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
    // 在构造此类时传入的是主线程Looper
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
