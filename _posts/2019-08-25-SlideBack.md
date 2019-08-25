---
layout:     post
title:      "Android滑动返回实现"
date:       2019-08-25
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

背景
------------

滑动退出最早出现在 __iOS7__，是系统提供的标准功能，一直沿用到现在。

对 __Android__  来说，更多是因为iOS出现该功能，产品经理为统一移动终端的 __user experience__ 而跟进。相比 iOS在系统层完美的实现方式，Android无论用哪种方式，都需要在内存、处理器性能、流畅性之一作出牺牲。

方案
---------

现时讨论最多的两种方案是：

- 截屏图片模仿透明背景；
- Window及Activity透明处理；

#### 截屏图片

__截屏图片__ 是截取当前屏幕的图片，打开新界面的时候传递给新界面。新界面把该图片作为底下一层界面的透视图。很多技术不错的同行提起过这种方案，不过这种方案不具备可用性。

Android图片多使用 __RGB565__ 内存定义，按照流行屏幕尺寸 __1920*1080__ 宽高各减半算内存占用：

> (5 + 6 + 5) / 8 *  (1920 / 2) * (1080 / 2) = 1, 036, 800Bytes = 1013KB

每个页面为保存截图都要占用近1MB内存，那截图时处理器占用，图片传递延迟，有考虑过吗？

#### Window及Activity透明处理

如果只能在两个不完美的方案里挑一个，我选这个方案。

流行开源库 [SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout) 也选择这种方案。不过这个工程从 __2018.07.10__ 后再也没有维护，在此前提出的问题都没有修复，更不用说，对源码迁移到AndroidX这种要求。

除对好些异常没有处理，本身透明逻辑定义的时间点也是有问题的。

先看界面正常生命周期：

```
I/MainActivity@bc1f0e: onCreate
I/MainActivity@bc1f0e: onStart
I/MainActivity@bc1f0e: onResume

I/MainActivity@bc1f0e: onPause
I/MainActivity@2e5c3a1d: onCreate
I/MainActivity@2e5c3a1d: onStart
I/MainActivity@2e5c3a1d: onResume
I/MainActivity@bc1f0e: onStop

I/MainActivity@2e5c3a1d: onPause
I/MainActivity@2a2ef203: onCreate
I/MainActivity@2a2ef203: onStart
I/MainActivity@2a2ef203: onResume
I/MainActivity@2e5c3a1d: onStop

I/MainActivity@2a2ef203: onPause
I/MainActivity@3d0b6459: onCreate
I/MainActivity@3d0b6459: onStart
I/MainActivity@3d0b6459: onResume
I/MainActivity@2a2ef203: onStop
```

以上周期大家都很熟悉，不再赘述。

__SwipeBackLayout__ 异常生命周期：

```
I/DemoActivity@bd11e3a: onCreate
I/DemoActivity@bd11e3a: onStart
I/DemoActivity@bd11e3a: onResume

I/DemoActivity@bd11e3a: onPause
I/DemoActivity@24fbcaab: onCreate
I/DemoActivity@24fbcaab: onStart
I/DemoActivity@24fbcaab: onResume

I/DemoActivity@24fbcaab: onPause
I/DemoActivity@378d89c8: onCreate
I/DemoActivity@378d89c8: onStart
I/DemoActivity@378d89c8: onResume

I/DemoActivity@378d89c8: onPause
I/DemoActivity@20264021: onCreate
I/DemoActivity@20264021: onStart
I/DemoActivity@20264021: onResume

I/DemoActivity@20264021: onPause
I/DemoActivity@2a17ef06: onCreate
I/DemoActivity@2a17ef06: onStart
I/DemoActivity@2a17ef06: onResume
```

所有已启动界面！所有已启动界面！在打开新界面后都不会走到 __onStop__，也就是说所有应该在 __onStop__ 停止的操作、释放的资源，都没有机会触发。导致的结果是：当打开简单页面数量超过7个，就会出现新页面进程卡顿。

后来独立调研后，发现我的解决方案没法基于 __SwipeBackLayout__ 进行修改。所以实现新开源库 [phantomVK/SlideBack](https://github.com/phantomVK/SlideBack) 达到期望技术目标。新库对 __反射操作__、__页面过度绘制__、__内存占用__，和 __生命周期异常__ 进行修复。

__SlideBack__ 优化方案：

```
I/MainActivity@3b063dbb: onCreate
I/MainActivity@3b063dbb: onStart
I/MainActivity@3b063dbb: onResume

I/MainActivity@2304ed15: onStop
I/MainActivity@3b063dbb: onPause
I/MainActivity@2941f6d1: onCreate
I/MainActivity@2941f6d1: onStart
I/MainActivity@2941f6d1: onResume

I/MainActivity@3b063dbb: onStop
I/MainActivity@2941f6d1: onPause
I/MainActivity@b2f0dd7: onCreate
I/MainActivity@b2f0dd7: onStart
I/MainActivity@b2f0dd7: onResume

I/MainActivity@2941f6d1: onStop
I/MainActivity@b2f0dd7: onPause
I/MainActivity@238b744d: onCreate
I/MainActivity@238b744d: onStart
I/MainActivity@238b744d: onResume
```

和前文日志对比可以发现，新方案只有当前界面的下一级界面生命周期没有走到 __onStop__。具体原因可看 [透明Activity生命周期变化](/2016/12/16/Android_LifeCycle/)。优化后界面都能正确释放资源，无论打开多少个新页面，不再有卡顿问题。

改进
-----

#### 反射优化

反射在Java操作中耗费性能，而且基于应用场景的原因不能在子线程内运行。对此进行以下改进，把类方法查找操作放在静态初始化块内完成，后续无需每次使用都查找一次：

```java
public class TranslucentHelper {

    private static Method sOptionsMethod;
    private static Method sInvokeMethod;
    private static Method sRevokeMethod;

    static {
        try {
            init();
        } catch (Throwable ignore) {
        }
    }

    private static void init() throws NoSuchMethodException {
        if (SDK_INT < KITKAT) return;

        Class<?>[] classes = Activity.class.getDeclaredClasses();
        Class<?> clz = null;
        for (final Class c : classes) {
            if (c.getSimpleName().equals("TranslucentConversionListener")) {
                clz = c;
                break;
            }
        }

        if (clz == null) return;
        if (SDK_INT >= LOLLIPOP) {
            sOptionsMethod = Activity.class.getDeclaredMethod("getActivityOptions");
            sOptionsMethod.setAccessible(true);

            sInvokeMethod = Activity.class.getDeclaredMethod("convertToTranslucent", clz, ActivityOptions.class);
            sInvokeMethod.setAccessible(true);
        } else {
            sInvokeMethod = Activity.class.getDeclaredMethod("convertToTranslucent", clz);
            sInvokeMethod.setAccessible(true);
        }

        sRevokeMethod = Activity.class.getDeclaredMethod("convertFromTranslucent");
        sRevokeMethod.setAccessible(true);
    }

    ....
}
```

#### 生命周期

[phantomVK/SlideBack](https://github.com/phantomVK/SlideBack) 比SwipeBackLayout最大优化主要是生命周期的处理。具体的细节建议自行看源码，全部操作基于唯一方针：__当前页面可见即可用，新页面启动即撤销__。

#### 其他优化

其他修复和改进：

- 源码库已迁移并适配 __AndroidX__；
- 继承的父类为 __AppCompatActivity__；
- 内部已捕获异常 __ArrayIndexOutOfBoundsException__；
- 使用基于 __Android28__ 定制 __ViewDragHelper__ 工具类；
- 更细致功能启用、停用开关控制，避免冗余内存开销；