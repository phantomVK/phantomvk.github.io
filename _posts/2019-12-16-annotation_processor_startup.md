---
layout:     post
title:      "Android注解处理器入门"
date:       2019-12-16
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

#### 基础工程

先创建名为 __AnnotationDemo__ 工程，如果已有工程可直接跳过：

![create_project](/img/android/annotation_processor/create_project.png)

#### 注解类组件

工程内新建名为 __annotation__ 的模块。

![annotaion_module](/img/android/annotation_processor/annotaion_module.png)

此模块存放所有注解类，示例注解类名名为 __XAnnotation__。

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface XAnnotation {
}
```

因为此注解可用于修饰类，所以目标类型为：__ElementType.TYPE__，并且注解信息在运行时保留。

#### 注解处理器组件

另外新建模块 __annotation_processor__ 用于存放注解处理器。存放注解处理器的模块必须和注解类模块分离，不然上层模块依赖后会出现问题。

![processor_module](/img/android/annotation_processor/processor_module.png)

创建之后模块布局就是这样：

![modules_created](/img/android/annotation_processor/modules_created.png)

然后，在模块里添加 __javapoet__ 或 __kotlinpoet__ 依赖，推荐使用 __javapoet__。还要导入上述创建的 __annotation__ 模块，这样注解处理器就能识别我们声明的所有注解类。


```groovy
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.squareup:javapoet:1.11.1'
    implementation project(":annotation")
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

 __javapoet__ 或 __kotlinpoet__ 的功能是通过代码编写，在编译前用预设代码拼接出新的类。根据名字就知道他们分别能生成java代码和kotlin代码。

创建注解处理器：

```java
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@SupportedAnnotationTypes({"com.phantomvk.annotation.XAnnotation"})
public class XProcessor extends AbstractProcessor {

    private Filer mFiler;
    private Messager mMessager;
    private Elements mElementUtils;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnvironment.getFiler();
        mMessager = processingEnvironment.getMessager();
        mElementUtils = processingEnvironment.getElementUtils();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(XAnnotation.class.getCanonicalName());
        return types;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.RELEASE_8;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 返回false编译不会通过，必须把输入的代码输出到目标目录
        return false;
    }
}
```

继续完善 __process()__ 的逻辑：

```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
    Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(XAnnotation.class);

    for (Element element : elements) {
        PackageElement packageElement = mElementUtils.getPackageOf(element);
        String packageName = packageElement.getQualifiedName().toString();

        MethodSpec methodSpec = MethodSpec.methodBuilder("showLog") // 方法名
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC) // 声明为公开的静态方法
                .returns(void.class) // 返回值为空
                .addStatement("$T.out.println($S)", System.class, "Hello JavaPoet.") // 方法体
                .build();

        TypeSpec typeSpec = TypeSpec.classBuilder("XGeneratedClass") // 类名
                .addModifiers(Modifier.PUBLIC) // 类可见性为public
                .addMethod(methodSpec) // 把上述方法添加到类，可继续添加
                .build();

        // 用构建完成的类生成Java文件
        JavaFile javaFile = JavaFile.builder(packageName, typeSpec).build();

        try {
            // 结果写入磁盘
            javaFile.writeTo(mFiler);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    return true;
}
```

注册上述注解处理器：

![register_annotation_processor](/img/android/annotation_processor/register_annotation_processor.png)

注册声明文件必须按照上述目录存放而且有文件名：

```
javax.annotation.processing.Processor
```

文件内容为注解处理器的全路径名：

![register_annotation_processor_ide](/img/android/annotation_processor/register_annotation_processor_ide.png)

只有正确注册的注解处理器，编译器才能在编译过程处理注解并执行注解逻辑。

#### 导入

最后在App工程导入注解声明库和注解处理器库：

```groovy
apply plugin: 'kotlin-kapt'

// JavaPoet required Java8
android {
    compileOptions {
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }
}

dependencies {
    // ....
    implementation project(":annotation")
    kapt project(":annotation_processor") // annotationProcessor for Java
}
```

导入之后就可以用声明好的注解类修饰 __MainActivity__：

```kotlin
@XAnnotation
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

点击一下构建，就会按照注解类能获取的信息生成新的类 __XGeneratedClass__：

![generated_class](/img/android/annotation_processor/generated_class.png)

同时项目也能引用生成类：

```kotlin
@XAnnotation
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        XGeneratedClass.showLog() // 直接引用
    }
}
```

#### 执行结果

运行之后控制台就能打印：

```
com.phantomvk.annotationdemo I/System.out: Hello JavaPoet.
```

上述文章完成声明和开发注解处理器最基本的方式，后续将在此基础上继续增强注解处理器的能力。

完整工程源码：[phantomVK/AnnotationDemo](https://github.com/phantomVK/AnnotationDemo)