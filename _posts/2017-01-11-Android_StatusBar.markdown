---
layout:     post
title:      "Android沉浸式状态栏变色"
date:       2017-01-11
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android
---

# 一、前言

Android推崇Material Design，其中一个亮点是沉浸式状态栏。该形式状态栏从Android 4.4（v19）开始得到原生支持。这次我们在应用中简单实现一下。

最终实现的效果
![img](/img/android/toolbar/toolBar.png)

# 二、依赖

在`build.gradle(Module:app)`中添加依赖项，版本根据Android Studio提示使用最新即可，修改完成记得同步一下。

```
compile 'com.android.support:appcompat-v7:25.1.0'
compile 'com.android.support:support-v4:25.1.0'
compile 'com.android.support:design:25.1.0'
```

# 三、样式

在`res/values/styles.xml`新建自定义主题`BaseAppTheme`，其父主题是`Theme.AppCompat.Light.NoActionBar`，然后再创建一个`BarTheme`继承`BaseAppTheme`。

```xml
<style name="BaseAppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
</style>

<style name="BarTheme" parent="BaseAppTheme" />
```

在`res`文件夹下新建一个名为`values-v19`的文件夹。
![img](/img/android/toolbar/res.png)

在`res/values-v19/`新建一个`styles.xml`，名字在IDE里面显示为`styles.xml(v19)`。这个样式里面只需编写`BarTheme`继承`BaseAppTheme`。被继承`BaseAppTheme`是上一个`styles.xml`设置好的，会自动引用过来。

```xml
<style name="BarTheme" parent="BaseAppTheme">
    <item name="android:windowTranslucentStatus">true</item>
    <item name="android:windowTranslucentNavigation">true</item>
    <item name="android:textColorPrimary">#FFFFFF</item>
</style>
```

三个参数分别作用：

* 半透明状态栏设置为启用，否则没有沉浸式状态栏效果，参数从v19开始引入；
* 为`NavigationView`提供半透明支持，通常是左抽屉。没有抽屉不需要这个参数；
* 用来改变Toolbar标题文字颜色；

# 四、自定义ToolBar

在`activity_main.xml`自定义ToolBar，`background`改变ToolBar的颜色。`fitsSystemWindows`是要系统把状态栏的动态和ToolBar相适应，状态栏会跟随ToolBar的颜色改变。

```xml
<android.support.v7.widget.Toolbar
    android:id="@+id/toolBar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="#0084eb"
    android:fitsSystemWindows="true"
    app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />
```


`fitSystemWindows`要点:

* 只把`fitSystemWindows`放在Layout的根布局上，状态栏就和根布局背景颜色一样。
* 同时在根布局和`ToolBar`上设置`fitSystemWindows`，效果跟随根布局，忽略`ToolBar`。
* ToolBar的`layout_height`不能用`?attr/actionBarSize`，只能`wrap_content`。


在`MainActivity.java`中进行初始化：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Toolbar toolbar = (Toolbar) findViewById(R.id.toolBar);
    setSupportActionBar(toolbar);
}
```

最后在`AndroidManifest.xml`里面把`Main_Activity`的主题换成我们自定义的`BarTheme`样式。

```
android:theme="@style/BarTheme"
```

因为使用其他主题时系统会自行添加一个ToolBar。如果我们又自定义一个，相当于有两个ToolBar，两个ToolBar的冲突会在编译时抛出：

```
01-11 19:16:01.596 19257-19257/com.phantomvk.statusbar E/AndroidRuntime: FATAL EXCEPTION: main
     Process: com.phantomvk.statusbar, PID: 19257
     java.lang.RuntimeException: Unable to start activity ComponentInfo{com.phantomvk.statusbar/com.phantomvk.statusbar.MainActivity}: java.lang.IllegalStateException: This Activity already has an action bar supplied by the window decor. Do not request Window.FEATURE_SUPPORT_ACTION_BAR and set windowActionBar to false in your theme to use a Toolbar instead.
         at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2702)
         at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2767)      
         ....
         ....
```

所以我们先把主题设置为`NoActionBar`，才在`onCreate`中`setSupportActionBar(toolbar)`里加入自定义的。

# 五、模拟器和物理机差别

我注意到Android5.0上物理机和虚拟机显示状态栏的效果不一样。网上也有一些开发者讨论了这个情况。因为手上刚好有Android5.0的三星S4，所以顺手测试了一下。

Android模拟器中状态栏的颜色是半透明，且颜色更深。
![img](/img/android/toolbar/emulator.png)

S4上就是完全透明，和ToolBar的颜色完全一致。
![img](/img/android/toolbar/s4.png)

因为现在的手机系统都经过定制，在模拟器上测试的结果，与物理机上测试未必一致。如果在主题样式显示差异有什么疑问，请务必要用多款物理机测试。


