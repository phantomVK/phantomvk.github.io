---
layout:     post
title:      "Default interface methods are only supported starting with Android N"
date:       2019-06-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

依赖旧版本：

```
implementation 'androidx.appcompat:appcompat:1.1.0-alpha04'
implementation 'androidx.core:core-ktx:1.1.0-alpha05'
```

更新为以下版本：

```
implementation 'androidx.appcompat:appcompat:1.1.0-beta01'
implementation 'androidx.core:core-ktx:1.2.0-alpha02'
```

构建时出现错误：

```
Error: Default interface methods are only supported starting with Android N (--min-api 24): android.view.MenuItem androidx.core.internal.view.SupportMenuItem.setContentDescription(java.lang.CharSequence)
```

按照以下步骤，先点击 __Android Studio__ 右上角 __Project Structure__ 图标(右二)

![project_structure_icon](/img/android/BuildError/project_structure_icon.jpg)

在该设置中选择 __app__ 模块，把 __Source Compatibility__ 和 __Target Compatibility__ 均设置为 __1.8__

![project_structure](/img/android/BuildError/project_structure.jpg)

当然，也可以手动在 __build.gradle(Module:app)__ 中添加代码：

```groovy
 compileOptions {
    sourceCompatibility = '1.8'
    targetCompatibility = '1.8'
}
```

这是添加完成后的效果，继续编译即可

![compileOptions](/img/android/BuildError/compileOptions.jpg)
