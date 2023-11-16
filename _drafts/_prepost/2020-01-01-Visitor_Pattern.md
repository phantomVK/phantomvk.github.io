---
layout:     post
title:      ""
date:       2023-11-26
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - 设计模式
---

```java
public interface Visitable {
    void accept(Visitor visitor);
}
```





分别可以访问汽车的三个部件：引擎、剥离、轮胎

```java
public class Engine implements Visitable {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

```java
public class Glass implements Visitable {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

```java
public class Wheel implements Visitable {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```



定义汽车类，内部包含上面定义的汽车部件，并接受访问者的访问

```java
public class Car {
    private final List<Visitable> parts = new ArrayList<>();

    public void addPart(Visitable part) {
        parts.add(part);
    }

    public void accept(Visitor visitor) {
        for (Visitable part : parts) {
            part.accept(visitor);
        }
    }
}
```



定义访问者访问汽车部件的行为接口

```java
public interface Visitor {
    void visit(Engine engine);

    void visit(Glass glass);

    void visit(Wheel wheel);
}
```



实现访问者的行为，对工人来说是修理汽车的部件

```java
public class Worker implements Visitor {
    @Override
    public void visit(Engine engine) {
        System.out.println("Worker is repairing engine.");
    }

    @Override
    public void visit(Glass glass) {
        System.out.println("Worker is repairing glass.");
    }

    @Override
    public void visit(Wheel wheel) {
        System.out.println("Worker is repairing wheel.");
    }
}
```



实现访问者的行为，对销售来说是介绍汽车的部件

```java
public class Seller implements Visitor {
    @Override
    public void visit(Engine engine) {
        System.out.println("Seller is introducing engine.");
    }

    @Override
    public void visit(Glass glass) {
        System.out.println("Seller is introducing glass.");
    }

    @Override
    public void visit(Wheel wheel) {
        System.out.println("Seller is introducing wheel.");
    }
}
```

```java
public class VisitorClass {
    public static void main(String[] args) {
        Car car = new Car();
        car.addPart(new Engine());
        car.addPart(new Glass());
        car.addPart(new Wheel());

        Visitor worker = new Worker();
        car.accept(worker);

        Seller seller = new Seller();
        car.accept(seller);
    }
}
```

```java
Worker is repairing engine.
Worker is repairing glass.
Worker is repairing wheel.

Seller is introducing engine.
Seller is introducing glass.
Seller is introducing wheel.
```

