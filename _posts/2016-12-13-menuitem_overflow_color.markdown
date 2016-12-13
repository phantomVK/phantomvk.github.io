---
layout:     post
title:      "Android MenuItem 背景颜色"
date:       2016-12-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

ToolBar上添加Menu，除了搜索图标，其他都收起来。修改完ToolBar背景色才发现MenuItem里面的背景色没有改变，还是原来白色。

![img](/img/android/menuitem/toolbar.jpg)

打开菜单后

![img](/img/android/menuitem/overflow.jpg)


原来`res/values/styles/xml`的配置

```xml
<resources>

    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="colorPrimary">@color/colorWeChatGray</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>

</resources>
```

修改后

```xml
<resources>

    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="colorPrimary">@color/colorWeChatGray</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="actionOverflowMenuStyle">@style/optionMenu</item>
    </style>

    <style name="optionMenu" parent="Widget.AppCompat.ActionBar.Overflow">
        <item name="android:popupBackground">@color/colorWeChatGray</item>
    </style>
    
</resources>
```

这样MenuItem的背景颜色就能修改了

![img](/img/android/menuitem/bg_overflow.jpg)

但这看起来明显不对：

* 菜单不能ToolBar的图标遮住
* 背景色改变后和文字颜色不协调

____

我们先处理文字的颜色，添加`android:textColor`和对应的颜色即可

```xml
<resources>

    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorWeChatGray</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="actionOverflowMenuStyle">@style/optionMenu</item>
        <item name="android:textColor">@color/colorWhite</item>
    </style>

    <style name="optionMenu" parent="Widget.AppCompat.ActionBar.Overflow">
        <item name="android:popupBackground">@color/colorWeChatGray</item>
    </style>

</resources>
```

![img](/img/android/menuitem/bg_text_overflow.jpg)

剩下遮蔽图标的问题比较好解决，把`<style name="optionMenu" parent="Widget.AppCompat.ActionBar.Overflow">`的`.Overflow`去掉

![img](/img/android/menuitem/bg_text_good.png)







