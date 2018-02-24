---
layout:     post
title:      "方法的JIT编译"
date:       2018-02-24
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - JVM
---

方法是否被JVM进行JIT编译受到以下两个参数影响：

> -XX:+DontCompileHugeMethods

> -XX:HugeMethodLimit=8000

虚拟机默认不会编译巨型方法，也就是`-XX:+DontCompileHugeMethods`。因此通过`-XX:-DontCompileHugeMethods`来支持巨型方法编译。

判断方法是否为大对象由`-XX:HugeMethodLimit=8000`来决定，`8000`表示JIT编译字节码大小超过8000字节的方法就是巨型方法，且这个阈值在产品版HotSpot里无法调整。

下面根据`JDK9`来看相关逻辑在`CompileTheWorld.java`，路径为

```bash
$ cd ./jdk9/hotspot/src/jdk.internal.vm.compiler/share/classes/org.graalvm.compiler.hotspot/src/org/graalvm/compiler/hotspot
```

`canBeCompiled()`决定Java方法是否被JIT编译，代码在756行。这个方法返回Boolean来表示一个方法是否需要编译。

```java
private boolean canBeCompiled(HotSpotResolvedJavaMethod javaMethod, int modifiers) {
    // 如果是抽象方法或原生方法，不能被编译
    if (Modifier.isAbstract(modifiers) || Modifier.isNative(modifiers)) {
        return false;
    }
    // 获取JVM配置参数
    GraalHotSpotVMConfig c = compiler.getGraalRuntime().getVMConfig();
    // 如果c.dontCompileHugeMethods为true，根据c.hugeMethodLimit
    // 判断该方法是否为巨型方法，因为该情况下巨型方法是不被编译的.
    if (c.dontCompileHugeMethods && javaMethod.getCodeSize() > c.hugeMethodLimit) {
        println(verbose || methodFilters != null,
                        String.format("CompileTheWorld (%d) : Skipping huge method %s (use -XX:-DontCompileHugeMethods or -XX:HugeMethodLimit=%d to include it)", classFileCounter,
                                        javaMethod.format("%H.%n(%p):%r"),
                                        javaMethod.getCodeSize()));
        return false;
    }

    // dontinline标志开启也不会编译方法，可以通过Java注解实现
    // Allow use of -XX:CompileCommand=dontinline to exclude problematic methods
    if (!javaMethod.canBeInlined()) {
        return false;
    }

    // 注解包含@Snippets也不编译
    // Skip @Snippets for now
    for (Annotation annotation : javaMethod.getAnnotations()) {
        if (annotation.annotationType().equals(Snippet.class)) {
            return false;
        }
    }
    // 经过上面判断的方法可以被JIT编译
    return true;
}
```

总的来说只要符合以下任一条件就不能被编译：

 1. 抽象方法或原生方法；
 2. 根据方法字节码大小确认为巨型方法；
 3. 有dontinline标志的方法；
 4. 方法注解包含@Snippets；

`canBeCompiled()`在在627行代码块中被调用。如果方法被判定可以被编译，方法会被送到`compileMethod()`等待编译。

```java
// Are we compiling this class?
MetaAccessProvider metaAccess = JVMCI.getRuntime().getHostJVMCIBackend().getMetaAccess();
if (classFileCounter >= startAt) {
    println("CompileTheWorld (%d) : %s", classFileCounter, className);

    // Compile each constructor/method in the class.
    // 这里开始循环遍历构造方法，并判断是否需要JIT编译
    for (Constructor<?> constructor : javaClass.getDeclaredConstructors()) {
        HotSpotResolvedJavaMethod javaMethod = (HotSpotResolvedJavaMethod) metaAccess.lookupJavaMethod(constructor);
        if (canBeCompiled(javaMethod, constructor.getModifiers())) {
            compileMethod(javaMethod);
        }
    }
    // 这里开始循环遍历实例方法，并判断是否需要JIT编译
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

实际方法编译在725行`compileMethod()`

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

文中讨论的两个参数的默认值在`jdk/jdk9/hotspot/src/share/vm/runtime/globals.hpp`

```c
product(bool, DontCompileHugeMethods, true,
      "Do not compile methods > HugeMethodLimit")  

develop(intx, HugeMethodLimit,  8000,                                     
      "Don't compile methods larger than this if "                      
      "+DontCompileHugeMethods") 
```

### 参考链接

[Java中一个方法字节码的长度为什么会影响程序并发下的性能？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/263322849/answer/268228465)

[-XX:-DontCompileHugeMethods - BLOG OF JEROEN LEENARTS](https://leenarts.net/2010/05/26/dontcompilehugemethods/)

