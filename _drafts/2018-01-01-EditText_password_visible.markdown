---
layout:     post
title:      "EditText密码显示和隐藏"
date:       2018-01-01
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

显示`EditText`内容为明文

```java
setTransformationMethod(HideReturnsTransformationMethod.getInstance());
```

显示`EditText`内容为密文

```java
setTransformationMethod(PasswordTransformationMethod.getInstance());
```

实现被点击图标`OnClickListener`

```kotlin
icon.setOnClickListener {
    editText.apply {
        if (inputType == InputType.TYPE_TEXT_VARIATION_VISIBLE_PASSWORD) {
            inputType = InputType.TYPE_TEXT_VARIATION_PASSWORD
            transformationMethod = PasswordTransformationMethod.getInstance()
            icon.setImageDrawable(ContextCompat.getDrawable(this@MainActivity, R.drawable.ic_eyes_closed))
            setSelection(text.length)
        } else {
            inputType = InputType.TYPE_TEXT_VARIATION_VISIBLE_PASSWORD
            transformationMethod = HideReturnsTransformationMethod.getInstance()
            icon.setImageDrawable(ContextCompat.getDrawable(this@MainActivity, R.drawable.ic_eyes))
            setSelection(text.length)
        }
    }
}
```


