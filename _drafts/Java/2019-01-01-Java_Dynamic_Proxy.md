---
layout:     post
title:      "Java动态代理"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

动态代理需要把所有行为抽象到接口中

```java
interface IUser{
    fun getUserId(): String

    fun getRoomId(): String

    fun getRemark(): String
}
```

```java
class User(private val userId: String,
           private val roomId: String,
           private val nickname: String) : IUser {

    override fun getUserId(): String {
        return userId
    }

    override fun getRoomId(): String {
        return roomId
    }

    override fun getRemark(): String {
        return nickname
    }
}
```

```java
class Handler : InvocationHandler {
    override fun invoke(proxy: Any, method: Method, args: Array<out Any>): Any {
        throw IllegalAccessException("Access to this method is denied.")
    }
}
```

```java
class User(private val userId: String,
           private val roomId: String,
           private val nickname: String) : IUser {

    .....

    companion object {
        val defUser by lazy {
            Proxy.newProxyInstance(
                    IUser::class.java.classLoader,
                    arrayOf(IUser::class.java),
                    Handler()) as IUser
        }
    }
}
```

```java
object ProxyRunner {
    @JvmStatic fun main(args: Array<String>) {
        User.defUser.getRemark()
    }
}
```

```
Exception in thread "main" java.lang.reflect.UndeclaredThrowableException
	at com.sun.proxy.$Proxy0.getRemark(Unknown Source)
	at proxy.ProxyRunner.main(ProxyRunner.java:6)
Caused by: java.lang.IllegalAccessException: Access to this method is denied.
	at proxy.Handler.invoke(Handler.kt:8)
	... 2 more
```

