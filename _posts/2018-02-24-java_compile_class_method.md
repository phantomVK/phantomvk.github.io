---
layout:     post
title:      "方法的JIT编译"
date:       2018-02-24
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Java
---

Java方法执行一般会利用分层编译，先通过c1解释执行。方法执行编译等级逐渐提升，有机会通过JIT编译为特定平台汇编执行，以此获得最好的性能。

方法执行除了达到一定热度外，是否JIT编译也受到以下两个参数影响：

> -XX:+DontCompileHugeMethods

> -XX:HugeMethodLimit=8000

HotSpot默认不会编译巨型方法，也就是`-XX:+DontCompileHugeMethods`。通过修改参数为`-XX:-DontCompileHugeMethods`开启巨型方法编译。

判断方法是否为大对象由`-XX:HugeMethodLimit=8000`来决定，`8000`表示JIT编译字节码大小超过8000字节的方法就是巨型方法，这个阈值在产品版HotSpot里无法调整。

`DontCompileHugeMethods`和`HugeMethodLimit`默认值在`/hotspot/src/share/vm/runtime/globals.hpp`

```c
product(bool, DontCompileHugeMethods, true,
      "Do not compile methods > HugeMethodLimit")  

develop(intx, HugeMethodLimit,  8000,                                     
      "Don't compile methods larger than this if "                      
      "+DontCompileHugeMethods") 
```

下面看`JDK9`的`CompileTheWorld.java`，路径为

```bash
$ cd ./hotspot/src/jdk.internal.vm.compiler/share/classes/org.graalvm.compiler.hotspot/src/org/graalvm/compiler/hotspot
```

源码`canBeCompiled() Line 756`决定Java方法是否能被JIT编译。

```java
private boolean canBeCompiled(HotSpotResolvedJavaMethod javaMethod, int modifiers) {
    // 抽象方法或原生方法不编译
    if (Modifier.isAbstract(modifiers) || Modifier.isNative(modifiers)) {
        return false;
    }

    // 获取JVM配置参数
    GraalHotSpotVMConfig c = compiler.getGraalRuntime().getVMConfig();
    // 如果c.dontCompileHugeMethods为true，根据c.hugeMethodLimit判断该方法是否为巨型方法
    if (c.dontCompileHugeMethods && javaMethod.getCodeSize() > c.hugeMethodLimit) {
        println(verbose || methodFilters != null,
                        String.format("CompileTheWorld (%d) : Skipping huge method %s (use -XX:-DontCompileHugeMethods or -XX:HugeMethodLimit=%d to include it)",
                                        classFileCounter,
                                        javaMethod.format("%H.%n(%p):%r"),
                                        javaMethod.getCodeSize()));
        return false;
    }

    // dontinline标志开启不编译方法，用来排除可能有问题的目标方法
    // Allow use of -XX:CompileCommand=dontinline to exclude problematic methods
    if (!javaMethod.canBeInlined()) {
        return false;
    }

    // 注解类型为Snippet.class不编译
    for (Annotation annotation : javaMethod.getAnnotations()) {
        if (annotation.annotationType().equals(Snippet.class)) {
            return false;
        }
    }

    return true; // 经过判断可以被JIT编译
}
```

总的来说只要符合以下任一条件就不能被编译：

 1. 抽象方法或原生方法；
 2. 巨型方法；
 3. 有`dontinline`标志的方法；
 4. 注解类型为Snippet.class；

经过`canBeCompiled() Line 627`被判定可以被编译的方法送到`compileMethod() Line 725`等待编译。

```java
// Are we compiling this class?
MetaAccessProvider metaAccess = JVMCI.getRuntime().getHostJVMCIBackend().getMetaAccess();
if (classFileCounter >= startAt) {
    println("CompileTheWorld (%d) : %s", classFileCounter, className);

    // 循环遍历构造方法，判断能否JIT编译
    for (Constructor<?> constructor : javaClass.getDeclaredConstructors()) {
        HotSpotResolvedJavaMethod javaMethod = (HotSpotResolvedJavaMethod) metaAccess.lookupJavaMethod(constructor);
        if (canBeCompiled(javaMethod, constructor.getModifiers())) {
            compileMethod(javaMethod);
        }
    }

    // 循环遍历实例方法，判断能否JIT编译
    for (Method method : javaClass.getDeclaredMethods()) {
        HotSpotResolvedJavaMethod javaMethod = (HotSpotResolvedJavaMethod) metaAccess.lookupJavaMethod(method);
        if (canBeCompiled(javaMethod, method.getModifiers())) {
            compileMethod(javaMethod);
        }
    }

    // Also compile the class initializer if it exists
    // 类初始化器也是能被编译
    HotSpotResolvedJavaMethod clinit = (HotSpotResolvedJavaMethod) metaAccess.lookupJavaType(javaClass).getClassInitializer();
    if (clinit != null && canBeCompiled(clinit, clinit.getModifiers())) {
        compileMethod(clinit);
    }
}
```

目标方法`compileMethod()`，编译完成后下次执行时新编译方法替换旧方式。

```java
// Compiles a method and gathers some statistics.
private void compileMethod(HotSpotResolvedJavaMethod method, int counter) {
    try {
        long start = System.currentTimeMillis();
        long allocatedAtStart = MemUseTrackerImpl.getCurrentThreadAllocatedBytes();
        int entryBCI = JVMCICompiler.INVOCATION_ENTRY_BCI;
        HotSpotCompilationRequest request = new HotSpotCompilationRequest(method, entryBCI, 0L);
        // For more stable CTW execution, disable use of profiling information
        boolean useProfilingInfo = false;
        boolean installAsDefault = false;
        // 创建一个编译任务
        CompilationTask task = new CompilationTask(jvmciRuntime, compiler, request, useProfilingInfo, installAsDefault);
        // 开始编译任务
        task.runCompilation();

        // Invalidate the generated code so the code cache doesn't fill up
        HotSpotInstalledCode installedCode = task.getInstalledCode();
        if (installedCode != null) {
            installedCode.invalidate();
        }

        memoryUsed.getAndAdd(MemUseTrackerImpl.getCurrentThreadAllocatedBytes() - allocatedAtStart);
        compileTime.getAndAdd(System.currentTimeMillis() - start);
        compiledMethodsCounter.incrementAndGet();
    } catch (Throwable t) {
        // Catch everything and print a message
        println("CompileTheWorld (%d) : Error compiling method: %s", counter, method.format("%H.%n(%p):%r"));
        printStackTrace(t);
    }
}
```


### 参考链接

[Java中一个方法字节码的长度为什么会影响程序并发下的性能？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/263322849/answer/268228465)

[-XX:-DontCompileHugeMethods - BLOG OF JEROEN LEENARTS](https://leenarts.net/2010/05/26/dontcompilehugemethods/)

