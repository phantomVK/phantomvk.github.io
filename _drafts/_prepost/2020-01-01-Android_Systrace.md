---
layout:     post
title:      "Systrace的使用及分析"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---



在命令行上捕获系统跟踪信息

```
https://developer.android.com/studio/profile/systrace/command-line?hl=zh-cn
```

![sdk_location_icon](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/sdk_location_icon.png)



![sdk_location](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/sdk_location.png)

```bash
$ cd /Users/$(whoami)/Library/Android/sdk/platform-tools/systrace
```



```shell
export PATH=${PATH}:/Users/$(whoami)/Library/Android/sdk/platform-tools
export PATH=${PATH}:/Users/$(whoami)/Library/Android/sdk/tools
```



~/Library/Android/sdk/platform-tools/systrace

```bash
$ python systrace.py -o mynewtrace.html sched freq idle am wm gfx view binder_driver hal dalvik camera input res

Starting tracing (stop with enter)
Tracing completed. Collecting output...
Outputting Systrace results...
Tracing complete, writing results

Wrote trace HTML file: file:///Users/[用户名]/Library/Android/sdk/platform-tools/systrace/mynewtrace.html
```



上半部分

![systrace_0](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_0.png)

下半部分

![systrace_1](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_1.png)

筛选进程

![Processes](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/Processes.png)



![systrce_app_frames](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrce_app_frames.png)



![systrce_app](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrce_app.png)

![systrace_app_highlight](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_app_highlight.png)



按键盘M键标记区域

![systrace_app_highlight_mark](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_app_highlight_mark.png)



把视图缩小，看标记区域在整个记录的位置



![systrace_app_highlight_mark_global](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_app_highlight_mark_global.png)

![systrace_app_highlight_mark_global_colored](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_app_highlight_mark_global_colored.png)





![systrace_delay_time](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_delay_time.png)



![systrace_delay_reason](/Users/tanwenkang/github/phantomvk.github.io/img/android/Systrace/systrace_delay_reason.png)