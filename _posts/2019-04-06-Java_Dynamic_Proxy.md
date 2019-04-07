---
layout:     post
title:      "Java动态代理"
date:       2019-04-06
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

在之前[空对象模式](/2019/01/01/Null_Object_Pattern/)一文中，讨论约束空对象方法调用时，提到可以使用 __动态代理__ 实现。复用前文的数据模型，现在就来实现这个方式。而动态代理的具体运行逻辑详情，将在以后文章单独进行源码剖析。

和静态代理一样，动态代理也需要把所有行为抽象化，于是把以前写在 __User__ 的行为全部抽象到接口 __IUser__。

```kotlin
interface IUser{
    fun getUserId(): String

    fun getRoomId(): String

    fun getRemark(): String
}
```

然后 __User__ 模型实现该抽象接口

```kotlin
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

实现动态代理的关键是实现 __InvocationHandler__。使用动态代理，是为了把所有方法代理到同一实例中，所以实现 __InvocationHandler__ 时还需要提供有参构造方法，让外部传入接收委托的实例。

不过，这次并不需要委托任何行为，而只是通过动态代理这个中转，以抛出异常的方式阻止实例里所有方法的调用。由以下代码可见，该中介类无需保存被委托类实例，只是在实现方法里抛异常阻止调用。

```kotlin
class Handler : InvocationHandler {
    // 实现唯一的抽象方法，SAM，Single abstract mathod.
    override fun invoke(proxy: Any, method: Method, args: Array<out Any>): Any {
        // 目的就是抛出异常
        throw IllegalAccessException("Access to this method is denied.")
    }
}
```

实现中介类 __InvocationHandler__ 之后就可以创建实例。按照惯例，空对象单例对象一般和其模型放在一起。

```kotlin
class User(private val userId: String,
           private val roomId: String,
           private val nickname: String) : IUser {

    .....

    companion object {
        val defUser by lazy {
            // 创建空对象实例，这个实例里所有方法已被代理
            Proxy.newProxyInstance(
                    IUser::class.java.classLoader,
                    arrayOf(IUser::class.java),
                    Handler()) as IUser
        }
    }
}
```

这个被代理的实例使用方式和普通对象无异

```kotlin
object ProxyRunner {
    @JvmStatic fun main(args: Array<String>) {
        User.defUser.getRemark()
    }
}
```

只在运行时走代理逻辑，然后主动抛出预定实现的异常：

```
Exception in thread "main" java.lang.reflect.UndeclaredThrowableException
	at com.sun.proxy.$Proxy0.getRemark(Unknown Source)
	at proxy.ProxyRunner.main(ProxyRunner.java:6)
Caused by: java.lang.IllegalAccessException: Access to this method is denied.
	at proxy.Handler.invoke(Handler.kt:8)
	... 2 more
```

使用动态代理的好处是，后来人再增加实现方法时，无需修改已有实现逻辑，就能阻止代码运行时调用空对象方法。