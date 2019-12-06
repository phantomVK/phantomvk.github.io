---
layout:     post
title:      "削减启动过程内存开销"
date:       2019-03-08
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

下图是 __Android Studio__ 3.5.1的 Android Profiler测量工具：

![android_profiler](../../img/android/performance/android_profiler.png)

主要有以下重要参数：

- Total：所有内存的总价，包括虚拟机堆内存、堆外内存、显示缓存、栈内存等等；
- Java：DVM/ART虚拟机运行时占用堆内存；
- Native：Native层的.so的malloc或new创建的内存；
- Graphics：OpenGL和SurfaceFlinger内存；
- Stack：线程栈内存；
- Code：内存保存Dex和so的大小；
- Other：不知道怎么算的内存占用；

不同系统版本获取还有其他额外的数据可以参考，但上面是基本几乎所有版本都能提供的。

一般来说，上面从上到下的顺序就是开发处理的难度技术，从Java开始难度逐渐提升。不同的用户场景各部分内存占用比例也不一样，要对比优化前后的效果要在同一场景下进行。

#### ARouter

```java
class RoomSummaryComparator : Comparator<RoomSummary> {
    override fun compare(lSummary: RoomSummary?, rSummary: RoomSummary?): Int {
        return when {
            lSummary == null && rSummary == null -> 0
            lSummary == null -> 1
            rSummary == null -> -1
            else -> {
                val lEvent = lSummary.latestReceivedEvent
                val rEvent = rSummary.latestReceivedEvent
                when {
                    lEvent == null && rEvent == null -> 0
                    lEvent == null -> 1
                    rEvent == null -> -1
                    else -> {
                        // 每次对比都ARouter获取新的对象SessionManager
                        val lTs = ServiceFactory.getInstance().sessionManager
                                .defaultLatestChatMessageCache
                                .getLatestTextTs(applicationContext, lSummary.roomId)
                                ?: lEvent.getOriginServerTs()

                        // 每次对比都ARouter获取新的对象SessionManager
                        val rTs = ServiceFactory.getInstance().sessionManager
                                .defaultLatestChatMessageCache
                                .getLatestTextTs(applicationContext, rSummary.roomId)
                                ?: rEvent.getOriginServerTs()

                        val delta = lTs - rTs
                        when {
                            delta < 0 -> 1
                            delta > 0 -> -1
                            else -> 0
                        }
                    }
                }
            }
        }
    }
}
```

根据 __ARouter__ 的实现，每次经过 __ARouter__ 的实现获取服务的对象引用

```kotlin
class RoomSummaryComparator : Comparator<RoomSummary> {

    private val messageCache = ServiceFactory.getInstance()
            .sessionManager
            .defaultLatestChatMessageCache

    override fun compare(lSummary: RoomSummary?, rSummary: RoomSummary?): Int {
        ....
    }
}
```

房间列表排序过程，累计对 __ARouter__ 调用高达3.6万次，每次调用都会创建新的 __PostCard__ 实例，累计浅内存使用2MB。

![arouter_postcard](../../img/android/performance/arouter_postcard_larger.png)

#### 堆栈跟踪开销

```kotlin
val flag = try {
    event.getContent().asJsonObject.get("flag").asInt
} catch (e: Exception) {
    null
}
```

多次抛出 __StackTraceElement__。每个32B抛出1041次，__ShallowSize__ 总计：33,312B，考虑到 __StackTraceElement__ 里面用于保存堆栈信息的的字符串变量，实际占用将远大于 33,312B。

```kotlin
if (event.getContent().asJsonObject.has("flag")) {
    event.getContent().asJsonObject.get("flag").asInt
} else {
    null
}
```

#### 重用Rect

下面方法 __onDraw()__ 和 __onResize()__ ，按照仅执行一次的条件，需要创建9个 __RectF__ 对象。

```kotlin
class BubbleShape @JvmOverloads constructor(context: Context,
                                            var arrowPosition: ArrowDirection = ArrowDirection.LEFT,
                                            var solidColor: Int = Color.WHITE,
                                            var strokeColor: Int = 0xFFCFCFCF.toInt(),
                                            var strokeWidth: Int = context.dip(1)) : Shape() {

    enum class ArrowDirection {
        LEFT, RIGHT
    }

    var arrowWidth: Float = context.dip(6).toFloat()
    var arrowHeight: Float = context.dip(12).toFloat()
    var arrowMarginTop: Float = context.dip(14).toFloat()
    var cornerRadius: Float = context.dip(10).toFloat()
    private val mTop = Path()
    private val mBottom = Path()

    override fun draw(canvas: Canvas, paint: Paint) {
        paint.color = solidColor
        paint.style = Paint.Style.FILL
        paint.isAntiAlias = true
        paint.isDither = true
        val isLeft = arrowPosition == ArrowDirection.LEFT

        canvas.save()
        if (!isLeft) {
            canvas.scale(-1f, 1f, width / 2, height / 2)
        }

        canvas.drawPath(mTop, paint)
        canvas.drawRect(RectF(arrowWidth, arrowMarginTop + arrowHeight, width, height - cornerRadius), paint)
        canvas.drawPath(mBottom, paint)

        paint.color = strokeColor
        paint.style = Paint.Style.STROKE
        paint.strokeCap = Paint.Cap.ROUND
        paint.strokeJoin = Paint.Join.ROUND
        paint.strokeWidth = strokeWidth.toFloat()
        val offset = strokeWidth / 2f
        val ext = cornerRadius * 0.5f

        canvas.drawArc(RectF(arrowWidth, offset, (arrowWidth + cornerRadius), cornerRadius + offset),
                180f, 90f, false, paint)
        canvas.drawLine(arrowWidth + cornerRadius - ext, offset,
                width - cornerRadius + ext, offset, paint)
        canvas.drawArc(RectF((width - cornerRadius - offset), offset, width - offset, cornerRadius + offset), 270f,
                90f, false, paint)
        canvas.drawLine(width - offset, cornerRadius - ext, width - offset, height - cornerRadius + ext, paint)
        canvas.drawArc(RectF(width - cornerRadius - offset, height - cornerRadius - offset, width - offset, height - offset),
                0f, 90f, false, paint)
        canvas.drawLine(width - cornerRadius + ext, height - offset,
                arrowWidth + cornerRadius - ext, height - offset, paint)
        canvas.drawArc(RectF(arrowWidth, (height - cornerRadius - offset), (arrowWidth + cornerRadius), height - offset),
                90f, 90f, false, paint)
        canvas.drawLine(arrowWidth, height - cornerRadius + ext, arrowWidth, arrowMarginTop + arrowHeight, paint)
        canvas.drawLine(arrowWidth, arrowMarginTop + arrowHeight, 0f, arrowMarginTop + arrowHeight / 2, paint)
        canvas.drawLine(0f, arrowMarginTop + arrowHeight / 2, arrowWidth, arrowMarginTop, paint)
        canvas.drawLine(arrowWidth, arrowMarginTop, arrowWidth, cornerRadius - ext, paint)
        canvas.restore()
    }

    override fun onResize(width: Float, height: Float) {
        mTop.reset()
        mTop.moveTo(arrowWidth, (arrowMarginTop + arrowHeight))
        mTop.lineTo(0f, (arrowMarginTop + arrowHeight / 2))
        mTop.lineTo(arrowWidth, arrowMarginTop)
        mTop.lineTo(arrowWidth, cornerRadius)
        mTop.arcTo(RectF(arrowWidth, 0f, (arrowWidth + cornerRadius), cornerRadius),
                180f, 90f)
        mTop.lineTo((width - cornerRadius), 0f)
        mTop.arcTo(RectF((width - cornerRadius), 0f, width, cornerRadius), 270f, 90f)
        mTop.lineTo(width, (arrowHeight + arrowMarginTop))

        mBottom.reset()
        mBottom.moveTo(width, (height - cornerRadius))
        mBottom.arcTo(RectF((width - cornerRadius), (height - cornerRadius), width, height), 0f, 90f)
        mBottom.lineTo((arrowWidth + cornerRadius), height)
        mBottom.arcTo(RectF(arrowWidth, (height - cornerRadius), (arrowWidth + cornerRadius), height),
                90f, 90f)
        mBottom.lineTo(arrowWidth, height - cornerRadius)

    }
}
```

下面修改为复用 RectF的变量，在需要使用是，先调用 RectF.set(left, top, right, button) 设置新值，然后把该RectF提供给绘制方法。

如果可以确保 __BubbleShape__ 只在主线程执行，则完全把 __RectF__ 设置为常量。这样无论多少个 __BubbleShape__ 实例，最终只会复用唯一一个 __RectF__。

```kotlin
class BubbleShape @JvmOverloads constructor(context: Context,
                                            var arrowPosition: ArrowDirection = ArrowDirection.LEFT,
                                            var solidColor: Int = Color.WHITE,
                                            var strokeColor: Int = 0xFFCFCFCF.toInt(),
                                            var strokeWidth: Int = context.dip(1)) : Shape() {

    enum class ArrowDirection {
        LEFT, RIGHT
    }

    var arrowWidth: Float = context.dip(6).toFloat()
    var arrowHeight: Float = context.dip(12).toFloat()
    var arrowMarginTop: Float = context.dip(14).toFloat()
    var cornerRadius: Float = context.dip(10).toFloat()
    private val mTop = Path()
    private val mBottom = Path()

    private val rect = RectF()

    override fun draw(canvas: Canvas, paint: Paint) {
        paint.color = solidColor
        paint.style = Paint.Style.FILL
        paint.isAntiAlias = true
        paint.isDither = true
        val isLeft = arrowPosition == ArrowDirection.LEFT

        canvas.save()
        if (!isLeft) {
            canvas.scale(-1f, 1f, width / 2, height / 2)
        }

        canvas.drawPath(mTop, paint)
        rect.set(arrowWidth, arrowMarginTop + arrowHeight, width, height - cornerRadius)
        canvas.drawRect(rect, paint)
        canvas.drawPath(mBottom, paint)

        paint.color = strokeColor
        paint.style = Paint.Style.STROKE
        paint.strokeCap = Paint.Cap.ROUND
        paint.strokeJoin = Paint.Join.ROUND
        paint.strokeWidth = strokeWidth.toFloat()
        val offset = strokeWidth / 2f
        val ext = cornerRadius * 0.5f

        rect.set(arrowWidth, offset, (arrowWidth + cornerRadius), cornerRadius + offset)
        canvas.drawArc(rect, 180f, 90f, false, paint)
        canvas.drawLine(arrowWidth + cornerRadius - ext, offset,
                width - cornerRadius + ext, offset, paint)
        rect.set((width - cornerRadius - offset), offset, width - offset, cornerRadius + offset)
        canvas.drawArc(rect, 270f, 90f, false, paint)
        canvas.drawLine(width - offset, cornerRadius - ext, width - offset, height - cornerRadius + ext, paint)

        rect.set(width - cornerRadius - offset, height - cornerRadius - offset, width - offset, height - offset)
        canvas.drawArc(rect, 0f, 90f, false, paint)
        canvas.drawLine(width - cornerRadius + ext, height - offset,
                arrowWidth + cornerRadius - ext, height - offset, paint)
        rect.set(arrowWidth, (height - cornerRadius - offset), (arrowWidth + cornerRadius), height - offset)
        canvas.drawArc(rect, 90f, 90f, false, paint)
        canvas.drawLine(arrowWidth, height - cornerRadius + ext, arrowWidth, arrowMarginTop + arrowHeight, paint)
        canvas.drawLine(arrowWidth, arrowMarginTop + arrowHeight, 0f, arrowMarginTop + arrowHeight / 2, paint)
        canvas.drawLine(0f, arrowMarginTop + arrowHeight / 2, arrowWidth, arrowMarginTop, paint)
        canvas.drawLine(arrowWidth, arrowMarginTop, arrowWidth, cornerRadius - ext, paint)
        canvas.restore()
    }

    override fun onResize(width: Float, height: Float) {
        mTop.reset()
        mTop.moveTo(arrowWidth, (arrowMarginTop + arrowHeight))
        mTop.lineTo(0f, (arrowMarginTop + arrowHeight / 2))
        mTop.lineTo(arrowWidth, arrowMarginTop)
        mTop.lineTo(arrowWidth, cornerRadius)

        rect.set(arrowWidth, 0f, (arrowWidth + cornerRadius), cornerRadius)
        mTop.arcTo(rect, 180f, 90f)
        mTop.lineTo((width - cornerRadius), 0f)

        rect.set((width - cornerRadius), 0f, width, cornerRadius)
        mTop.arcTo(rect, 270f, 90f)
        mTop.lineTo(width, (arrowHeight + arrowMarginTop))

        mBottom.reset()
        mBottom.moveTo(width, (height - cornerRadius))

        rect.set((width - cornerRadius), (height - cornerRadius), width, height)
        mBottom.arcTo(rect, 0f, 90f)
        mBottom.lineTo((arrowWidth + cornerRadius), height)

        rect.set(arrowWidth, (height - cornerRadius), (arrowWidth + cornerRadius), height)
        mBottom.arcTo(rect, 90f, 90f)
        mBottom.lineTo(arrowWidth, height - cornerRadius)
    }
}
```

#### 重复创建

```kotlin
class MessagesListAdapter(mContext: Context,
                          private val mSession: Session,
                          private val mRoom: Room,
                          private val mMediasCache: MediasCache) : RecyclerView.Adapter<ViewHolder>() {

    private val messageRows = ArrayList<MessageRow>()
    private val headerMessageRows = ArrayList<MessageRow>()
    private val footerMessageRows = ArrayList<MessageRow>()

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holders.bind(holder, allRows()[position])
    }

    override fun getPosition(messageRow: MessageRow?): Int {
        return allRows().indexOf(messageRow)
    }

    override fun getItemCount(): Int {
        return headerMessageRows.size + messageRows.size + footerMessageRows.size
    }

    override fun getItem(position: Int): MessageRow {
        return when (position) {
            in 0 until headerMessageRows.size -> {
                headerMessageRows[position]
            }

            in headerMessageRows.size until headerMessageRows.size + messageRows.size -> {
                messageRows[position - headerMessageRows.size]
            }

            else -> {
                footerMessageRows[position - headerMessageRows.size - messageRows.size]
            }
        }
    }

    private fun allRows(): ArrayList<MessageRow> {
      val rows = ArrayList<MessageRow>()
      rows.addAll(headerMessageRows)
      rows.addAll(messageRows)
      rows.addAll(footerMessageRows)
      return rows
    }
}
```

重复在主线程解析 __Json__ 导致性能开销

```java
class RoomTopic(private val topicString: String?) {

    // 转换字符串变量topicString为JsonObject
    private var topicJson = try {
        val element = JsonParser().parse(topicString)
        if (element.isJsonObject) {
            element.asJsonObject!!
        } else {
            JsonObject().apply {
                addProperty(JSON_KEY_TOPIC, topicString)
            }
        }
    } catch (e: Exception) {
        JsonObject().apply {
            addProperty(JSON_KEY_TOPIC, topicString)
        }
    }

    // 房间属性
    var topic by jsonProperty<String>(JSON_KEY_TOPIC)

    // 是否单聊
    var isDirect by jsonProperty<Boolean>(JSON_KEY_IS_DIRECT)

    // 是否发送enterRoom
    var isSendEnterRoom by jsonProperty<Boolean>(JSON_KEY_IS_SEND_ENTER_ROOM)

    // 是否开启水印
    var isWaterMark by jsonProperty<Boolean>(JSON_KEY_IS_WATER_MARK)
  
    fun getElement(key: String): JsonElement? = topicJson[key]

    operator fun <reified T> get(key: String): T? {
        return gson.fromJson(getElement(key), object : TypeToken<T>() {}.type)
    }

    operator fun <V> set(key: String, value: V) {
        topicJson.add(key, gson.toJsonTree(value))
    }

    private inline fun <reified T> jsonProperty(key: String): ReadWriteProperty<Any, T?> {
        return object : ReadWriteProperty<Any, T?> {
            override fun getValue(thisRef: Any, property: KProperty<*>): T? = get<T>(key)

            override fun setValue(thisRef: Any, property: KProperty<*>, value: T?) =
                    set(key, value)
        }
    }
}
```

#### 重复创建实例 

__Room__ 对象包含一个 __Gson__ 实例，用于处理 __Json__ 序列化操作：

```java
public class Room {
    private final Gson gson = new GsonBuilder().create();
}
```

但根据 [API文档](https://javadoc.io/doc/com.google.code.gson/gson/2.8.0/com/google/gson/Gson.html) 可知 __Gson__ 本身是线程安全，完全没必要每个 __Room__ 对象持有各自 __Gson__ 实例

> Gson instances are Thread-safe so you can reuse them freely across multiple threads.

所以让所有 __Room__ 共享同一个常量实例：

```java
public class Room {
    private static final Gson gson = new GsonBuilder().create();
}
```

按照测试账号有307个 __Room__ 实例，每个 __Gson__ 实例的引用占用4B(32位是4B，64位是8B)，实例本身至少654B，此优化节省内存197KB。

![gson_instance_retained_size](../../img/android/performance/gson_instance_retained_size.png)



#### try resource

```kotlin
try {
    BitmapFactory.decodeStream(FileInputStream(file), null, opts)
} catch (e: OutOfMemoryError) {
    e.printStackTrace()
    System.gc()
}
```

```java
try {
    FileInputStream(file).use {
        BitmapFactory.decodeStream(it, null, opts)
    }
} catch (e: OutOfMemoryError) {
    e.printStackTrace()
    System.gc()
}
```

### #

```
com.finogeeks.finochatapp D/StrictMode: StrictMode policy violation; ~duration=123 ms: android.os.strictmode.DiskReadViolation
        at android.os.StrictMode$AndroidBlockGuardPolicy.onReadFromDisk(StrictMode.java:1504)
        at java.io.UnixFileSystem.checkAccess(UnixFileSystem.java:251)
        at java.io.File.exists(File.java:815)
        at org.matrix.androidsdk.db.MXMediasCache.getFolderFile(MXMediasCache.java:202)
        at org.matrix.androidsdk.db.MXMediasCache.mediaCacheFile(MXMediasCache.java:473)
        at org.matrix.androidsdk.db.MXMediasCache.mediaCacheFileByCompressionType(MXMediasCache.java:432)
        at com.finogeeks.finochat.widget.ImageViewer$loadOriginImage$3.invoke(ImageViewer.kt:224)
        at com.finogeeks.finochat.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:252)
        at com.finogeeks.finochat.widget.ImageViewer$loadOriginImage$4$onDownloadComplete$1.invoke(ImageViewer.kt:234)
        at org.jetbrains.anko.AsyncKt.runOnUiThread(Async.kt:34)
        at com.finogeeks.finochat.widget.ImageViewer$loadOriginImage$4.onDownloadComplete(ImageViewer.kt:252)
        at org.matrix.androidsdk.db.MXMediasCache.downloadMedia(MXMediasCache.java:1234)
        at com.finogeeks.finochat.widget.ImageViewer.loadOriginImage(ImageViewer.kt:232)
        at com.finogeeks.finochat.widget.ImageViewer.loadImage(ImageViewer.kt:114)
        at com.finogeeks.finochat.widget.ImageViewer.load(ImageViewer.kt:71)
        at com.finogeeks.finochatmessage.chat.adapter.ImageVideoViewerAdapter.loadImage(ImageVideoViewerAdapter.kt:162)
        at com.finogeeks.finochatmessage.chat.adapter.ImageVideoViewerAdapter.instantiateItem(ImageVideoViewerAdapter.kt:138)
```

![dump_size_compare](../../img/android/performance/dump_size_compare.png)

#### 监听器泄漏

![leak_MediaContentObserver](../../img/android/performance/leak_MediaContentObserver.png)

```java
public class ScreenShotListenManager {

    private MediaContentObserver mInternalObserver;
    private MediaContentObserver mExternalObserver;
    private final Handler mUiHandler = new Handler(Looper.getMainLooper());

    public void startListen() {
        assertInMainThread();

        // 记录开始监听的时间戳
        mStartListenTime = System.currentTimeMillis();

        // 创建内容观察者
        mInternalObserver = new MediaContentObserver(MediaStore.Images.Media.INTERNAL_CONTENT_URI, mUiHandler);
        mExternalObserver = new MediaContentObserver(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, mUiHandler);

        // 注册内容观察者
        mContext.getApplicationContext()
                .getContentResolver()
                .registerContentObserver(
                        MediaStore.Images.Media.INTERNAL_CONTENT_URI,
                        false,
                        mInternalObserver);

        mContext.getApplicationContext()
                .getContentResolver()
                .registerContentObserver(
                        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                        false,
                        mExternalObserver);
    }
    /**
     * 停止监听，注销内容观察者
     */
    public void stopListen() {
        assertInMainThread();

        if (mInternalObserver != null) {
            try {
                mContext.getApplicationContext()
                        .getContentResolver()
                        .unregisterContentObserver(mInternalObserver);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                mInternalObserver = null;
            }
        }

        if (mExternalObserver != null) {
            try {
                mContext.getApplicationContext()
                        .getContentResolver()
                        .unregisterContentObserver(mExternalObserver);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                mExternalObserver = null;
            }
        }

        // 清空数据
        mStartListenTime = 0;

        // 必须设置mListener为空，因为mListener会隐式持有Activity
        mListener = null;
    }
}
```

```java
private class MediaContentObserver extends ContentObserver {

    private Uri mContentUri;

    MediaContentObserver(Uri contentUri, Handler handler) {
        super(handler);
        mContentUri = contentUri;
    }

    @Override
    public void onChange(boolean selfChange) {
        super.onChange(selfChange);
        handleMediaContentChange(mContentUri);
    }
}
```

```java
@Override
protected void onResume() {
    super.onResume();
    if (mManager != null) {
        mManager.startListen();
    }
}

@Override
protected void onDestroy() {   
    super.onDestroy();
    if (mManager != null) {
        mManager.stopListen();
        mManager = null;
    }
}
```

```java
@Override
protected void onResume() {
    super.onResume();
    if (mManager != null) {
        mManager.startListen();
    }
}

@Override
protected void onPause() {
    super.onPause();
    if (mManager != null) {
        mManager.stopListen();
    }
}
```

