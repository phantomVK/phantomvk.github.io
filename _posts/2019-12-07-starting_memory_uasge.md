---
layout:     post
title:      "削减应用启动内存"
date:       2019-12-07
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

### 导读

笔者负责的工程长年进行功能迭代，没有处理性能问题，最近终于有空进行内存问题修复。本文记录此次内存问题定位过程及修复方法，并总结开发心得和经验。

应用频繁申请小对象，会占用处理器时间片(虚拟机TLAB划分、小对象初始化)；巨型对象、长生命周期对象，则增加垃圾回收的难度，最终给调试处理器占用、主线程阻塞增加不可预知的状况。同时，大量内存申请会导致性能取样工具卡顿和崩溃。因此，内存应该是性能问题中最优先解决的。

### 一、目标

每次性能优化都要确定阶段性目标，明确目标是”启动速度优化“，还是”修复内存占用高“。不确定目标就开始处理，容易陷入迷惘的状态。

基于产品功能所需，本产品需集成到宿主的应用作为运行部分。所以产品启动过程高内存占用，加上宿主已经申请的空间，容易造成 __OutOfMemoryError__ 而引起应用崩溃。

所以本次首要目标是削减启动申请内存的数量，其次查找启动后的内存泄漏，而其他优化会在后续开展。

### 二、工具

#### 2.1 Android Profiler

按照Google最新开发指引，__Android Profiler__ 是首要推荐性能检测工具，用于分析运行过程产生对象的数量和总大小，下文基于 __Android Studio 3.5.1__ 版本的工具

![android_profiler](/img/android/performance/android_profiler.png)

主要有以下参数：

- **Java**：从 Java 或 Kotlin 代码分配的对象的内存。

- **Native**：从 C 或 C++ 代码分配的对象的内存。

  即使您的应用中不使用 C++，您也可能会看到此处使用的一些原生内存，因为 Android 框架使用原生内存代表您处理各种任务，如处理图像资源和其他图形时，即使您编写的代码采用 Java 或 Kotlin 语言。

- **Graphics**：图形缓冲区队列向屏幕显示像素（包括 GL 表面、GL 纹理等等）所使用的内存。（请注意，这是与 CPU 共享的内存，不是 GPU 专用内存。）

- **Stack**：您的应用中的原生堆栈和 Java 堆栈使用的内存。这通常与您的应用运行多少线程有关。

- **Code**：您的应用用于处理代码和资源（如 dex 字节码、经过优化或编译的 dex 代码、.so 库和字体）的内存。

- **Others**：您的应用使用的系统不确定如何分类的内存。

- **Allocated**：您的应用分配的 Java/Kotlin 对象数。此数字没有计入 C 或 C++ 中分配的对象。

较新的系统版本还有其他额外的数据可以参考，但上面参数多数版本都提供。一般来说从上到下的顺序就是优化难度，即从 __Java堆内存__ 开始难度逐渐提升，到 __Other__ 内存最难处理。

不同用户场景，各部分内存占用比例也不一样，要对比优化前后效果要在同一场景进行测量。

#### 2.2 Eclipse MAT

老牌内存分析工具，用于检查dump之后的内存引用及泄漏问题。不过这个工具对 __Android__ 内存泄漏自动推断不准确，建议人工分析。

__Android Profiler__ 导出内存快照，必须先用 __platform-tools__ 的 __hprof-conv__ 转换后才能导入 __MAT__，而 __-z__ 参数转换得到结果只包含应用自身内存，更易于查看

```bash
$ cd /Users/k/Library/Android/sdk/platform-tools # MacOS
$ hprof-conv -z dump_from.hprof dump_to.hprof
```

导入 __dump_to.hprof__ 到 __MAT__ 后可见内存分类，选择左下角 __Histogram__ 视图

![histogram](/img/android/performance/histogram.png)

如图右键结果，先排除软、弱、虚引用对该实例干扰

![exclude_references](/img/android/performance/exclude_references.png)

剩下的全都是强引用，选择其中一项展开。

![ScreenShotListenManager](/img/android/performance/ScreenShotListenManager.png)

右键点击 __List objects__ ，选择 __with incoming references__ 可看见对象被引用的位置

![leak_MediaContentObserver](/img/android/performance/leak_MediaContentObserver.png)

__MAT__ 中其他可见结果，一般是类加载器持有的静态变量、常量。下图是枚举类型 __FuncType__ 被类加载器持有的示意图，内部28个数值共使用448B内存：

![no_memory_leak](/img/android/performance/no_memory_leak.png)

__Android__ 枚举类型在混淆的优化阶段内联到调用点，不再出现上述448B内存占用。但别忘记现在处于没有开启混淆的 __debug__ 模式，所以能看见枚举类在内存的形态。

### 三、详情

#### 3.1 使用ARouter

__ARouter__ 在本工程中负责页面路由跳转和服务提供，实现组件化的大部分工作，正常来说一般不会有性能问题。

```java
class RoomSummaryComparator : Comparator<RoomSummary> {
    override fun compare(lSummary: RoomSummary?, rSummary: RoomSummary?): Int {
        return when {
            ....
            else -> {
                ....
                when {
                    ....
                    else -> {
                        // 从ARouter获取新的对象SessionManager
                        val lTs = ServiceFactory.getInstance().sessionManager
                                .defaultLatestChatMessageCache
                                .getLatestTextTs(applicationContext, lSummary.roomId)
                                ?: lEvent.getOriginServerTs()

                        // 从ARouter获取新的对象SessionManager
                        val rTs = ServiceFactory.getInstance().sessionManager
                                .defaultLatestChatMessageCache
                                .getLatestTextTs(applicationContext, rSummary.roomId)
                                ?: rEvent.getOriginServerTs()
                        ....
                        }
                    }
                }
            }
        }
    }
}
```

但这里调用嵌套在 __Comparator__ 中，因此房间列表排序累计对 __ARouter__ 调用多达3.6万次，每次调用都创建 __PostCard__ 实例，累计浅内存使用2MB。

![arouter_postcard](/img/android/performance/arouter_postcard_larger.png)

处理方法是整合调用点，避免重复调用 __ARouter__。优化后相同场景只有大约1000次调用。

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

#### 3.2 堆栈跟踪开销

下面代码目的是检查 __JsonObject__ 是否存在名为 __flag__ 的整形值，没有的时候通过捕获异常返回null

```kotlin
val flag = try {
    event.getContent().asJsonObject.get("flag").asInt
} catch (e: Exception) {
    null
}
```

但是 __JsonObject__ 存在该值是少数情况。而异常使用 __StackTraceElement__ 记录堆栈信息，因此上述代码多次生成该实例：每个大小32B，抛出1041次，__ShallowSize__ 总计：33,312B。

考虑到 __StackTraceElement__ 里面用于保存堆栈信息的的字符串变量，实际占用将大于 33,312B。

```kotlin
val jsObj = event.getContent().asJsonObject
if (jsObj.has("flag")) {
    jsObj.get("flag").asInt // 该值约定为int，不考虑其他类型
} else {
    null
}
```

处理方法比较简单，从 __JsonObject__ 获取整形值之前先检查该值是否合法。

#### 3.3 重用对象

很多文章都提到绘制过程如 __onDraw()__ 不应该进行对象创建操作，但自己能准守并不代表能阻止同事这样做。

下面是同事的 __onDraw()__ 和 __onResize()__ 图省事频繁创建 __RectF__。按照每个方法执行一次的情况算，共计创建9个 __RectF__ 对象。

```kotlin
class BubbleShape @JvmOverloads constructor(context: Context) : Shape() {
    override fun draw(canvas: Canvas, paint: Paint) {
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

        // 使用时直接创建RectF实例，下列代码中此问题多次出现
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

修改为复用 __RectF__ 的变量，在使用时先调用 __RectF.set(left, top, right, button)__ 更新值再把 __RectF__ 提供给绘制方法。若确保 __BubbleShape__ 只在主线程执行，则能把 __RectF__ 设置为常量。无论多少个 __BubbleShape__ 实例都只会复用单个 __RectF__ 常量。

```kotlin
class BubbleShape @JvmOverloads constructor(context: Context) : Shape() {
    private val rect = RectF()

    override fun draw(canvas: Canvas, paint: Paint) {
        canvas.drawPath(mTop, paint)
        // 更新rectF实例的数据
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

#### 3.4 重复创建

在 __RecyclerView.Adapter__ 的 __onBindViewHolder()__ 从 __allRows()__ 获取列表，看具体实现发现 __allRows()__ 每次都生成新 __ArrayList__

```kotlin
class MessagesListAdapter(mContext: Context,
                          private val mSession: Session,
                          private val mRoom: Room,
                          private val mMediasCache: MediasCache) : RecyclerView.Adapter<ViewHolder>() {

    private val messageRows = ArrayList<MessageRow>()
    private val headerMessageRows = ArrayList<MessageRow>()
    private val footerMessageRows = ArrayList<MessageRow>()

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        // 从allRows()获取列表
        holders.bind(holder, allRows()[position])
    }

    override fun getPosition(messageRow: MessageRow?): Int {
        // 从allRows()获取列表
        return allRows().indexOf(messageRow)
    }
  
    private fun allRows(): ArrayList<MessageRow> {
      // 每次调用都会生成新的ArrayList实例
      val rows = ArrayList<MessageRow>()
      rows.addAll(headerMessageRows)
      rows.addAll(messageRows)
      rows.addAll(footerMessageRows)
      return rows
    }
}
```

为此还特意咨询相关同事，他还不以为意地说：“不会占用多少内存吧，只是会频繁分配内存”。

最后修改为 __headerMessageRows__、__messageRows__、__footerMessageRows__ 没改变时，复用上次生成的 __ArrayList__ 结果。

#### 3.5 多次Json解析

重复在主线程解析 __Json__ 不仅占用处理器解析导致卡顿，还累积产生很多小对象，直接缓存结果即可

```java
class RoomTopic(private val topicString: String?) {

    // 转换字符串变量topicString为JsonObject
    private var topicJson = try {
        // 小对象开销
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

    ....

    operator fun <reified T> get(key: String): T? {
        return gson.fromJson(getElement(key), object : TypeToken<T>() {}.type)
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

#### 3.6 冗余实例 

每个 __Room__ 对象包含一个 __Gson__ 实例，用于处理 __Json__ 序列化操作

```java
public class Room {
    private final Gson gson = new GsonBuilder().create();
}
```

但根据 [API文档](https://javadoc.io/doc/com.google.code.gson/gson/2.8.0/com/google/gson/Gson.html) 可知 __Gson__ 本身是线程安全，没必要每个 __Room__ 对象持有各自 __Gson__ 实例

> Gson instances are Thread-safe so you can reuse them freely across multiple threads.

让所有 __Room__ 共享常量实例

```java
public class Room {
    private static final GSON_INSTANCE gson = new GsonBuilder().create();
}
```

按照测试账号有307个 __Room__ 实例，每个 __Gson__ 引用占用4B(32位4B，64位8B)，实例体积654B，此优化总计节省内存197KB。

![gson_instance_retained_size](/img/android/performance/gson_instance_retained_size.png)

#### 3.7 关闭资源

内存取样过程还发现 __InputStream__ 实例留存在内存没有正确关闭，从内存申请点发现以下代码

```kotlin
try {
    BitmapFactory.decodeStream(FileInputStream(file), null, opts)
} catch (e: OutOfMemoryError) {
    e.printStackTrace()
}
```

使用 __Kotlin__ 的 __try-resources__ 关闭

```kotlin
try {
    FileInputStream(file).use {
        BitmapFactory.decodeStream(it, null, opts)
    }
} catch (e: OutOfMemoryError) {
    e.printStackTrace()
}
```

#### 3.8 监听器泄漏

最后在 __MAT__ 检查发现小的内存泄漏，会导致 __ScreenShotListenManager__ 被系统引用。不过幸好 __ScreenShotListenManager__ 不引用 __Activity__，所以该泄漏没有造成严重后果。

![leak_MediaContentObserver](/img/android/performance/leak_MediaContentObserver.png)

源码实现如下：

```java
public class ScreenShotListenManager {

    private MediaContentObserver mInternalObserver;
    private MediaContentObserver mExternalObserver;
    private final Handler mUiHandler = new Handler(Looper.getMainLooper());

    // 向系统注册监听器
    public void startListen() {
        mInternalObserver = new MediaContentObserver(MediaStore.Images.Media.INTERNAL_CONTENT_URI, mUiHandler);
        mExternalObserver = new MediaContentObserver(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, mUiHandler);

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
  
    // 从系统注销监听器
    public void stopListen() {
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
    }
}
```

发现在 __Activity__ 的生命周期中注册和注销没有匹配：__onResume()__ 多次注册但在 __onDestroy()__ 仅注销最后一次注册的监听器，导致之前的监听器被系统持有而造成泄漏。

把原来在 __onDestroy__ 的注销操作，移动到 __onResume__ 匹配的 __onPause__ 上即可修复。

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

### 四、效果

没有优化前应用启动过程，在29秒已经显著突破32MB的内存分配，并在40秒到50秒期间出现接近64MB的内存尖峰和多次垃圾回收。

![dump_size_compare](/img/android/performance/dump_size_compare.png)

对比优化后，在33秒仅轻微越过32MB堆内存，而且随后内存申请曲线相对柔和，甚至节省了一次垃圾回收操作，能证明处理方案可减少启动过程对象产生数量，并缩减最大内存的占用量。

时间到1分钟整时启动已完成并停在主界面。最后把应用退到后台，可见两者的堆内存都减少到21MB，应用在后台占用内存不算多。

### 五、总结

若考虑模拟用户动态使用场景、隐含未知的内存泄漏、图中尚存在的尖峰，后续还有进一步优化空间。

经验总结：

- 若频繁输出日志，使用mmap而不是基于Java的日志框架，避免String内存开销；
- 线程安全工具类可处理为常量，为所有调用点提供服务，减少冗余实例；
- 选用知名的图片加载框架不仅节约开发时间，还能避免图片加载引起的内存问题；

- 非受检异常如 __NullPointerException__、__ClassCastException__ 都能预防，处理后可避免堆栈打印快照引起的停顿及 __StackTraceElement__ 开销；
- 削减空闲线程也是优化内存占用大方法之一，需要时可以进行优化；
- 谨慎对待每个内存开销。在 __Comparator__ 或 __for-for__ 助力下，开销能以 __2n__ 甚至 __n*n__ 的方式增加；

参考链接：

- [使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler?hl=zh-cn)
- [我这样减少了26.5M Java内存！](https://wetest.qq.com/lab/view/359.html)
- [Android 内存暴减的秘密？！](https://wetest.qq.com/lab/view/362.html)