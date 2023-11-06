---
layout:     post
title:      "Android源码系列(27) -- BitmapFactory"
date:       2020-07-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

### 图片体积

不同 __Bitmap.Config__ 对应像素不同的内存占用大小，__ARGB_8888__ 的像素还原度最高，__RGB_565__ 在不需要透明通道时能兼顾显示质量和内存占用。

| Bitmap.Config |     特性     |    bit     | byte(8bit=1byte) |
| :-----------: | :----------: | :--------: | :--------------: |
|    ALPHA_8    | 8位透明位图  |     8      |        1         |
|    RGB_565    | 32位RGB位图  |  5+6+6=16  |        2         |
|   ARGB_4444   | 16位ARGB位图 | 4+4+4+4=16 |        2         |
|   ARGB_8888   | 32位ARGB位图 | 8+8+8+8=32 |        4         |

除了像素体积，图片大小还和图片分辨率有关，而分辨率又和具体的数据途径有关：

- 从网络下载：网络提供图片原始大小；
- 从 __Asset__ 获取：原资源原始大小；
- 从 __xhdpi__ 获取：若能从屏幕对应缩放资源获取，为获取图片大小，否则根据比例缩放加载；
- 指定额外缩放参数而影响分辨率；

因此，图片实际内存体积由像素体积和分辨率共同作用。



### 源码解析

源码版本 **Android 9.0**

#### decodeResourceStream

本地通过 __BitmapFactory__ 解析本地资源时，主要调用方法 __decodeResourceStream__


```java
// Decode a new Bitmap from an InputStream. This InputStream was obtained from
// resources, which we pass to be able to scale the bitmap accordingly.
// @throws IllegalArgumentException if {@link BitmapFactory.Options#inPreferredConfig}
//         is {@link android.graphics.Bitmap.Config#HARDWARE}
//         and {@link BitmapFactory.Options#inMutable} is set, if the specified color space
//         is not {@link ColorSpace.Model#RGB RGB}, or if the specified color space's transfer
//         function is not an {@link ColorSpace.Rgb.TransferParameters ICC parametric curve}
@Nullable
public static Bitmap decodeResourceStream(@Nullable Resources res, @Nullable TypedValue value,
        @Nullable InputStream is, @Nullable Rect pad, @Nullable Options opts) {
    validate(opts);
    // 没有传入options就创建默认options
    if (opts == null) {
        opts = new Options();
    }

    if (opts.inDensity == 0 && value != null) {
        // value.density是资源所在文件夹对应的dpi
        final int density = value.density;
        if (density == TypedValue.DENSITY_DEFAULT) {     // DENSITY_DEFAULT = 0;
            // 默认drawable文件夹对应的密度160
            // DENSITY_MEDIUM = DENSITY_MEDIUM
            opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
        } else if (density != TypedValue.DENSITY_NONE) { // DENSITY_NONE= 0xffff
            opts.inDensity = density;
        }
    }

    if (opts.inTargetDensity == 0 && res != null) {
        // 如果inTargetDensity没有指定，默认为当前系统屏幕像素密度
        opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
    }

    return decodeStream(is, pad, opts);
}
```



#### decodeStream

继续调用方法 __Bitmap decodeStream(InputStream, Rect, Options);__

```java
// Decode an input stream into a bitmap. If the input stream is null, or
// cannot be used to decode a bitmap, the function returns null.
// The stream's position will be where ever it was after the encoded data
// was read.
//
// @param is The input stream that holds the raw data to be decoded into a
//           bitmap.
// @param outPadding If not null, return the padding rect for the bitmap if
//                   it exists, otherwise set padding to [-1,-1,-1,-1]. If
//                   no bitmap is returned (null) then padding is
//                   unchanged.
// @param opts null-ok; Options that control downsampling and whether the
//             image should be completely decoded, or just is size returned.
// @return The decoded bitmap, or null if the image data could not be
//         decoded, or, if opts is non-null, if opts requested only the
//         size be returned (in opts.outWidth and opts.outHeight)
// @throws IllegalArgumentException if {@link BitmapFactory.Options#inPreferredConfig}
//         is {@link android.graphics.Bitmap.Config#HARDWARE}
//         and {@link BitmapFactory.Options#inMutable} is set, if the specified color space
//         is not {@link ColorSpace.Model#RGB RGB}, or if the specified color space's transfer
//         function is not an {@link ColorSpace.Rgb.TransferParameters ICC parametric curve}
//
// <p class="note">Prior to {@link android.os.Build.VERSION_CODES#KITKAT},
// if {@link InputStream#markSupported is.markSupported()} returns true,
// <code>is.mark(1024)</code> would be called. As of
// {@link android.os.Build.VERSION_CODES#KITKAT}, this is no longer the case.</p>
@Nullable
public static Bitmap decodeStream(@Nullable InputStream is, @Nullable Rect outPadding,
        @Nullable Options opts) {
    // we don't throw in this case, thus allowing the caller to only check
    // the cache, and not force the image to be decoded.
    if (is == null) {
        return null;
    }
    validate(opts);

    Bitmap bm = null;

    Trace.traceBegin(Trace.TRACE_TAG_GRAPHICS, "decodeBitmap");
    try {
        if (is instanceof AssetManager.AssetInputStream) {
            // Asset资源有自己的native方法
            final long asset = ((AssetManager.AssetInputStream) is).getNativeAsset();
            bm = nativeDecodeAsset(asset, outPadding, opts, Options.nativeInBitmap(opts),
                Options.nativeColorSpace(opts));
        } else {
            // 关注此方法
            bm = decodeStreamInternal(is, outPadding, opts);
        }

        if (bm == null && opts != null && opts.inBitmap != null) {
            throw new IllegalArgumentException("Problem decoding into existing bitmap");
        }

        setDensityFromOptions(bm, opts);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_GRAPHICS);
    }

    return bm;
}
```

```java
/**
 * Set the newly decoded bitmap's density based on the Options.
 */
private static void setDensityFromOptions(Bitmap outputBitmap, Options opts) {
    if (outputBitmap == null || opts == null) return;

    final int density = opts.inDensity;
    if (density != 0) {
        outputBitmap.setDensity(density);
        final int targetDensity = opts.inTargetDensity;
        if (targetDensity == 0 || density == targetDensity || density == opts.inScreenDensity) {
            return;
        }

        byte[] np = outputBitmap.getNinePatchChunk();
        final boolean isNinePatch = np != null && NinePatch.isNinePatchChunk(np);
        if (opts.inScaled || isNinePatch) {
            outputBitmap.setDensity(targetDensity);
        }
    } else if (opts.inBitmap != null) {
        // bitmap was reused, ensure density is reset
        outputBitmap.setDensity(Bitmap.getDefaultDensity());
    }
}
```



调用方法 __decodeStreamInternal(InputStream, Rect, Options)__

```java
// Private helper function for decoding an InputStream natively. Buffers the input enough to
// do a rewind as needed, and supplies temporary storage if necessary. is MUST NOT be null.
private static Bitmap decodeStreamInternal(@NonNull InputStream is,
        @Nullable Rect outPadding, @Nullable Options opts) {
    byte [] tempStorage = null;
    if (opts != null) tempStorage = opts.inTempStorage;
    if (tempStorage == null) tempStorage = new byte[DECODE_BUFFER_SIZE];
    // 从这里开始进入Native方法解析Bitmap
    return nativeDecodeStream(is, tempStorage, outPadding, opts,
            Options.nativeInBitmap(opts),
            Options.nativeColorSpace(opts));
}
```

__BitmapFactory__ 在Java层的声明：

```java
@UnsupportedAppUsage
private static native Bitmap nativeDecodeStream(InputStream is, byte[] storage,
        Rect padding, Options opts, long inBitmapHandle, long colorSpaceHandle);
```

__Java Native__ 方法 __nativeDecodeStream__ 注册在 [BitmapFactory.cpp(Android9.0)](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#611) 的611行。

从源码可知 __BitmapFactory__ 支持多种解码方式：__FileDescriptor__、__InputStream__、__Asset__、__ByteArray__。

```cpp
static const JNINativeMethod gMethods[] = {
    {   "nativeDecodeStream",
        "(Ljava/io/InputStream;[BLandroid/graphics/Rect;Landroid/graphics/BitmapFactory$Options;)Landroid/graphics/Bitmap;",
        (void*)nativeDecodeStream
    },

    {   "nativeDecodeFileDescriptor",
        "(Ljava/io/FileDescriptor;Landroid/graphics/Rect;Landroid/graphics/BitmapFactory$Options;)Landroid/graphics/Bitmap;",
        (void*)nativeDecodeFileDescriptor
    },

    {   "nativeDecodeAsset",
        "(JLandroid/graphics/Rect;Landroid/graphics/BitmapFactory$Options;)Landroid/graphics/Bitmap;",
        (void*)nativeDecodeAsset
    },

    {   "nativeDecodeByteArray",
        "([BIILandroid/graphics/BitmapFactory$Options;)Landroid/graphics/Bitmap;",
        (void*)nativeDecodeByteArray
    },

    {   "nativeIsSeekable",
        "(Ljava/io/FileDescriptor;)Z",
        (void*)nativeIsSeekable
    },
};
```

这里看 __InputStream__ 方式，对应C++的函数签名 [__(void*)nativeDecodeStream__](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#517)：

```cpp
static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options) {

    // 声明Bitmap对象
    jobject bitmap = NULL;
    // 创建JavaInputStream
    std::unique_ptr<SkStream> stream(CreateJavaInputStreamAdaptor(env, is, storage));
    
    // 可以获取流
    if (stream.get()) {
        // 封装InputStream为BufferedStream
        std::unique_ptr<SkStreamRewindable> bufferedStream(
                SkFrontBufferedStream::Make(std::move(stream), SkCodec::MinBufferedBytesNeeded()));
        SkASSERT(bufferedStream.get() != NULL);
        // 用BufferedStream和参数解码图片
        bitmap = doDecode(env, std::move(bufferedStream), padding, options);
    }
    // 返回图片，解码失败返回NULL
    return bitmap;
}
```

继续调用 __[doDecode(env, std::move(bufferedStream), padding, options)]((http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#181))__ ：

```cpp
// env: jni指针
// stream: 流对象BufferedStream
// padding: 边距对象
// options: 图片参数
static jobject doDecode(JNIEnv* env, std::unique_ptr<SkStreamRewindable> stream,
                        jobject padding, jobject options) {
    // 默认缩放值
    int sampleSize = 1;
    bool onlyDecodeSize = false; // 是否只获取图片尺寸不读取图像
    SkColorType prefColorType = kN32_SkColorType;
    bool isHardware = false;
    bool isMutable = false;  // bitmap是否可变(共享)
    float scale = 1.0f;
    bool requireUnpremultiplied = false;
    // Java的Bitmap对象
    jobject javaBitmap = NULL;
    sk_sp<SkColorSpace> prefColorSpace = nullptr;

    // 从Java的options对象获取参数，赋值给C++的变量
    if (options != NULL) {
        // 从Java获取sampleSize值到C++，值为2的幂，如：1、2、4....
        sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
        // 校正非法的sampleSize值，必须为正数
        if (sampleSize <= 0) {
            sampleSize = 1; // 1就是原始尺寸
        }

        // 从Java获取justBounds值到C++
        if (env->GetBooleanField(options, gOptions_justBoundsFieldID)) {
            onlyDecodeSize = true;
        }

        // 给options设置默认的outWidth
        env->SetIntField(options, gOptions_widthFieldID, -1);
        // 给options设置默认的outHeight
        env->SetIntField(options, gOptions_heightFieldID, -1);
        env->SetObjectField(options, gOptions_mimeFieldID, 0);
        env->SetObjectField(options, gOptions_outConfigFieldID, 0);
        env->SetObjectField(options, gOptions_outColorSpaceFieldID, 0);

        // 解析inPreferredConfi为jconfig
        jobject jconfig = env->GetObjectField(options, gOptions_configFieldID);
        // 对应颜色类型：ALPHA_8、RGB_565、ARGB_4444、ARGB_8888
        prefColorType = GraphicsJNI::getNativeBitmapColorType(env, jconfig);

        // 解析颜色空间
        jobject jcolorSpace = env->GetObjectField(options, gOptions_colorSpaceFieldID);
        prefColorSpace = GraphicsJNI::getNativeColorSpace(env, jcolorSpace);

        // 硬件解码
        isHardware = GraphicsJNI::isHardwareConfig(env, jconfig);

        // 图像是否可变
        isMutable = env->GetBooleanField(options, gOptions_mutableFieldID);
        requireUnpremultiplied = !env->GetBooleanField(options, gOptions_premultipliedFieldID);
        javaBitmap = env->GetObjectField(options, gOptions_bitmapFieldID);

        // 获取缩放值
        if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
            // 像素密度
            const int density = env->GetIntField(options, gOptions_densityFieldID);
            // 目标像素密度
            const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
            // 屏幕像素密度
            const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
            if (density != 0 && targetDensity != 0 && density != screenDensity) {
                // 计算缩放比例
                scale = (float) targetDensity / density;
            }
        }
    }

    if (isMutable && isHardware) {
        doThrowIAE(env, "Bitmaps with Config.HARWARE are always immutable");
        return nullObjectReturn("Cannot create mutable hardware bitmap");
    }

    // 创建解码器
    NinePatchPeeker peeker;
    std::unique_ptr<SkAndroidCodec> codec;
    {
        SkCodec::Result result;
        std::unique_ptr<SkCodec> c = SkCodec::MakeFromStream(std::move(stream), &result,
                                                             &peeker);
        if (!c) {
            SkString msg;
            msg.printf("Failed to create image decoder with message '%s'",
                       SkCodec::ResultToString(result));
            return nullObjectReturn(msg.c_str());
        }

        codec = SkAndroidCodec::MakeFromCodec(std::move(c));
        if (!codec) {
            return nullObjectReturn("SkAndroidCodec::MakeFromCodec returned null");
        }
    }

    // Do not allow ninepatch decodes to 565.  In the past, decodes to 565
    // would dither, and we do not want to pre-dither ninepatches, since we
    // know that they will be stretched.  We no longer dither 565 decodes,
    // but we continue to prevent ninepatches from decoding to 565, in order
    // to maintain the old behavior.
    // 若ninepatches图的颜色类型被设置为RGB_565，则更正为ARGB_8888
    if (peeker.mPatch && kRGB_565_SkColorType == prefColorType) {
        prefColorType = kN32_SkColorType; // kN32_SkColorType is ARGB_8888
    }

    // 计算指定sampleSize对应dimen值
    SkISize size = codec->getSampledDimensions(sampleSize);

    // 结果原始图片的宽高值
    int scaledWidth = size.width();
    int scaledHeight = size.height();
    bool willScale = false;

    // Apply a fine scaling step if necessary.
    if (needsFineScale(codec->getInfo().dimensions(), size, sampleSize)) {
        willScale = true;
        scaledWidth = codec->getInfo().width() / sampleSize;
        scaledHeight = codec->getInfo().height() / sampleSize;
    }

    // 设置颜色：ALPHA_8、RGB_565、ARGB_4444、ARGB_8888
    SkColorType decodeColorType = codec->computeOutputColorType(prefColorType);
    sk_sp<SkColorSpace> decodeColorSpace = codec->computeOutputColorSpace(
            decodeColorType, prefColorSpace);

    // 设置options，如果只想要图片尺寸大小，这个代码块里会完成返回
    if (options != NULL) {
        // 解析mimeType
        jstring mimeType = encodedFormatToString(
                env, (SkEncodedImageFormat)codec->getEncodedFormat());
        if (env->ExceptionCheck()) {
            return nullObjectReturn("OOM in encodedFormatToString()");
        }
        // 用options保存图片outWidth
        env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
        // 用options保存图片的outHeight
        env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
        // 用options保存图片的mimeType
        env->SetObjectField(options, gOptions_mimeFieldID, mimeType);

        jint configID = GraphicsJNI::colorTypeToLegacyBitmapConfig(decodeColorType);
        if (isHardware) {
            configID = GraphicsJNI::kHardware_LegacyBitmapConfig;
        }
        jobject config = env->CallStaticObjectMethod(gBitmapConfig_class,
                gBitmapConfig_nativeToConfigMethodID, configID);
        env->SetObjectField(options, gOptions_outConfigFieldID, config);

        env->SetObjectField(options, gOptions_outColorSpaceFieldID,
                GraphicsJNI::getColorSpace(env, decodeColorSpace, decodeColorType));

        if (onlyDecodeSize) {
            // 因为只解析图片尺寸，所以Bitmap返回nullptr
            return nullptr;
        }
    }

    // Scale is necessary due to density differences.
    if (scale != 1.0f) {
        willScale = true;
        // 用scale计算尺寸
        scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);
        scaledHeight = static_cast<int>(scaledHeight * scale + 0.5f);
    }

    android::Bitmap* reuseBitmap = nullptr;
    unsigned int existingBufferSize = 0;
    if (javaBitmap != NULL) {
        reuseBitmap = &bitmap::toBitmap(env, javaBitmap);
        // 由于可重用的Bitmap被设置为不可变，所以重置该引用
        if (reuseBitmap->isImmutable()) {
            ALOGW("Unable to reuse an immutable bitmap as an image decoder target.");
            javaBitmap = NULL;
            reuseBitmap = nullptr;
        } else {
            existingBufferSize = bitmap::getBitmapAllocationByteCount(env, javaBitmap);
        }
    }

    HeapAllocator defaultAllocator;
    RecyclingPixelAllocator recyclingAllocator(reuseBitmap, existingBufferSize);
    ScaleCheckingAllocator scaleCheckingAllocator(scale, existingBufferSize);
    SkBitmap::HeapAllocator heapAllocator;
    SkBitmap::Allocator* decodeAllocator;
    if (javaBitmap != nullptr && willScale) {
        // This will allocate pixels using a HeapAllocator, since there will be an extra
        // scaling step that copies these pixels into Java memory.  This allocator
        // also checks that the recycled javaBitmap is large enough.
        decodeAllocator = &scaleCheckingAllocator;
    } else if (javaBitmap != nullptr) {
        decodeAllocator = &recyclingAllocator;
    } else if (willScale || isHardware) {
        // This will allocate pixels using a HeapAllocator,
        // for scale case: there will be an extra scaling step.
        // for hardware case: there will be extra swizzling & upload to gralloc step.
        decodeAllocator = &heapAllocator;
    } else {
        decodeAllocator = &defaultAllocator;
    }

    SkAlphaType alphaType = codec->computeOutputAlphaType(requireUnpremultiplied);

    const SkImageInfo decodeInfo = SkImageInfo::Make(size.width(), size.height(),
            decodeColorType, alphaType, decodeColorSpace);

    // For wide gamut images, we will leave the color space on the SkBitmap.  Otherwise,
    // use the default.
    SkImageInfo bitmapInfo = decodeInfo;
    if (decodeInfo.colorSpace() && decodeInfo.colorSpace()->isSRGB()) {
        bitmapInfo = bitmapInfo.makeColorSpace(GraphicsJNI::colorSpaceForType(decodeColorType));
    }

    if (decodeColorType == kGray_8_SkColorType) {
        // The legacy implementation of BitmapFactory used kAlpha8 for
        // grayscale images (before kGray8 existed).  While the codec
        // recognizes kGray8, we need to decode into a kAlpha8 bitmap
        // in order to avoid a behavior change.
        bitmapInfo =
                bitmapInfo.makeColorType(kAlpha_8_SkColorType).makeAlphaType(kPremul_SkAlphaType);
    }
    SkBitmap decodingBitmap;
    if (!decodingBitmap.setInfo(bitmapInfo) ||
            !decodingBitmap.tryAllocPixels(decodeAllocator)) {
        // SkAndroidCodec should recommend a valid SkImageInfo, so setInfo()
        // should only only fail if the calculated value for rowBytes is too
        // large.
        // tryAllocPixels() can fail due to OOM on the Java heap, OOM on the
        // native heap, or the recycled javaBitmap being too small to reuse.
        return nullptr;
    }

    // 使用SkAndroidCodec执行
    SkAndroidCodec::AndroidOptions codecOptions;
    codecOptions.fZeroInitialized = decodeAllocator == &defaultAllocator ?
            SkCodec::kYes_ZeroInitialized : SkCodec::kNo_ZeroInitialized;
    codecOptions.fSampleSize = sampleSize;
    SkCodec::Result result = codec->getAndroidPixels(decodeInfo, decodingBitmap.getPixels(),
            decodingBitmap.rowBytes(), &codecOptions);
    switch (result) {
        case SkCodec::kSuccess:
        case SkCodec::kIncompleteInput:
            break;
        default:
            return nullObjectReturn("codec->getAndroidPixels() failed.");
    }

    // This is weird so let me explain: we could use the scale parameter
    // directly, but for historical reasons this is how the corresponding
    // Dalvik code has always behaved. We simply recreate the behavior here.
    // The result is slightly different from simply using scale because of
    // the 0.5f rounding bias applied when computing the target image size
    const float scaleX = scaledWidth / float(decodingBitmap.width());
    const float scaleY = scaledHeight / float(decodingBitmap.height());

    jbyteArray ninePatchChunk = NULL;
    if (peeker.mPatch != NULL) {
        if (willScale) {
            peeker.scale(scaleX, scaleY, scaledWidth, scaledHeight);
        }

        size_t ninePatchArraySize = peeker.mPatch->serializedSize();
        ninePatchChunk = env->NewByteArray(ninePatchArraySize);
        if (ninePatchChunk == NULL) {
            return nullObjectReturn("ninePatchChunk == null");
        }

        jbyte* array = (jbyte*) env->GetPrimitiveArrayCritical(ninePatchChunk, NULL);
        if (array == NULL) {
            return nullObjectReturn("primitive array == null");
        }

        memcpy(array, peeker.mPatch, peeker.mPatchSize);
        env->ReleasePrimitiveArrayCritical(ninePatchChunk, array, 0);
    }

    jobject ninePatchInsets = NULL;
    if (peeker.mHasInsets) {
        ninePatchInsets = peeker.createNinePatchInsets(env, scale);
        if (ninePatchInsets == NULL) {
            return nullObjectReturn("nine patch insets == null");
        }
        if (javaBitmap != NULL) {
            env->SetObjectField(javaBitmap, gBitmap_ninePatchInsetsFieldID, ninePatchInsets);
        }
    }

    SkBitmap outputBitmap;
    if (willScale) {
        // Set the allocator for the outputBitmap.
        SkBitmap::Allocator* outputAllocator;
        if (javaBitmap != nullptr) {
            outputAllocator = &recyclingAllocator;
        } else {
            outputAllocator = &defaultAllocator;
        }

        SkColorType scaledColorType = decodingBitmap.colorType();
        // FIXME: If the alphaType is kUnpremul and the image has alpha, the
        // colors may not be correct, since Skia does not yet support drawing
        // to/from unpremultiplied bitmaps.
        outputBitmap.setInfo(
                bitmapInfo.makeWH(scaledWidth, scaledHeight).makeColorType(scaledColorType));
        if (!outputBitmap.tryAllocPixels(outputAllocator)) {
            // This should only fail on OOM.  The recyclingAllocator should have
            // enough memory since we check this before decoding using the
            // scaleCheckingAllocator.
            return nullObjectReturn("allocation failed for scaled bitmap");
        }

        SkPaint paint;
        // kSrc_Mode instructs us to overwrite the uninitialized pixels in
        // outputBitmap.  Otherwise we would blend by default, which is not
        // what we want.
        paint.setBlendMode(SkBlendMode::kSrc);
        paint.setFilterQuality(kLow_SkFilterQuality); // bilinear filtering

        SkCanvas canvas(outputBitmap, SkCanvas::ColorBehavior::kLegacy);
        canvas.scale(scaleX, scaleY);
        canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);
    } else {
        outputBitmap.swap(decodingBitmap);
    }

    if (padding) {
        peeker.getPadding(env, padding);
    }

    // If we get here, the outputBitmap should have an installed pixelref.
    if (outputBitmap.pixelRef() == NULL) {
        return nullObjectReturn("Got null SkPixelRef");
    }

    if (!isMutable && javaBitmap == NULL) {
        // 承诺图像像素不会改变，有利于共享图像
        outputBitmap.setImmutable();
    }

    bool isPremultiplied = !requireUnpremultiplied;
    if (javaBitmap != nullptr) {
        bitmap::reinitBitmap(env, javaBitmap, outputBitmap.info(), isPremultiplied);
        outputBitmap.notifyPixelsChanged();
        // 把传入复用的inBitmap作为结果返回
        return javaBitmap;
    }

    int bitmapCreateFlags = 0x0;
    if (isMutable) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Mutable;
    if (isPremultiplied) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Premultiplied;

    // 硬件加载Bitmap
    if (isHardware) {
        sk_sp<Bitmap> hardwareBitmap = Bitmap::allocateHardwareBitmap(outputBitmap);
        if (!hardwareBitmap.get()) {
            return nullObjectReturn("Failed to allocate a hardware bitmap");
        }
        return bitmap::createBitmap(env, hardwareBitmap.release(), bitmapCreateFlags,
                ninePatchChunk, ninePatchInsets, -1);
    }

    // 创建java的bitmap
    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}
```

__bitmap::createBitmap(….)__ 在 [Bitmap Line199](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/Bitmap.cpp)

```cpp
jobject createBitmap(JNIEnv* env, Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {
    bool isMutable = bitmapCreateFlags & kBitmapCreateFlag_Mutable;
    bool isPremultiplied = bitmapCreateFlags & kBitmapCreateFlag_Premultiplied;
    // The caller needs to have already set the alpha type properly, so the
    // native SkBitmap stays in sync with the Java Bitmap.
    assert_premultiplied(bitmap->info(), isPremultiplied);
    BitmapWrapper* bitmapWrapper = new BitmapWrapper(bitmap);
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isMutable, isPremultiplied, ninePatchChunk, ninePatchInsets);

    if (env->ExceptionCheck() != 0) {
        ALOGE("*** Uncaught exception returned from Java call!\n");
        env->ExceptionDescribe();
    }
    // 返回Java可用Bitmap
    return obj;
}
```

__Java__ 与 __C++__ 层参数名对照表

```c++
int register_android_graphics_BitmapFactory(JNIEnv* env) {
    jclass options_class = FindClassOrDie(env, "android/graphics/BitmapFactory$Options");
    gOptions_bitmapFieldID = GetFieldIDOrDie(env, options_class, "inBitmap",
            "Landroid/graphics/Bitmap;");
    gOptions_justBoundsFieldID = GetFieldIDOrDie(env, options_class, "inJustDecodeBounds", "Z");
    gOptions_sampleSizeFieldID = GetFieldIDOrDie(env, options_class, "inSampleSize", "I");
    gOptions_configFieldID = GetFieldIDOrDie(env, options_class, "inPreferredConfig",
            "Landroid/graphics/Bitmap$Config;");
    gOptions_colorSpaceFieldID = GetFieldIDOrDie(env, options_class, "inPreferredColorSpace",
            "Landroid/graphics/ColorSpace;");
    gOptions_premultipliedFieldID = GetFieldIDOrDie(env, options_class, "inPremultiplied", "Z");
    gOptions_mutableFieldID = GetFieldIDOrDie(env, options_class, "inMutable", "Z");
    gOptions_ditherFieldID = GetFieldIDOrDie(env, options_class, "inDither", "Z");
    gOptions_preferQualityOverSpeedFieldID = GetFieldIDOrDie(env, options_class,
            "inPreferQualityOverSpeed", "Z");
    gOptions_scaledFieldID = GetFieldIDOrDie(env, options_class, "inScaled", "Z");
    gOptions_densityFieldID = GetFieldIDOrDie(env, options_class, "inDensity", "I");
    gOptions_screenDensityFieldID = GetFieldIDOrDie(env, options_class, "inScreenDensity", "I");
    gOptions_targetDensityFieldID = GetFieldIDOrDie(env, options_class, "inTargetDensity", "I");
    gOptions_widthFieldID = GetFieldIDOrDie(env, options_class, "outWidth", "I");
    gOptions_heightFieldID = GetFieldIDOrDie(env, options_class, "outHeight", "I");
    gOptions_mimeFieldID = GetFieldIDOrDie(env, options_class, "outMimeType", "Ljava/lang/String;");
    gOptions_outConfigFieldID = GetFieldIDOrDie(env, options_class, "outConfig",
             "Landroid/graphics/Bitmap$Config;");
    gOptions_outColorSpaceFieldID = GetFieldIDOrDie(env, options_class, "outColorSpace",
             "Landroid/graphics/ColorSpace;");
    gOptions_mCancelID = GetFieldIDOrDie(env, options_class, "mCancel", "Z");

    jclass bitmap_class = FindClassOrDie(env, "android/graphics/Bitmap");
    gBitmap_ninePatchInsetsFieldID = GetFieldIDOrDie(env, bitmap_class, "mNinePatchInsets",
            "Landroid/graphics/NinePatch$InsetStruct;");

    gBitmapConfig_class = MakeGlobalRefOrDie(env, FindClassOrDie(env,
            "android/graphics/Bitmap$Config"));
    gBitmapConfig_nativeToConfigMethodID = GetStaticMethodIDOrDie(env, gBitmapConfig_class,
            "nativeToConfig", "(I)Landroid/graphics/Bitmap$Config;");

    return android::RegisterMethodsOrDie(env, "android/graphics/BitmapFactory",
                                         gMethods, NELEM(gMethods));
}
```

### 总结

从 __hdpi__ 文件夹读取图片：

- 缩放比例：__scaledsize = inTargetDensity / inDensity__

- 图片大小：__imageSize = (width * scaledsize) * (height * scaledsize) * pixelSize__

### 参考链接

- [Android Bitmap 那些事](https://juejin.im/entry/57cd1c7cbf22ec006c2e2261)
- [高效加载Bitmap(一) --BitmapFactory加载bitmap](https://livesun.github.io/2017/10/20/Bitmap(一)/)