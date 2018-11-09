---
layout:     post
title:      "外观设计模式探索"
date:       2017-10-15
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Architectural Pattern
---

## 前言

移动端开发中，经常遇到通用逻辑加载：例如IM中个人头像加载。在应用开发初期，由于缺乏把所有加载逻辑整合到一起的思维，导致后期头像逻辑变化时，只能用搜索找出相关的代码并修改。即使代码分散的地方比较少，也经常出现遗漏修改。

头像加载这种统一的加载逻辑，必须整合到一个类中，并做好抽象，以便应对将来的需求变更。思维拓展开来，用户名获取、应用配置读取、读取Session等多层次数据获取(Repository模式)都能从中受益。下面基于IM头像加载的实战，总结出相关经验。

## 一、设计思想

总结实战后接口设计的三个基本目的：

1. 通过接口定义指明实现的具体方向，明确接口的功能范围;
2. 对调用屏蔽实现：调用方只需了解接口方法的功能，而不需要了解具体实现;
3. 抽象需要预留一定的拓展性：如个人头像接口设计时，考虑群头像和个人头像交互;

## 二、接口功能

产品开发过程要满足以下产品需求：

1. 用`userId`即可把用户头像加载到`ImageView`中;
2. 能指定加载来源的`filePath`，不仅根据`userId`;
3. `userId`对应的`url`由指定规则拼接获得；
4. 群头像由四个用户的个人头像拼接，个人头像为群头像拼装提供Bitmap.

根据上述需求可以整理以下抽象接口，这些接口虽然已经投入到生产环境，但设计不一定合理，还在进一步演进，请酌情参考。方法具体能力在注释中已经写得很明确，不再复述。

```java
// Global avatar loading facade interface.
public interface IUserAvatarLoader {

    // Load avatar into ImageView if exists, else load the default drawable.
    void loadByUserId(Context context, String userId, ImageView view);

    // Load avatar into ImageView by local file path.
    void loadByFilePath(Context context, String filePath, ImageView view);

    // Return a {@link Bitmap} instance according to the specific image url and size.
    Bitmap getBitmap(Context context, String imageUrl, int ImageSize) throws ExecutionException, InterruptedException;

    // Return the url of avatar.
    String getUrl(String userId);
}
```

## 三、具体实现

定义抽象接口后，下一步是选择具体的图片加载框架。常用的图片加载框架有`Picasso`和`Glide`。由于头像已经抽象为接口，具体实现对用户是无感的。

如果将来决定迁移到其他图片加载框架，只需要在`UserAvatarLoader`层修改，所有调用点不需要任何改动。若更换实现时要修改抽象接口，想必在接口设计就存在设计遗漏甚至设计错误。因此，前期接口设计决定了后期实现难度和稳定性。

用`Glide`提供三级缓存功能，再配合`OkHttp`网络框架，能节省大量开发时间。

```java
public class UserAvatarLoader implements IUserAvatarLoader {

    private static final String API = "https://api.phantomvk.com";

    @Override
    public void loadByUserId(Context context, String userId, ImageView view) {
        Glide.with(context)
                .load(getUrl(userId))
                .apply(ImageOptions.userAvatarOptions)
                .into(view);
    }

    @Override
    public void loadByFilePath(Context context, String filePath, ImageView view) {
        Glide.with(context)
                .load(filePath)
                .apply(ImageOptions.userAvatarOptions)
                .into(view);
    }

    @Override
    public Bitmap getBitmap(Context context, String imageUrl, int imageSize)
            throws ExecutionException, InterruptedException {
            
        return Glide.with(context)
                .asBitmap()
                .load(imageUrl)
                .submit(imageSize, imageSize)
                .get();
    }

    @Override
    public String getUrl(String userId) {
        return API + "/api/v1/rcs/profile/" + userId + "/avatar";
    }
}
```

上面还用到了`Glide`的`RequestOption`，所以把所有`RequestOption`整理到一个类中。

其中

1. `R.drawable.def_avatar`是默认头像资源;
2. `android.R.color.darker_gray`是消息图片加载过程的背景色.

```java
public final class ImageOptions {

    public static RequestOptions userAvatarOptions = new RequestOptions()
            .placeholder(R.drawable.def_avatar)
            .error(R.drawable.def_avatar)
            .autoClone();

    public static RequestOptions chatMessageOptions = new RequestOptions()
            .placeholder(android.R.color.darker_gray)
            .centerCrop()
            .autoClone();
}
```

## 四、外观

实现已经完成，还差一步调用点的整合。如果不通过外观来提供调用点，使用时还是得各自构建对象。这样不仅消耗堆内存空间增加GC的压力，后期具体实现类还得逐个更换。所以还要增加一层，对调用屏蔽构建方式：

```java
public class ImageLoaders {

    private static IUserAvatarLoader userAvatarLoader = new UserAvatarLoader();
    
    private static IChatMessageLoader chatMessageLoader = new ChatMessageLoader();

    public static IUserAvatarLoader userAvatarLoader() {
        return userAvatarLoader;
    }

    public static IChatMessageLoader chatMessageLoader() {
        return chatMessageLoader;
    }
}
```

考虑到应用首次启动就要调用头像加载逻辑，主线程完成静态变量初始化且不考虑懒加载。

## 五、用法

用userId加载头像：

```java
ImageLoaders.userAvatarLoader().loadByUserId(context, userID, avatarView);
```

由文件加载头像：

```java
ImageLoaders.userAvatarLoader().loadByFilePath(context, file.getAbsolutePath(), view)
```

## 六、 参考链接

* [外观模式 - 维基百科](https://zh.wikipedia.org/wiki/%E5%A4%96%E8%A7%80%E6%A8%A1%E5%BC%8F)
* [The Repository Pattern - Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff649690(v=pandp.10))