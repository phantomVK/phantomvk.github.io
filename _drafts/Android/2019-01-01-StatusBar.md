---
layout:     post
title:      "StatusBar"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

```java
public int getStatusBarHeight(@NonNull Activity activity) {
    int id = activity.getResources().getIdentifier("status_bar_height", "dimen", "android");
    return (id > 0) ? getResources().getDimensionPixelSize(id) : 0;
}
```

```java
protected int getStatusBarHeight(@NonNull Activity activity) {
    Rect rect = new Rect();
    activity.getWindow().getDecorView().getWindowVisibleDisplayFrame(rect);
    return rect.top;
}
```

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        Window window = getWindow();
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.setStatusBarColor(Color.TRANSPARENT);
    }
    super.onCreate(savedInstanceState);
}
```

For MIUI

```java
public static void setStatusBarDarkMode(@NonNull Activity activity, boolean dark) {
    try {
        Class params = Class.forName("android.view.MiuiWindowManager$LayoutParams");
        Field field = params.getField("EXTRA_FLAG_STATUS_BAR_DARK_MODE");
        int flag = field.getInt(params);

        Window window = activity.getWindow();
        Class clazz = window.getClass();
        Method method = clazz.getMethod("setExtraFlags", int.class, int.class);
        method.invoke(window, dark ? flag : 0, flag);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

For Meizu

```java
public static void setStatusBarDarkMode(@NonNull Activity activity, boolean dark) {
    try {
        Window window = activity.getWindow();
        WindowManager.LayoutParams lp = window.getAttributes();

        Field darkFlag = WindowManager.LayoutParams.class.getDeclaredField("MEIZU_FLAG_DARK_STATUS_BAR_ICON");
        darkFlag.setAccessible(true);

        Field flags = WindowManager.LayoutParams.class.getDeclaredField("meizuFlags");
        flags.setAccessible(true);

        int bit = darkFlag.getInt(null);
        int value = flags.getInt(lp);
        if (dark) {
            value |= bit;
        } else {
            value &= ~bit;
        }

        flags.setInt(lp, value);
        window.setAttributes(lp);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

For Android 6.0+

```java
public static void setStatusBarDarkMode(@NonNull Activity activity, boolean dark) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        if (dark) {
            activity.getWindow().getDecorView().setSystemUiVisibility(
                    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
    }
}
```

