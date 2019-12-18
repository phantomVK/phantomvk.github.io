---
layout:     post
title:      "Android性能优化范例"
date:       2019-12-27
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

- __静态实例__：短生命周期对象赋值到静态变量；
- __单例__：初始化单例时使用初始化变量是短生命周期对象，类似上述 __静态实例__；
- __子线程__：创建子线程可能会隐式引用父类，但父类结束时没关闭未完成子线程，和 __Handler__ 泄漏类似；
- __匿名内部类__：匿名内部类引用外部类导致无法释放，比如回调；
- __未关闭废弃资源__：__BroadcastReceiver__、__ContentObserver__、__File__、__Stream__、__Bitmap__等；



LeakCanary、BlockCanary

StrictMode

```kotlin
StrictMode.setThreadPolicy(StrictMode.ThreadPolicy.Builder().detectAll().build())
StrictMode.setVmPolicy(VmPolicy.Builder().detectAll().build())
```

#### 减少线程

在 __Android Profiler__ 看到展示带时间的列表时线程数量抖动较大，检查发现从时间戳计算可读时间字符串过程问题很大。

首先，通过RxJava.single发送结果，还把工作放在子线程完成。然后，一个是线程类型使用错误，让计算密集型的工作放在io线程造成浪费。其次，这个时间计算方法经过我上期优化后，实际耗时很低，可以放主线程直接算。

```java
public static Single<String> getFormattedTimestamp(Context context, Event event) {
    return Single.create((SingleOnSubscribe<String>) e -> {
        String text = DateUtil.summaryTsToString(context, event.originServerTs);
        e.onSuccess(text == null ? "" : text);
    })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread());
}
```

进行以下改造：

- 检查 __summaryTsToString(context, long)__ 耗时情况按实际情况处理；

- __RxJava__ 也会小量消耗性能，处理简单数据不必依赖 __RxJava__；
- 移除 __RxJava__ 后，其线程绑定和切换开销理所当然地消除掉；
- 最后调整外部调用点移除 __RxJava__ 调用；

优化后相同场景峰值从134减低到125，减少数量相当于瞬时屏上items数量

```java
public static String getFormattedTimestamp(Context context, Event event) {
    String text = DateUtil.summaryTsToString(context, event.originServerTs);
    return text == null ? "" : text;
}
```

#### 减少线程

```java
public class ExecutorService {
    private static final int POOL_SIZE = 6; // 10->6

    public static ExecutorService executor = new ThreadPoolExecutor(POOL_SIZE, POOL_SIZE,
            60L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>());
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
        at com.foobar.androidsdk.db.MediasCache.getFolderFile(MediasCacheava:202)
        at com.foobar.androidsdk.db.MediasCache.mediaCacheFile(MediasCache.java:473)
        at com.foobar.androidsdk.db.MediasCache.mediaCacheFileByCompressionType(MediasCache.java:432)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$3.invoke(ImageViewer.kt:224)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:252)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:234)
        at org.jetbrains.anko.AsyncKt.runOnUiThread(Async.kt:34)
        at com.foobar.foo.widget.ImageViewer$loadOriginImage$4.onDownloadComplete(ImageViewer.kt:252)
        at com.foobar.androidsdk.db.MediasCache.downloadMedia(MediasCache.java:1234)
        at com.foobar.foo.widget.ImageViewer.loadOriginImage(ImageViewer.kt:232)
        at com.foobar.foo.widget.ImageViewer.loadImage(ImageViewer.kt:114)
        at com.foobar.foo.widget.ImageViewer.load(ImageViewer.kt:71)
        at com.foobar.foomessage.chat.adapter.ImageVideoViewerAdapter.loadImage(ImageVideoViewerAdapter.kt:162)
        at com.foobar.foomessage.chat.adapter.ImageVideoViewerAdapter.instantiateItem(ImageVideoViewerAdapter.kt:138)
```

#### 案例

原因：静态实例持有当前页面引用

```java
class WebViewActivity : BaseWebViewActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.webview)
        webFragmentment = WebFragmentment()
    }

    companion object {
        private lateinit var webFragmentment: WebViewFragment // 内存泄漏

        fun callJs(functionInJs: String, data: String?, callBack: CallBackFunction?) {
            webFragment.callJs(functionInJs, data, callBack)
        }
    }
}
```

修复方案：

```java
class WebViewActivity : BaseWebViewActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.webview)
        webFragmentment = WebFragmentment()
    }
  
    override fun onDestroy() {
        webFragment = null
        super.onDestroy()
    }

    companion object {
        private var webFragmentment: WebViewFragment? = null

        // 从外部调用页面的JS
        fun callJs(functionInJs: String, data: String?, callBack: CallBackFunction?) {
            webFragment.callJs(functionInJs, data, callBack)
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



