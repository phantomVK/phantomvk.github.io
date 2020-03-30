---
layout:     post
title:      "Android源码系列(27) -- BitmapFactory"
date:       2020-03-31
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

除了像素体积，图片大小还和图片分辨率有关，而分辨率又和具体的数据来源有关：

- 从网络下载：网络提供图片原始大小；
- 从 __raw、Asset__ 获取：原资源原始大小；
- 从 __xhdpi__ 等获取：若能从屏幕对应缩放资源获取，则为获取图片大小。否则按比例缩放加载；
- 指定额外缩放参数而影响分辨率；

因此，图片实际内存体积由像素体积和分辨率共同作用。

### 源码解析

本地通过 __BitmapFactory__ 解析本地时，主要调用方法 __decodeResourceStream__


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
    if (opts == null) {
        opts = new Options();
    }

    if (opts.inDensity == 0 && value != null) {
        final int density = value.density;
        if (density == TypedValue.DENSITY_DEFAULT) {
            opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
        } else if (density != TypedValue.DENSITY_NONE) {
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
        // 一般不是Asset资源
        if (is instanceof AssetManager.AssetInputStream) {
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

Java Native方法 __nativeDecodeStream__ 注册在 [BitmapFactory.cpp(Android9.0)](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#611) 的611行：

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

对应cpp的函数签名为 [__(void*)nativeDecodeStream__](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#517)：

```cpp
static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options) {

    jobject bitmap = NULL;
    std::unique_ptr<SkStream> stream(CreateJavaInputStreamAdaptor(env, is, storage));

    if (stream.get()) {
        std::unique_ptr<SkStreamRewindable> bufferedStream(
                SkFrontBufferedStream::Make(std::move(stream), SkCodec::MinBufferedBytesNeeded()));
        SkASSERT(bufferedStream.get() != NULL);
        bitmap = doDecode(env, std::move(bufferedStream), padding, options);
    }
    return bitmap;
}
```

上面调用的 __[doDecode(env, std::move(bufferedStream), padding, options)]((http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#181))__ ：

```cpp
static jobject doDecode(JNIEnv* env, std::unique_ptr<SkStreamRewindable> stream,
                        jobject padding, jobject options) {
    // Set default values for the options parameters.
    int sampleSize = 1;
    bool onlyDecodeSize = false;
    SkColorType prefColorType = kN32_SkColorType;
    bool isHardware = false;
    bool isMutable = false;
    float scale = 1.0f;
    bool requireUnpremultiplied = false;
    jobject javaBitmap = NULL;
    sk_sp<SkColorSpace> prefColorSpace = nullptr;

    // Update with options supplied by the client.
    if (options != NULL) {
        sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
        // Correct a non-positive sampleSize.  sampleSize defaults to zero within the
        // options object, which is strange.
        if (sampleSize <= 0) {
            sampleSize = 1;
        }

        if (env->GetBooleanField(options, gOptions_justBoundsFieldID)) {
            onlyDecodeSize = true;
        }

        // initialize these, in case we fail later on
        env->SetIntField(options, gOptions_widthFieldID, -1);
        env->SetIntField(options, gOptions_heightFieldID, -1);
        env->SetObjectField(options, gOptions_mimeFieldID, 0);
        env->SetObjectField(options, gOptions_outConfigFieldID, 0);
        env->SetObjectField(options, gOptions_outColorSpaceFieldID, 0);

        jobject jconfig = env->GetObjectField(options, gOptions_configFieldID);
        prefColorType = GraphicsJNI::getNativeBitmapColorType(env, jconfig);
        jobject jcolorSpace = env->GetObjectField(options, gOptions_colorSpaceFieldID);
        prefColorSpace = GraphicsJNI::getNativeColorSpace(env, jcolorSpace);
        isHardware = GraphicsJNI::isHardwareConfig(env, jconfig);
        isMutable = env->GetBooleanField(options, gOptions_mutableFieldID);
        requireUnpremultiplied = !env->GetBooleanField(options, gOptions_premultipliedFieldID);
        javaBitmap = env->GetObjectField(options, gOptions_bitmapFieldID);

        if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
            const int density = env->GetIntField(options, gOptions_densityFieldID);
            const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
            const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
            if (density != 0 && targetDensity != 0 && density != screenDensity) {
                scale = (float) targetDensity / density;
            }
        }
    }

    if (isMutable && isHardware) {
        doThrowIAE(env, "Bitmaps with Config.HARWARE are always immutable");
        return nullObjectReturn("Cannot create mutable hardware bitmap");
    }

    // Create the codec.
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
    if (peeker.mPatch && kRGB_565_SkColorType == prefColorType) {
        prefColorType = kN32_SkColorType;
    }

    // Determine the output size.
    SkISize size = codec->getSampledDimensions(sampleSize);

    int scaledWidth = size.width();
    int scaledHeight = size.height();
    bool willScale = false;

    // Apply a fine scaling step if necessary.
    if (needsFineScale(codec->getInfo().dimensions(), size, sampleSize)) {
        willScale = true;
        scaledWidth = codec->getInfo().width() / sampleSize;
        scaledHeight = codec->getInfo().height() / sampleSize;
    }

    // Set the decode colorType
    SkColorType decodeColorType = codec->computeOutputColorType(prefColorType);
    sk_sp<SkColorSpace> decodeColorSpace = codec->computeOutputColorSpace(
            decodeColorType, prefColorSpace);

    // Set the options and return if the client only wants the size.
    if (options != NULL) {
        jstring mimeType = encodedFormatToString(
                env, (SkEncodedImageFormat)codec->getEncodedFormat());
        if (env->ExceptionCheck()) {
            return nullObjectReturn("OOM in encodedFormatToString()");
        }
        env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
        env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
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
            return nullptr;
        }
    }

    // Scale is necessary due to density differences.
    if (scale != 1.0f) {
        willScale = true;
        scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);
        scaledHeight = static_cast<int>(scaledHeight * scale + 0.5f);
    }

    android::Bitmap* reuseBitmap = nullptr;
    unsigned int existingBufferSize = 0;
    if (javaBitmap != NULL) {
        reuseBitmap = &bitmap::toBitmap(env, javaBitmap);
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

    // Use SkAndroidCodec to perform the decode.
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
        // promise we will never change our pixels (great for sharing and pictures)
        outputBitmap.setImmutable();
    }

    bool isPremultiplied = !requireUnpremultiplied;
    if (javaBitmap != nullptr) {
        bitmap::reinitBitmap(env, javaBitmap, outputBitmap.info(), isPremultiplied);
        outputBitmap.notifyPixelsChanged();
        // If a java bitmap was passed in for reuse, pass it back
        return javaBitmap;
    }

    int bitmapCreateFlags = 0x0;
    if (isMutable) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Mutable;
    if (isPremultiplied) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Premultiplied;

    if (isHardware) {
        sk_sp<Bitmap> hardwareBitmap = Bitmap::allocateHardwareBitmap(outputBitmap);
        if (!hardwareBitmap.get()) {
            return nullObjectReturn("Failed to allocate a hardware bitmap");
        }
        return bitmap::createBitmap(env, hardwareBitmap.release(), bitmapCreateFlags,
                ninePatchChunk, ninePatchInsets, -1);
    }

    // now create the java bitmap
    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}
```

### 总结

缩放比例：__scaledsize = nTargetDensity / inDensity__

图片大小：__imageSize = (width * scaledsize) * (height * scaledsize) * pixelSize__