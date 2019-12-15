---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

- __静态实例__：短生命周期对象赋值到静态变量；
- __单例__：初始化单例时使用初始化变量是短生命周期对象，类似上述 __静态实例__；
- __子线程__：创建子线程可能会隐式引用父类，但父类结束时没关闭未完成子线程，和 __Handler__ 泄漏类似；
- __匿名内部类__：匿名内部类引用外部类导致无法释放，比如回调；
- __未关闭废弃资源__：BraodcastReceiver、ContentObserver、File、Stream、Bitmap等等；

#### 减少线程

```java
public static Single<String> getFormattedTimestamp(Context context, Event event) {
    return Single.create((SingleOnSubscribe<String>) e -> {
        if (context == null || event == null) {
            e.onSuccess("");
            return;
        }

        // 避免出现时间戳显示1970/1/1 8:0:0的情况
        if (event.originServerTs == 0) {
            event.originServerTs = System.currentTimeMillis();
        }

        String text = DateUtil.INSTANCE.summaryTsToString(context, event.originServerTs);
        e.onSuccess(text == null ? "" : text);
    })
            .subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread());
}
```

峰值134减低到125

```java
public static Single<String> getFormattedTimestamp(Context context, Event event) {
    return Single.create(e -> {
        if (context == null || event == null) {
            e.onSuccess("");
            return;
        }

        // 避免出现时间戳显示1970/1/1 8:0:0的情况
        if (event.originServerTs == 0) {
            event.originServerTs = System.currentTimeMillis();
        }

        String text = DateUtil.INSTANCE.summaryTsToString(context, event.originServerTs);
        e.onSuccess(text == null ? "" : text);
    });
}
```

#### 减少线程

```java
public class MXExecutorService {

    private static final int POOL_SIZE = 6; // 10->6

    public static ExecutorService executor = new ThreadPoolExecutor(POOL_SIZE, POOL_SIZE,
            60L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>());

    public static void execute(Runnable runnable) {
        executor.execute(runnable);
    }

    public static void shutDown() {
        executor.shutdown();
    }
}
```

#### 同步消除

__isInited__ 标志注明模块是否加载完毕，但是新值修改并不依赖旧值

```java
private boolean isInited = false;

// 调用时修改
Synchronized(this){
    isInited = true
}
```

把 __isInited__ 增加 __volatile__ 修饰，这样修改后的值在所有线程即时可见，不需要依赖同步

```java
private volatile boolean isInited = false;

// 调用时修改
isInited = true
```

#### 子线程读写
```
com.foobar.fooapp D/StrictMode: StrictMode policy violation; ~duration=123 ms: android.os.strictmode.DiskReadViolation
        at android.os.StrictMode$AndroidBlockGuardPolicy.onReadFromDisk(StrictMode.java:1504)
        at java.io.UnixFileSystem.checkAccess(UnixFileSystem.java:251)
        at java.io.File.exists(File.java:815)
        at com.foobar.androidsdk.db.MXMediasCache.getFolderFile(MXMediasCache.java:202)
        at com.foobar.androidsdk.db.MXMediasCache.mediaCacheFile(MXMediasCache.java:473)
        at com.foobar.androidsdk.db.MXMediasCache.mediaCacheFileByCompressionType(MXMediasCache.java:432)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$3.invoke(ImageViewer.kt:224)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:252)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:234)
        at org.jetbrains.anko.AsyncKt.runOnUiThread(Async.kt:34)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4.onDownloadComplete(ImageViewer.kt:252)
        at com.foobar.androidsdk.db.MXMediasCache.downloadMedia(MXMediasCache.java:1234)
        at com.foobar.foo.widget.ImageViewer.loadOriginImage(ImageViewer.kt:232)
        at com.foobar.foo.widget.ImageViewer.loadImage(ImageViewer.kt:114)
        at com.foobar.foo.widget.ImageViewer.load(ImageViewer.kt:71)
        at com.foobar.foomessage.chat.adapter.ImageVideoViewerAdapter.loadImage(ImageVideoViewerAdapter.kt:162)
        at com.foobar.foomessage.chat.adapter.ImageVideoViewerAdapter.instantiateItem(ImageVideoViewerAdapter.kt:138)
```

#### 案例

原因：静态实例持有当前页面引用

```java
open class WebViewActivity : BaseWebViewActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.webview)
        webFragment = webViewFragment
    }

    companion object {
        private lateinit var webFragment: WebViewFragment // 内存泄漏
        fun callJs(functionInJs: String, data: String?, callBack: CallBackFunction?) {
            webFrag.callJs(functionInJs, data, callBack)
        }
    }
}
```

修复方案：

```java
open class WebViewActivity : BaseWebViewActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.webview)
        webFragment = webViewFragment
    }
  
    override fun onDestroy() {
        webFrag = null
        super.onDestroy()
    }

    companion object {
        private var webFragment: WebViewFragment? = null
        fun callJs(functionInJs: String, data: String?, callBack: CallBackFunction?) {
            webFrag.callJs(functionInJs, data, callBack)
        }
    }
}
```

#### 案例

原因：子线程持有当前Activity级Context

mContext可能是Activity，当页面关闭时 __RoomAvatarHelper__ 的子线程还持有引用进行图片获取

```java
class RoomAvatarHelper(val mContext: Context,
                       val mRoom: Room,
                       private val mRoomMembers: List<RoomMember>)
```

引用applicationContext调用，不能在内部处理为activity->application

```java
RoomAvatarHelper(this@RoomQRCodeActivity.applicationContext)
```

#### 案例

没有使用正确Context导致Glide泄漏页面View

```java
@Override
public <T> void loadObject(Context context, T object, ImageView view, boolean isRound) {
    if (!isValidContext(context)) {
        return;
    }

    RequestOptions rq = isRound ? ImageOptions.roomRoundAvatarOptions : ImageOptions.roomAvatarOptions;
    Glide.with(context)
            .load(object)
            .apply(rq)
            .into(view);
}
```

```java
@Override
public <T> void loadObject(Context context, T object, ImageView view, boolean isRound) {
    RequestOptions rq = isRound ? ImageOptions.roomRoundAvatarOptions : ImageOptions.roomAvatarOptions;
    Glide.with(view.getContext())
            .load(object)
            .apply(rq)
            .into(view);
}
```

#### 案例

MessageListFragment销毁时没调用

```java
EventTimeLine.mDataHandler.getDataRetriever().cancelHistoryRequest(mRoomId);
```

#### 案例

RoomCreateSettingFragment 字符输入定时器没注销

```java
RxView.clicks(btnFinishCreate)
                .bindToLifecycle(this)  // <-
                .throttleFirst(10, TimeUnit.SECONDS)
                .subscribe { createRoom() }
```

#### 案例6

```java
private fun loadAsync(view: ImageView, isRound: Boolean) {
    if (mBitmaps.isImageUrlsEmpty()) return
    Single.create<File> {
        downloadBitmap()
        it.onSuccess(mCache.saveFile(mSynthesizer.synthesizeBitmap()))
    }
            .asyncIO()
            .bindToLifecycle(view) // <==
            .subscribe({ ImageLoaders.userAvatarLoader().loadFile(applicationContext, it, view, isRound) },
                    { Log.e(TAG, it.localizedMessage) })
}
```

#### 案例

回调被Matrix长时间引用

```java
Observable.fromCallable { RoomSummaryUtils.loadRoomSummaries(currentSession!!) }
    .compose(bindUntilEvent(FragmentEvent.DESTROY_VIEW))
    .map { it.inviteSummaries ?: emptyList() }
    .bindToLifecycle(this) // <===
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
            {
                mNewFriend = if (it != null && it.isNotEmpty()) {
                    ContactsNewFriendWrapper(context!!, it.size, it[0])
                } else {
                    null
                }
            },
            { Log.e(LOG_TAG, "refreshNewFriend", it) },
            { sortData() })
```

#### 案例

```java
private static void displayUnInvitedRoomLatestMessage(@NonNull Context context,
                                                      @NonNull MXSession session,
                                                      @NonNull Room room,
                                                      @NonNull RoomSummary summary,
                                                      boolean isNoDisturbed,
                                                      @NonNull DisplayLatestMessageCallback callback) {
    Event latestEvent = summary.getLatestReceivedEvent();
    if (latestEvent == null) {
        EventTimeline eventTimeline = room.getLiveTimeLine();
        if (eventTimeline == null) {
            callback.displayMessage("");
            callback.displayTime("");
        } else {
            // Fix memory leak: Callback shows below kept by `eventTimeline.backPaginate`.
            final WeakReference<DisplayLatestMessageCallback> weakRef = new WeakReference<>(callback);

            if (eventTimeline.backPaginate(Constants.FILTER_REQUEST_EVENTS, new ApiCallback<Integer>() {
                @Override
                public void onSuccess(Integer integer) {
                    DisplayLatestMessageCallback weakCallback = weakRef.get();
                    if (weakCallback != null) {
                        new Handler().postDelayed(() -> displayUnInvitedRoomLatestMessage(context, session, room,
                                summary, isNoDisturbed, weakCallback), 100);
                        (new BackPaginateEvent(true)).post();
                    }
                }

                @Override
                public void onNetworkError(Exception e) {
                    onError(e.getLocalizedMessage());
                }

                @Override
                public void onMatrixError(MatrixError matrixError) {
                    onError(matrixError.getLocalizedMessage());
                }

                @Override
                public void onUnexpectedError(Exception e) {
                    onError(e.getLocalizedMessage());
                }

                private void onError(String error) {
                    Log.e(TAG, error);
                    DisplayLatestMessageCallback weakCallback = weakRef.get();
                    if (weakCallback != null) {
                        weakCallback.displayMessage("");
                        weakCallback.displayTime("");
                    }
                }
            })) {
            } else {
                callback.displayMessage("");
                callback.displayTime("");
            }
        }
    } else {
        getRoomLatestMessageToDisplay(context, session, summary, isNoDisturbed)
                .subscribe(charSequence -> {
                    callback.displayMessage(charSequence);

                    IMXStore store = session.getDataHandler().getStore();
                    if (store == null)
                        return;
                    Event event = store.getEvent(summary.getLatestReceivedEvent().eventId, room.getRoomId());

                    if (event != null) {
                        if (event.isSending()) {
                            callback.displaySendStatus(true, R.drawable.fc_ic_message_sending);
                        } else if (event.canBeResent()) {
                            callback.displaySendStatus(true, R.drawable.fc_ic_message_send_failed);
                        } else {
                            callback.displaySendStatus(false, -1);
                        }
                    }
                }, throwable -> {
                    Log.e(TAG, "getRoomLastMessageToDisplay : " + throwable.getLocalizedMessage());
                    callback.displayMessage("");
                });
        getFormattedTimestamp(context, latestEvent)
                .subscribe(callback::displayTime, throwable -> Log.e(TAG, "getFormattedTimestamp : " + throwable.getLocalizedMessage()));
    }
}
```

#### 案例

修复

```java
@Override
public void loadFile(Context context, File file, ImageView view, boolean isRound) {
    RequestOptions rq = isRound ? ImageOptions.roomRoundAvatarOptions : ImageOptions.roomAvatarOptions;
    Glide.with(view.getContext()) // 监听ImageView的生命周期
            .load(file)
            .apply(rq)
            .into(view);
}
```

原因

```java
@Override
public void loadFile(Context context, File file, ImageView view, boolean isRound) {
    if (!isValidContext(context)) {
        return;
    }

    RequestOptions rq = isRound ? ImageOptions.roomRoundAvatarOptions : ImageOptions.roomAvatarOptions;
    Glide.with(view.getContext())
            .load(file)
            .apply(rq)
            .into(view);
}
```