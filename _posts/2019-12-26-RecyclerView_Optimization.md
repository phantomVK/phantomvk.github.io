---
layout:     post
title:      "RecyclerView优化"
date:       2019-12-26
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Optimization
---

## 一、数据拉取与处理

#### 1.1 数据处理

大家都知道，数据从网络和本地磁盘加载到内存，为避免主线程阻塞会在io线程上进行。

但绑定视图数据时，重转换操作也会导致列表滚动卡顿。如果能够预计算，如字符串替换、拼接，数据填充到 __Adapter__ 前就合并到加载过程一并完成转换。

#### 1.2 数据缓存

__ViewHolder__ 公用数据放在 __Adapter__ 中，视图绑定时从 __Adapter__ 获取避免保存多份或实时计算结果。

#### 1.3 局部更新

局部更新可减少刷新所需时间，推荐使用 __DiffUtil__ 计算数据集或手动调用 __notifyItemInserted()__ 方法。

```kotlin
fun addItem(reply: Reply) {
    list.add(0, reply)
    notifyItemInserted(0)
}
```

## 二、视图优化

#### 2.1 填充次数

__RecyclerView__ 三级缓存目的是复用已填充视图。当缓存数量不足填满屏幕而频繁创建，滑出屏幕后又超过缓存数量被销毁，占用处理器同时又产生临时对象。而这种情况多发生在高度很小的视图。

![recyclerview_notice](/img/android/performance/recyclerview_notice.jpg)

根据经验来看默认缓存阈值偏小，不同类型需要缓存最大数量也不同。最好根据屏幕和视图尺寸动态计算分类缓存数量。

#### 2.2 过渡绘制

减少过度绘制同样适用于 __RecyclerView__ 视图布局，提高滑动帧率。

#### 2.3 布局填充时间

__LayoutInflater__ 实例化xml视图时不仅需要遍历xml节点，而且视图要用反射实例化，导致复杂视图耗时较长，列表快速滚动容易卡顿。降低布局复杂度、增加布局缓存数量都能有效缓解问题。

虽然 __ConstraintLayout__ 具备去除布局层次的能力，不过很多开发者发现和  __RecyclerView__ 使用有严重问题。

原生视图组装、__Anko__、__Litho__、__JetPack compose__ 原理都类似，没有xml遍历和类反射过程提高了性能。

不过上述方案各自缺点也明显：原生视图编写代码量大，__Anko__ 配合style使用要自定义方法，__Litho__ 由于技术实现令视图灵活性较低，__JetPack compose__ 还处于实验性截断。并且开发预览支持度不同，__Anko__ 编辑时不能预览。

按照现在的发展形势来看，个人推荐 __Anko__ 和 __JetPack compose__：

原生视图组装：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    val params = FrameLayout.LayoutParams(WRAP_CONTENT, WRAP_CONTENT)
    params.gravity = Gravity.CENTER

    val button = Button(this)
    button.text = "Start"
    button.isAllCaps = false
    button.setOnClickListener { startActivity<MessagesActivity>() }

    addContentView(button, params)
}
```

__Anko__ 代码示例 [phantomVK/MessageKit - MessageHolder](https://github.com/phantomVK/MessageKit/blob/f7ddced2a75b95b821354de37074f7b2bfde9b4e/app/src/main/java/com/phantomvk/messagekit/adapter/MessageHolder.kt)

```kotlin
class TextMessageLayout : AnkoComponent<ViewGroup> {
    override fun createView(ui: AnkoContext<ViewGroup>) = with(ui) {
        frameLayout {
            lparams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT)

            textView {
                id = R.id.text
                autoLinkMask = Linkify.EMAIL_ADDRESSES or Linkify.WEB_URLS
                gravity = Gravity.CENTER_VERTICAL
                includeFontPadding = false
                minimumHeight = dip(40)
                padding = dip(10)
                textColor = Color.parseColor("#222222")
                setTextIsSelectable(false)
                textSize = 16f //sp
            }
        }
    }
}
```

__Jetpack compose__ 官方示例 [Jetpack Compose Basics](https://developer.android.com/jetpack/compose/tutorial)

```kotlin
@Preview
@Composable
fun NewsStory() {
    val image = +imageResource(R.drawable.header)
    MaterialTheme {
            Column(
                modifier = Spacing(16.dp)
            ) {
                Container(modifier = Height(180.dp) wraps Expanded) {
                    Clip(shape = RoundedCornerShape(8.dp)) {
                        DrawImage(image)
                    }
                }
                HeightSpacer(16.dp)
                Text("A day in Shark Fin Cove")
                Text("Davenport, California")
                Text("December 2018")
            }
    }
}
```

关于 __Litho__ 的使用美团有深入研究，参考 [基本功 - Litho的使用及原理剖析](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html) 和 [Litho在美团动态化方案MTFlexbox中的实践](https://tech.meituan.com/2019/09/19/litho-practice-in-dynamic-program-mtflexbox.html)。

#### 2.4 对象生成

当 __RecyclerView__ 视图分类较多时，缓存元素总数也会很多，加上复杂视图 __ViewHolder__ 很多数据成员占用很多内存。用测量工具对比列表展示前后内存占用，可以大概确定用量。结构样式相似的视图可用 __View.visibility__ 减少不同类别。

## 三、参数配置

__RecyclerView__ 自有参数也能提升渲染性能。

#### 3.1 setHasFixedSize

描述 __RecyclerView__ 是否根据items总宽高重新计算大小。多数情况 __RecyclerView__ 宽高是 __MATCH_PARENT__，设置为true减少重新计算大小次数。父布局大小改变而导致 __RecyclerView__ 调整不受此参数限制。

```kotlin
recyclerView.setHasFixedSize(true)
```

#### 3.2 mCachedViews

__RecyclerView__ 离屏缓存 __mCachedViews__ 默认为2，即视图以移出屏幕后，放到共享缓存池前缓存2个元素，目的优化慢速向前、向后滑动抖动。

```kotlin
recyclerView.setItemViewCacheSize(4)
```

按照个人经验判断这个值设置超过4效果不明显。

#### 3.3 setHasStableIds

指定参数为true避免增删items时，引起无关的视图刷新。

```korlin
adapter.setHasStableIds(true)
```

当然还需要重写方法获取id计算结果。

```kotlin
override fun getItemId(position: Int): Long {
    return items[position].id.hashCode().toLong()
}
```

## 四、参考链接

- [RecyclerView Scrolling Performance](https://stackoverflow.com/q/27188536/8750399)
- [How to optimize Recyclerview in Android](https://mobikul.com/how-to-optimize-recyclerview-in-android/)
- [RecyclerView item optimizations](https://medium.com/@programmerr47/recyclerview-item-optimizations-cae1aed0c321)
- [Improve RecyclerView Performance](https://blog.usejournal.com/improve-recyclerview-performance-ede5cec6c5bf)
- [基本功 - Litho的使用及原理剖析](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)
- [Litho在美团动态化方案MTFlexbox中的实践](https://tech.meituan.com/2019/09/19/litho-practice-in-dynamic-program-mtflexbox.html)
- [jetpack compose - Android’s modern toolkit for building native UI](https://developer.android.com/jetpack/compose)

