---
layout:     post
title:      "Android源码系列(8) -- LayoutInflater"
date:       2018-03-03
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、基础用法

`inflate()`把传入的layout_res_id获得构建后的实例：

```java
val view = LayoutInflater.from(this).inflate(R.layout.layout, null)
```

先了解源码中几个频繁出现的`TAG_*`字符串：

```java
TAG_MERGE = "merge";
TAG_INCLUDE = "include";
TAG_1995 = "blink";
TAG_REQUEST_FOCUS = "requestFocus";
TAG_TAG = "tag";
```

## 二、方法入口

`inflate()`为指定的xml资源文件构建视图布局

```java
// resource: xml布局文件id 
// root: 根布局
// attachToRoot: 是否加入到父布局
public View inflate(@LayoutRes int resource,
                    @Nullable ViewGroup root,
                    boolean attachToRoot) {
    // 从Context获取Resources
    final Resources res = getContext().getResources();

    // 用Resources创建xml资源语义解析器，解析layout布局
    final XmlResourceParser parser = res.getLayout(resource);

    try {
        // 返回创建的View
        return inflate(parser, root, attachToRoot);
    } finally {
        // 关闭该语义解析器
        parser.close();
    }
}
```

下方方法的注释着重提到几点:

1. 填充行为重度依赖编译过程预处理过的xml文件，以此提升查找性能；
2. 运行期不可能用`XmlPullParser`调用`inflate`方法处理普通xml；
3. 普通xml文件指开发可见的xml，此类文件没有经过预处理，所以运行期不能处理。

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    // mConstructorArgs作为同步对象，LayoutInflate实例同步执行
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        
        // mContext就是LayoutInflater(this)中的context.
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);

        // mConstructorArgs[0]临时保存为lastContext
        Context lastContext = (Context) mConstructorArgs[0];
        // 用inflaterContext，即mContext赋值给mConstructorArgs[0]
        mConstructorArgs[0] = inflaterContext;

        View result = root;

        try {
            // 开始查找Root节点
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                    // 跳过非START_TAG或END_DOCUMENT标签
            }
            
            // 可能已找到START_TAG，也可能没有START_TAG而结束；
            if (type != XmlPullParser.START_TAG) {
                // 没有发现START_TAG意味xml内容非法，上层捕捉异常终止inflate
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            
            // 有START_TAG则解析根布局名
            final String name = parser.getName();
            
            if (TAG_MERGE.equals(name)) {
                // merge只能在根布局非空且attachToRoot为true时使用
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                
                // 循环遍历子标签
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 不是merge表明该标签是xml文件的根元素，命名为temp
                // 例如name为"LinearLayout"，则temp是LinearLayout实例
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;
                
                // 传进来的root不为空
                if (root != null) {
                    // 根据root构建temp的LayoutParams
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // attachToRoot为false，从root获得LayoutParams并设到根元素temp
                        temp.setLayoutParams(params);
                    }
                }

                // 填充temp下所有子View.
                rInflateChildren(parser, temp, attrs, true);

                // xml构建的View添加到root中
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // 如果temp不需要添加到root，则返回temp
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // 在context中不保留静态引用
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        
        // root非空且attachToRoot为true时构建temp添加到root，返回root;
        // 否则返回temp，temp可能设置了有关root的LayoutParams.
        return result;
    }
}
```

## 三、递归填充子视图

此方法用于递归填充非根的内部子视图。父context作为填充context，调用方法 __rInflate__

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

## 四、填充子视图

__r__ 原意为 __recursive__，深度递归xml布局初始化view，一并初始化此view的子view。

```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    // 获取树最大深度
    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    // 没遇到views本身的END_TAG或遍历子views时没遇到END_DOCUMENT就继续遍历
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();
        
        // 此View设置了TAG_REQUEST_FOCUS
        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            // 解析View的tag
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            // include仅能作为子元素
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs); // 解析include标签
        } else if (TAG_MERGE.equals(name)) {
            // 子视图不能是merge标签，merge只能用在根元素
            throw new InflateException("<merge /> must be the root element");
        } else {
            // 通过createViewFromTag构建标签所指示的View
            final View view = createViewFromTag(parent, name, context, attrs);
            // 利用parent作为ViewGroup，构建出LayoutParams给子View使用
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            // rInflateChildren()会调用rInflate()，深度遍历初始化
            rInflateChildren(parser, view, attrs, true);
            // 构建成功的View添加到ViewGroup
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```


## 五、从tag构建视图

根据标签名创建view

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // ignoreThemeAttr为false则给context配置主题装饰器
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }
    
    // TAG_1995返回BlinkLayout，即上世纪Disco舞厅灯光的闪烁感
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                // 不包含符号'.'表示原生View，不是自定义如 <com.phatomvk.custom.view />
                // onCreateView()调用createView(name, prefix = "android.view.", attrs)
                // 示例：android.support.v7.widget.RecyclerView不会添加前缀
                if (-1 == name.indexOf('.')) {
                    // 例：TextView全路径名为'android.view.TextView'
                    view = onCreateView(parent, name, attrs);
                } else {
                    // 自定义View创建
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
        
        return view; // 返回构建的view
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    }
}
```

## 六、视图创建

全局静态HashMap缓存，缓存View的构造方法
```java
private static final HashMap<String, Constructor<? extends View>> sConstructorMap =
        new HashMap<String, Constructor<? extends View>>();
```

View最终通过其全路径名在`ClassLoader`中反射出对应的View

```java
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    
    // 构建方法缓存HashMap
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }

    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
        
        if (constructor == null) { // 该类没有缓存
            // 从ClassLoader中反射类
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
                    
            // 未经许可的类不能实例化，并抛出异常
            if (mFilter != null && clazz != null) {
                // 指定类不允许被填充构建
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }

            // 获取反射类的构造方法，并把构造方法添加到缓存
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // 已经缓存的构造方法还要经过mFilter的检查
            if (mFilter != null) {
                // 查看此name之前是否遇见过
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // 获取类并记录此类是否允许使用
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);

                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }

        Object lastContext = mConstructorArgs[0];
        if (mConstructorArgs[0] == null) {
            mConstructorArgs[0] = mContext;
        }
        Object[] args = mConstructorArgs;
        args[1] = attrs;
        
        // 创建类实例，args是自定义主题相关变量
        final View view = constructor.newInstance(args);

        // 类型是ViewStub
        if (view instanceof ViewStub) {
            // 使用同一个Context给ViewStub设置LayoutInflater
            // 后面View填充是会使用此LayoutInflater
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        mConstructorArgs[0] = lastContext;
        return view;

    } catch (NoSuchMethodException e) {
        // 无法找到构造方法
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (ClassCastException e) {
        // 填充类不是View的子类
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Class is not a View " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } catch (ClassNotFoundException e) {
        // 类加载失败，抛出异常
        throw e;
    } catch (Exception e) {
        final InflateException ie = new InflateException(
                attrs.getPositionDescription() + ": Error inflating class "
                        + (clazz == null ? "<unknown>" : clazz.getName()), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

## 七、验证类加载器

```java
private final boolean verifyClassLoader(Constructor<? extends View> constructor) {
    final ClassLoader constructorLoader = constructor.getDeclaringClass().getClassLoader();
    if (constructorLoader == BOOT_CLASS_LOADER) {
        // boot class loader是合法的类加载器
        return true;
    }
    // in all normal cases (no dynamic code loading), we will exit the following loop on the
    // first iteration (i.e. when the declaring classloader is the contexts class loader).
    ClassLoader cl = mContext.getClassLoader();
    do {
        if (constructorLoader == cl) {
            return true;
        }
        cl = cl.getParent();
    } while (cl != null);
    return false;
}
```

## 八、总结

整个过程最耗时间部分有两个：

- 读取并分析xml标签数据；
- 类反射获得全路径名的View实例；

从开始`LayoutInflater.inflate`填充布局，调用`rInflate`递归遍历子View，每个子View在`createViewFromTag`通过全限定名调用`createView`反射实例，层层处理结束后返回视图。

最后总结`attachToRoot`:

- `root`为null，`attachToRoot`设置值不起作用；
- `root`不为null，`attachToRoot`为true，给加载的布局文件指定父布局root；
- `root`不为null，`attachToRoot`为false，则设置布局文件最外层layout属性。当该view添加到父view时，这些layout属性起效；
- 不设置`attachToRoot`参数且root不为null，`attachToRoot`参数默认为true。

