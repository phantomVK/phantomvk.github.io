---
layout:     post
title:      "原型模式"
date:       2019-01-22
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Architectural Pattern
---

__原型模式(Prototype Pattern)__，用于创建重复对象的设计模式。设计模式分三大类：__创建型模式__、__结构型模式__、__行为型模式__，而原型模式则属于 __创建型模式__。

原型模式适合用在创建对象成本较高的场景。因为频繁地直接构建对象，会对运行时系统造成压力。通过原型模式并 __缓存__ 初次构建的对象，以后需要类似对象时从缓存获取对象母版并克隆。克隆获取副本的操作，避免访问压力直接打到原始对象构建点上。

或者，当构建原始对象参数较多时，需要的新对象和原型对象差异不大，创建新对象后需逐一拷贝原参数的操作较麻烦。使用原型模式后，可在获取的新对象上修改目标数据成员。

在 __Java__ 语言里，基类已包含 __clone()__ 方法。根据实际需要，决定对新对象实现的是深克隆还是浅克隆，避免新对象共享数据时出现问题。

在 [仿Android微信消息气泡](/2018/09/17/wechat-bubble-shape/) 一文也使用了原型模式为相同气泡创建两层不同的交互颜色控制实例。下面把相关的代码抽出展示。不过原型模式包含一个缓存原始对象的列表，而下面的实例根据实际省略了。

类名为 __BubbleShape__ 的气泡样式类，里面保存了绘制气泡需要的参数和绘制方法(省略)。

```java
class BubbleShape constructor(var arrowDirection: DIRECTION,
                              @ColorInt var solidColor: Int,
                              @ColorInt var strokeColor: Int,
                              var strokeWidth: Int,
                              var cornerRadius: Int,
                              var arrowWidth: Int,
                              var arrowHeight: Int,
                              var arrowMarginTop: Int) : Shape(){
    ......

    // Shallow clone as a new object.
    override fun clone(): BubbleShape = super.clone() as BubbleShape
}
```

成员方法 __clone()__ 重写了父类 __Shape__ 继承来的接口 __Cloneable__。实现此接口后实例就具有被克隆的能力。

```java
public abstract class Shape implements Cloneable {
    private float mWidth;
    private float mHeight;

    ......
}
```

由于实现子类中使用的数据成员大部分为基本类型，而数据成员为类的字段也可在不同颜色控制对象中复用，所以重写 __clone()__ 方法时用浅克隆返回实例。

获取新副本的地方只需要如下调用，从 __shape__ 获取拷贝对象 __darkShape__，修改目标变量。

```java
val darkShape = shape.clone()
darkShape.solidColor = darkColor
```

如有需要，增加缓存列表对象 __darkShape__。