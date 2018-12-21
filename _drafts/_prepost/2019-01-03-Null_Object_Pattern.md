---
layout:     post
title:      "空对象模式"
date:       2019-01-03
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Architectural Pattern
---

空对象模式指通过特定的、没有负载有效数据的空对象，表示实例不存在的默认状态。可用在不能接受 __null__ 的逻辑，或者利用空对象实现区别于实例为空的应用场景。

这是一般场景的用户模型，包含用户id和昵称：

```java
class User(private val userId: String, private val nickname: String) {

    fun getUserId(): String {
        return userId
    }

    fun getRemark(): String {
        return nickname
    }
}
```

实现空对象的形式有很多，比较正式的实现方法是：子类继承父类并重写所有可见成员方法，通过抛出异常的方式，阻止代码调用空对象方法。这样做的目的是强调不要通过空对象获取无效数据。这种方式在父类有很多可重写方法的场景会略显麻烦，可自行约定对空对象的使用方式。

```java
class UserNull() : User("", "", "") {

    // 重写父类方法，禁止通过空对象实例获取无效数据
    override fun getUserId(): String {
        throw IllegalAccessException("Access to this method is denied.")
    }

    override fun getRemark(): String {
        throw IllegalAccessException("Access to this method is denied.")
    }
}
```

当然，父类可(根据实际)允许子类重写部分父类方法，以下是 __User__ 的方法使用open修饰允许重写。

```java
open class User(private val userId: String,
                private val roomId: String,
                private val nickname: String) {

    open fun getUserId(): String {
        return userId
    }

    open fun getRemark(): String {
        return nickname
    }
   
    // 同时增加此方法判断实例是否为空对象
    fun isNullObject(): Boolean {
        return this == User.USER_NULL_OBJ
    }

    companion object {
        // 定义单例空对象，避免创建多余实例
        val USER_NULL_OBJ = UserNull()
    }
}
```

对于 __RxJava__ 这种不能接受 __null__ 对象的应用场景来说，使用空对象表达 __null__ 就最合适不过了。查不到用户时使用 __空对象__ 代替 __null__，既利用 __filter__ 提前结束操作，又避免 __fromCallable__ 返回 __null__ 引起空指针异常堆栈跟踪的性能损耗。

```java
Observable.fromCallable { UserDao.load(userId) ?: User.USER_NULL_OBJ }
        .filter { !it.isNullObject() }
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe { removeUser(it.userId) }
```

