---
layout:     post
title:      "Glide -- ResourceTranscoder"
date:       2019-02-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Glide
---

源码版本 __Glide 4.8.0__

## ResourceTranscoder

__ResourceTranscoder__ 是 __Glide__ 内类型转换器的抽象接口，表示资源从一种类型转换为另一种类型。泛型 __Z__ 表示转换前资源的原始类型，泛型 __R__ 表示资源转换后的目标类型。

```java
public interface ResourceTranscoder<Z, R> {

  // 子类实现此方法，根据既定规则和传入参数返回转换结果
  @Nullable
  Resource<R> transcode(@NonNull Resource<Z> toTranscode, @NonNull Options options);
}
```

__ResourceTranscoder__ 的实现子类一种有5个：

![ResourceTranscoder](/img/Glide/ResourceTranscoder.png)

## BitmapBytesTranscoder

转换 __Bitmap__ 资源为字节数组

```java
public class BitmapBytesTranscoder implements ResourceTranscoder<Bitmap, byte[]> {
  private final Bitmap.CompressFormat compressFormat;
  private final int quality;

  public BitmapBytesTranscoder() {
    // 默认转换为JPEG，压缩质量100，该质量相当于最轻量的压缩，而非不压缩
    this(Bitmap.CompressFormat.JPEG, 100);
  }

  @SuppressWarnings("WeakerAccess")
  public BitmapBytesTranscoder(@NonNull Bitmap.CompressFormat compressFormat, int quality) {
    this.compressFormat = compressFormat;
    this.quality = quality;
  }

  @Nullable
  @Override
  public Resource<byte[]> transcode(@NonNull Resource<Bitmap> toTranscode,
      @NonNull Options options) {
    // 创建输出流
    ByteArrayOutputStream os = new ByteArrayOutputStream();
    // android.graphics.Bitmap#compress(android.graphics.Bitmap.CompressFormat, int, java.io.OutputStream)
    toTranscode.get().compress(compressFormat, quality, os);
    toTranscode.recycle();
    return new BytesResource(os.toByteArray());
  }
}
```

## BitmapDrawableTranscoder

转换 __Bitmap__ 资源为 __BitmapDrawable__

```java
public class BitmapDrawableTranscoder implements ResourceTranscoder<Bitmap, BitmapDrawable> {
  private final Resources resources;

  @SuppressWarnings("unused")
  public BitmapDrawableTranscoder(@NonNull Context context) {
    this(context.getResources());
  }

  /**
   * @deprecated Use {@link #BitmapDrawableTranscoder(Resources)}, {@code bitmapPool} is unused.
   */
  @Deprecated
  public BitmapDrawableTranscoder(
      @NonNull Resources resources, @SuppressWarnings("unused") BitmapPool bitmapPool) {
    this(resources);
  }

  public BitmapDrawableTranscoder(@NonNull Resources resources) {
    this.resources = Preconditions.checkNotNull(resources);
  }

  @Nullable
  @Override
  public Resource<BitmapDrawable> transcode(@NonNull Resource<Bitmap> toTranscode,
      @NonNull Options options) {
    return LazyBitmapDrawableResource.obtain(resources, toTranscode);
  }
}
```

## DrawableBytesTranscoder

转换 __Drawable__ 资源为字节数组

```java
public final class DrawableBytesTranscoder implements ResourceTranscoder<Drawable, byte[]> {
  private final BitmapPool bitmapPool;
  private final ResourceTranscoder<Bitmap, byte[]> bitmapBytesTranscoder;
  private final ResourceTranscoder<GifDrawable, byte[]> gifDrawableBytesTranscoder;

  public DrawableBytesTranscoder(
      @NonNull BitmapPool bitmapPool,
      @NonNull ResourceTranscoder<Bitmap, byte[]> bitmapBytesTranscoder,
      @NonNull ResourceTranscoder<GifDrawable, byte[]> gifDrawableBytesTranscoder) {
    this.bitmapPool = bitmapPool;
    this.bitmapBytesTranscoder = bitmapBytesTranscoder;
    this.gifDrawableBytesTranscoder = gifDrawableBytesTranscoder;
  }

  @Nullable
  @Override
  public Resource<byte[]> transcode(@NonNull Resource<Drawable> toTranscode,
      @NonNull Options options) {
    Drawable drawable = toTranscode.get();
    if (drawable instanceof BitmapDrawable) {
      return bitmapBytesTranscoder.transcode(
          BitmapResource.obtain(((BitmapDrawable) drawable).getBitmap(), bitmapPool), options);
    } else if (drawable instanceof GifDrawable) {
      return gifDrawableBytesTranscoder.transcode(toGifDrawableResource(toTranscode), options);
    }
    return null;
  }

  @SuppressWarnings("unchecked")
  @NonNull
  private static Resource<GifDrawable> toGifDrawableResource(@NonNull Resource<Drawable> resource) {
    return (Resource<GifDrawable>) (Resource<?>) resource;
  }
}
```

## GifDrawableBytesTranscoder

把 __com.bumptech.glide.load.resource.gif.GifDrawable__ 资源转换为字节数组

```java
public class GifDrawableBytesTranscoder implements ResourceTranscoder<GifDrawable, byte[]> {
  @Nullable
  @Override
  public Resource<byte[]> transcode(@NonNull Resource<GifDrawable> toTranscode,
      @NonNull Options options) {
    GifDrawable gifData = toTranscode.get();
    ByteBuffer byteBuffer = gifData.getBuffer();
    return new BytesResource(ByteBufferUtil.toBytes(byteBuffer));
  }
}
```

## UnitTranscoder

__ResourceTranscoder__ 的实现最简单，输入和输出类型完全相同

```java
public class UnitTranscoder<Z> implements ResourceTranscoder<Z, Z> {
  // 本实现类的单例
  private static final UnitTranscoder<?> UNIT_TRANSCODER = new UnitTranscoder<>();

  @SuppressWarnings("unchecked")
  public static <Z> ResourceTranscoder<Z, Z> get() {
    return (ResourceTranscoder<Z, Z>) UNIT_TRANSCODER;
  }

  @Nullable
  @Override
  public Resource<Z> transcode(@NonNull Resource<Z> toTranscode, @NonNull Options options) {
    // 直接返回传入对象
    return toTranscode;
  }
}
```
