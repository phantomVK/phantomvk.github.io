---
layout:     post
title:      "Android源码系列(27) -- BitmapFactory"
date:       2020-01-31
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
610static const JNINativeMethod gMethods[] = {
611    {   "nativeDecodeStream",
612        "(Ljava/io/InputStream;[BLandroid/graphics/Rect;Landroid/graphics/BitmapFactory$Options;)Landroid/graphics/Bitmap;",
613        (void*)nativeDecodeStream
614    },
615    ....
635};
```

对应cpp的函数签名为 [__(void*)nativeDecodeStream__](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#517)：

```cpp
517static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
518        jobject padding, jobject options) {
519
520    jobject bitmap = NULL;
521    std::unique_ptr<SkStream> stream(CreateJavaInputStreamAdaptor(env, is, storage));
522
523    if (stream.get()) {
524        std::unique_ptr<SkStreamRewindable> bufferedStream(
525                SkFrontBufferedStream::Make(std::move(stream), SkCodec::MinBufferedBytesNeeded()));
526        SkASSERT(bufferedStream.get() != NULL);
527        bitmap = doDecode(env, std::move(bufferedStream), padding, options);
528    }
529    return bitmap;
530}
```

上面调用的 __[doDecode(env, std::move(bufferedStream), padding, options)]((http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp#181))__ ：

```cpp
181static jobject doDecode(JNIEnv* env, std::unique_ptr<SkStreamRewindable> stream,
182                        jobject padding, jobject options) {
183    // 给options对象的数据成员设置初始值
184    int sampleSize = 1;
185    bool onlyDecodeSize = false;
186    SkColorType prefColorType = kN32_SkColorType;
187    bool isHardware = false;
188    bool isMutable = false;
189    float scale = 1.0f;
190    bool requireUnpremultiplied = false;
191    jobject javaBitmap = NULL;
192    sk_sp<SkColorSpace> prefColorSpace = nullptr;
193
194    // Update with options supplied by the client.
195    if (options != NULL) {
196        sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
197        // Correct a non-positive sampleSize.  sampleSize defaults to zero within the
198        // options object, which is strange.
199        if (sampleSize <= 0) {
200            sampleSize = 1;
201        }
202
203        if (env->GetBooleanField(options, gOptions_justBoundsFieldID)) {
204            onlyDecodeSize = true;
205        }
206
207        // initialize these, in case we fail later on
208        env->SetIntField(options, gOptions_widthFieldID, -1);
209        env->SetIntField(options, gOptions_heightFieldID, -1);
210        env->SetObjectField(options, gOptions_mimeFieldID, 0);
211        env->SetObjectField(options, gOptions_outConfigFieldID, 0);
212        env->SetObjectField(options, gOptions_outColorSpaceFieldID, 0);
213
214        jobject jconfig = env->GetObjectField(options, gOptions_configFieldID);
215        prefColorType = GraphicsJNI::getNativeBitmapColorType(env, jconfig);
216        jobject jcolorSpace = env->GetObjectField(options, gOptions_colorSpaceFieldID);
217        prefColorSpace = GraphicsJNI::getNativeColorSpace(env, jcolorSpace);
218        isHardware = GraphicsJNI::isHardwareConfig(env, jconfig);
219        isMutable = env->GetBooleanField(options, gOptions_mutableFieldID);
220        requireUnpremultiplied = !env->GetBooleanField(options, gOptions_premultipliedFieldID);
221        javaBitmap = env->GetObjectField(options, gOptions_bitmapFieldID);
222
223        if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
224            const int density = env->GetIntField(options, gOptions_densityFieldID);
225            const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID); // 目标图片像素密度
226            const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID); // 获取屏幕像素密度
227            if (density != 0 && targetDensity != 0 && density != screenDensity) {
228                scale = (float) targetDensity / density; // 算出像素密度比例
229            }
230        }
231    }
232
233    if (isMutable && isHardware) {
234        doThrowIAE(env, "Bitmaps with Config.HARWARE are always immutable");
235        return nullObjectReturn("Cannot create mutable hardware bitmap");
236    }
237
238    // Create the codec.
239    NinePatchPeeker peeker;
240    std::unique_ptr<SkAndroidCodec> codec;
241    {
242        SkCodec::Result result;
243        std::unique_ptr<SkCodec> c = SkCodec::MakeFromStream(std::move(stream), &result,
244                                                             &peeker);
245        if (!c) {
246            SkString msg;
247            msg.printf("Failed to create image decoder with message '%s'",
248                       SkCodec::ResultToString(result));
249            return nullObjectReturn(msg.c_str());
250        }
251
252        codec = SkAndroidCodec::MakeFromCodec(std::move(c));
253        if (!codec) {
254            return nullObjectReturn("SkAndroidCodec::MakeFromCodec returned null");
255        }
256    }
257
258    // Do not allow ninepatch decodes to 565.  In the past, decodes to 565
259    // would dither, and we do not want to pre-dither ninepatches, since we
260    // know that they will be stretched.  We no longer dither 565 decodes,
261    // but we continue to prevent ninepatches from decoding to 565, in order
262    // to maintain the old behavior.
263    if (peeker.mPatch && kRGB_565_SkColorType == prefColorType) {
264        prefColorType = kN32_SkColorType;
265    }
266
267    // Determine the output size.
268    SkISize size = codec->getSampledDimensions(sampleSize);
269
270    int scaledWidth = size.width();
271    int scaledHeight = size.height();
272    bool willScale = false;
273
274    // Apply a fine scaling step if necessary.
275    if (needsFineScale(codec->getInfo().dimensions(), size, sampleSize)) {
276        willScale = true;
277        scaledWidth = codec->getInfo().width() / sampleSize;
278        scaledHeight = codec->getInfo().height() / sampleSize;
279    }
280
281    // Set the decode colorType
282    SkColorType decodeColorType = codec->computeOutputColorType(prefColorType);
283    sk_sp<SkColorSpace> decodeColorSpace = codec->computeOutputColorSpace(
284            decodeColorType, prefColorSpace);
285
286    // Set the options and return if the client only wants the size.
287    if (options != NULL) {
288        jstring mimeType = encodedFormatToString(
289                env, (SkEncodedImageFormat)codec->getEncodedFormat());
290        if (env->ExceptionCheck()) {
291            return nullObjectReturn("OOM in encodedFormatToString()");
292        }
293        env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
294        env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
295        env->SetObjectField(options, gOptions_mimeFieldID, mimeType);
296
297        jint configID = GraphicsJNI::colorTypeToLegacyBitmapConfig(decodeColorType);
298        if (isHardware) {
299            configID = GraphicsJNI::kHardware_LegacyBitmapConfig;
300        }
301        jobject config = env->CallStaticObjectMethod(gBitmapConfig_class,
302                gBitmapConfig_nativeToConfigMethodID, configID);
303        env->SetObjectField(options, gOptions_outConfigFieldID, config);
304
305        env->SetObjectField(options, gOptions_outColorSpaceFieldID,
306                GraphicsJNI::getColorSpace(env, decodeColorSpace, decodeColorType));
307
308        if (onlyDecodeSize) {
309            return nullptr;
310        }
311    }
312
313    // Scale is necessary due to density differences.
314    if (scale != 1.0f) {
315        willScale = true;
316        scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);
317        scaledHeight = static_cast<int>(scaledHeight * scale + 0.5f);
318    }
319
320    android::Bitmap* reuseBitmap = nullptr;
321    unsigned int existingBufferSize = 0;
322    if (javaBitmap != NULL) {
323        reuseBitmap = &bitmap::toBitmap(env, javaBitmap);
324        if (reuseBitmap->isImmutable()) {
325            ALOGW("Unable to reuse an immutable bitmap as an image decoder target.");
326            javaBitmap = NULL;
327            reuseBitmap = nullptr;
328        } else {
329            existingBufferSize = bitmap::getBitmapAllocationByteCount(env, javaBitmap);
330        }
331    }
332
333    HeapAllocator defaultAllocator;
334    RecyclingPixelAllocator recyclingAllocator(reuseBitmap, existingBufferSize);
335    ScaleCheckingAllocator scaleCheckingAllocator(scale, existingBufferSize);
336    SkBitmap::HeapAllocator heapAllocator;
337    SkBitmap::Allocator* decodeAllocator;
338    if (javaBitmap != nullptr && willScale) {
339        // This will allocate pixels using a HeapAllocator, since there will be an extra
340        // scaling step that copies these pixels into Java memory.  This allocator
341        // also checks that the recycled javaBitmap is large enough.
342        decodeAllocator = &scaleCheckingAllocator;
343    } else if (javaBitmap != nullptr) {
344        decodeAllocator = &recyclingAllocator;
345    } else if (willScale || isHardware) {
346        // This will allocate pixels using a HeapAllocator,
347        // for scale case: there will be an extra scaling step.
348        // for hardware case: there will be extra swizzling & upload to gralloc step.
349        decodeAllocator = &heapAllocator;
350    } else {
351        decodeAllocator = &defaultAllocator;
352    }
353
354    SkAlphaType alphaType = codec->computeOutputAlphaType(requireUnpremultiplied);
355
356    const SkImageInfo decodeInfo = SkImageInfo::Make(size.width(), size.height(),
357            decodeColorType, alphaType, decodeColorSpace);
358
359    // For wide gamut images, we will leave the color space on the SkBitmap.  Otherwise,
360    // use the default.
361    SkImageInfo bitmapInfo = decodeInfo;
362    if (decodeInfo.colorSpace() && decodeInfo.colorSpace()->isSRGB()) {
363        bitmapInfo = bitmapInfo.makeColorSpace(GraphicsJNI::colorSpaceForType(decodeColorType));
364    }
365
366    if (decodeColorType == kGray_8_SkColorType) {
367        // The legacy implementation of BitmapFactory used kAlpha8 for
368        // grayscale images (before kGray8 existed).  While the codec
369        // recognizes kGray8, we need to decode into a kAlpha8 bitmap
370        // in order to avoid a behavior change.
371        bitmapInfo =
372                bitmapInfo.makeColorType(kAlpha_8_SkColorType).makeAlphaType(kPremul_SkAlphaType);
373    }
374    SkBitmap decodingBitmap;
375    if (!decodingBitmap.setInfo(bitmapInfo) ||
376            !decodingBitmap.tryAllocPixels(decodeAllocator)) {
377        // SkAndroidCodec should recommend a valid SkImageInfo, so setInfo()
378        // should only only fail if the calculated value for rowBytes is too
379        // large.
380        // tryAllocPixels() can fail due to OOM on the Java heap, OOM on the
381        // native heap, or the recycled javaBitmap being too small to reuse.
382        return nullptr;
383    }
384
385    // Use SkAndroidCodec to perform the decode.
386    SkAndroidCodec::AndroidOptions codecOptions;
387    codecOptions.fZeroInitialized = decodeAllocator == &defaultAllocator ?
388            SkCodec::kYes_ZeroInitialized : SkCodec::kNo_ZeroInitialized;
389    codecOptions.fSampleSize = sampleSize;
390    SkCodec::Result result = codec->getAndroidPixels(decodeInfo, decodingBitmap.getPixels(),
391            decodingBitmap.rowBytes(), &codecOptions);
392    switch (result) {
393        case SkCodec::kSuccess:
394        case SkCodec::kIncompleteInput:
395            break;
396        default:
397            return nullObjectReturn("codec->getAndroidPixels() failed.");
398    }
399
400    // This is weird so let me explain: we could use the scale parameter
401    // directly, but for historical reasons this is how the corresponding
402    // Dalvik code has always behaved. We simply recreate the behavior here.
403    // The result is slightly different from simply using scale because of
404    // the 0.5f rounding bias applied when computing the target image size
405    const float scaleX = scaledWidth / float(decodingBitmap.width());
406    const float scaleY = scaledHeight / float(decodingBitmap.height());
407
408    jbyteArray ninePatchChunk = NULL;
409    if (peeker.mPatch != NULL) {
410        if (willScale) {
411            peeker.scale(scaleX, scaleY, scaledWidth, scaledHeight);
412        }
413
414        size_t ninePatchArraySize = peeker.mPatch->serializedSize();
415        ninePatchChunk = env->NewByteArray(ninePatchArraySize);
416        if (ninePatchChunk == NULL) {
417            return nullObjectReturn("ninePatchChunk == null");
418        }
419
420        jbyte* array = (jbyte*) env->GetPrimitiveArrayCritical(ninePatchChunk, NULL);
421        if (array == NULL) {
422            return nullObjectReturn("primitive array == null");
423        }
424
425        memcpy(array, peeker.mPatch, peeker.mPatchSize);
426        env->ReleasePrimitiveArrayCritical(ninePatchChunk, array, 0);
427    }
428
429    jobject ninePatchInsets = NULL;
430    if (peeker.mHasInsets) {
431        ninePatchInsets = peeker.createNinePatchInsets(env, scale);
432        if (ninePatchInsets == NULL) {
433            return nullObjectReturn("nine patch insets == null");
434        }
435        if (javaBitmap != NULL) {
436            env->SetObjectField(javaBitmap, gBitmap_ninePatchInsetsFieldID, ninePatchInsets);
437        }
438    }
439
440    SkBitmap outputBitmap;
441    if (willScale) {
442        // Set the allocator for the outputBitmap.
443        SkBitmap::Allocator* outputAllocator;
444        if (javaBitmap != nullptr) {
445            outputAllocator = &recyclingAllocator;
446        } else {
447            outputAllocator = &defaultAllocator;
448        }
449
450        SkColorType scaledColorType = decodingBitmap.colorType();
451        // FIXME: If the alphaType is kUnpremul and the image has alpha, the
452        // colors may not be correct, since Skia does not yet support drawing
453        // to/from unpremultiplied bitmaps.
454        outputBitmap.setInfo(
455                bitmapInfo.makeWH(scaledWidth, scaledHeight).makeColorType(scaledColorType));
456        if (!outputBitmap.tryAllocPixels(outputAllocator)) {
457            // This should only fail on OOM.  The recyclingAllocator should have
458            // enough memory since we check this before decoding using the
459            // scaleCheckingAllocator.
460            return nullObjectReturn("allocation failed for scaled bitmap");
461        }
462
463        SkPaint paint;
464        // kSrc_Mode instructs us to overwrite the uninitialized pixels in
465        // outputBitmap.  Otherwise we would blend by default, which is not
466        // what we want.
467        paint.setBlendMode(SkBlendMode::kSrc);
468        paint.setFilterQuality(kLow_SkFilterQuality); // bilinear filtering
469
470        SkCanvas canvas(outputBitmap, SkCanvas::ColorBehavior::kLegacy);
471        canvas.scale(scaleX, scaleY);
472        canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);
473    } else {
474        outputBitmap.swap(decodingBitmap);
475    }
476
477    if (padding) {
478        peeker.getPadding(env, padding);
479    }
480
481    // If we get here, the outputBitmap should have an installed pixelref.
482    if (outputBitmap.pixelRef() == NULL) {
483        return nullObjectReturn("Got null SkPixelRef");
484    }
485
486    if (!isMutable && javaBitmap == NULL) {
487        // promise we will never change our pixels (great for sharing and pictures)
488        outputBitmap.setImmutable();
489    }
490
491    bool isPremultiplied = !requireUnpremultiplied;
492    if (javaBitmap != nullptr) {
493        bitmap::reinitBitmap(env, javaBitmap, outputBitmap.info(), isPremultiplied);
494        outputBitmap.notifyPixelsChanged();
495        // If a java bitmap was passed in for reuse, pass it back
496        return javaBitmap;
497    }
498
499    int bitmapCreateFlags = 0x0;
500    if (isMutable) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Mutable;
501    if (isPremultiplied) bitmapCreateFlags |= android::bitmap::kBitmapCreateFlag_Premultiplied;
502
503    if (isHardware) {
504        sk_sp<Bitmap> hardwareBitmap = Bitmap::allocateHardwareBitmap(outputBitmap);
505        if (!hardwareBitmap.get()) {
506            return nullObjectReturn("Failed to allocate a hardware bitmap");
507        }
508        return bitmap::createBitmap(env, hardwareBitmap.release(), bitmapCreateFlags,
509                ninePatchChunk, ninePatchInsets, -1);
510    }
511
512    // 创建Java的bitmap并返回结果
513    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
514            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
515}
```

### 总结

缩放比例：__scaledsize = nTargetDensity / inDensity__

图片大小：__imageSize = (width * scaledsize) * (height * scaledsize) * pixelSize__