---
layout:     post
title:      "Android源码系列(18) -- GlobalScreenshot"
date:       2018-12-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

这篇文章介绍Android如何实现屏幕截取操作，为截取事件提供思路。在下篇文章[Android源码系列(18) -- TakeScreenshotService](/2019/01/08/TakeScreenshotService/)将介绍截图如何写入系统磁盘。源码版本 __Android 28__。

## 一、TakeScreenshotService

__TakeScreenshotService__ 是 __Service__，通过IPC的方式接受截屏请求，并通过 __GlobalScreenshot__ 实现屏幕截取和图片保存。

```java
public class TakeScreenshotService extends Service {
    private static final String TAG = "TakeScreenshotService";

    private static GlobalScreenshot mScreenshot;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            final Messenger callback = msg.replyTo;
            // 构建finisher响应Messenger
            Runnable finisher = new Runnable() {
                @Override
                public void run() {
                    Message reply = Message.obtain(null, 1);
                    try {
                        callback.send(reply);
                    } catch (RemoteException e) {
                    }
                }
            };

            // 如果此用户的存储被锁定无法保存屏幕截图，跳过执行而不是显示误导性的动画和错误通知
            if (!getSystemService(UserManager.class).isUserUnlocked()) {
                Log.w(TAG, "Skipping screenshot because storage is locked!");
                // 截图没有保存，发送finisher
                post(finisher);
                return;
            }

            // 初始化GlobalScreenshot
            if (mScreenshot == null) {
                mScreenshot = new GlobalScreenshot(TakeScreenshotService.this);
            }

            // 获取消息类型，根据类型执行操作
            switch (msg.what) {
                // 全屏截取
                case WindowManager.TAKE_SCREENSHOT_FULLSCREEN:
                    mScreenshot.takeScreenshot(finisher, msg.arg1 > 0, msg.arg2 > 0);
                    break;

                // 局部屏幕截取
                case WindowManager.TAKE_SCREENSHOT_SELECTED_REGION:
                    mScreenshot.takeScreenshotPartial(finisher, msg.arg1 > 0, msg.arg2 > 0);
                    break;
                    
                // 不支持类型，输出日志后不执行操作
                default:
                    Log.d(TAG, "Invalid screenshot option: " + msg.what);
            }
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return new Messenger(mHandler).getBinder();
    }

    @Override
    public boolean onUnbind(Intent intent) {
        if (mScreenshot != null) mScreenshot.stopScreenshot();
        return true;
    }
}
```

## 二、GlobalScreenshot

此类负责获取截图，下面出现的定义类都是 __GlobalScreenshot__ 的内部类。

```java
class GlobalScreenshot
```

#### 2.1 常量

```java
static final String SCREENSHOT_URI_ID = "android:screenshot_uri_id";
static final String SHARING_INTENT = "android:screenshot_sharing_intent";

private static final int SCREENSHOT_FLASH_TO_PEAK_DURATION = 130;
private static final int SCREENSHOT_DROP_IN_DURATION = 430;
private static final int SCREENSHOT_DROP_OUT_DELAY = 500;
private static final int SCREENSHOT_DROP_OUT_DURATION = 430;
private static final int SCREENSHOT_DROP_OUT_SCALE_DURATION = 370;
private static final int SCREENSHOT_FAST_DROP_OUT_DURATION = 320;
private static final float BACKGROUND_ALPHA = 0.5f;
private static final float SCREENSHOT_SCALE = 1f;
private static final float SCREENSHOT_DROP_IN_MIN_SCALE = SCREENSHOT_SCALE * 0.725f;
private static final float SCREENSHOT_DROP_OUT_MIN_SCALE = SCREENSHOT_SCALE * 0.45f;
private static final float SCREENSHOT_FAST_DROP_OUT_MIN_SCALE = SCREENSHOT_SCALE * 0.6f;
private static final float SCREENSHOT_DROP_OUT_MIN_SCALE_OFFSET = 0f;
```

#### 2.2 数据成员

```java
// 预览图宽高
private final int mPreviewWidth;
private final int mPreviewHeight;

private Context mContext;
private WindowManager mWindowManager;
private WindowManager.LayoutParams mWindowLayoutParams;
private NotificationManager mNotificationManager;

// 用于测量屏幕宽高
private Display mDisplay;
private DisplayMetrics mDisplayMetrics;
private Matrix mDisplayMatrix;

// 截图的Bitmap
private Bitmap mScreenBitmap;
private View mScreenshotLayout;

// 截图选择器
private ScreenshotSelectorView mScreenshotSelectorView;
private ImageView mBackgroundView;
private ImageView mScreenshotView;
private ImageView mScreenshotFlash;

// 截屏的屏幕动画
private AnimatorSet mScreenshotAnimation;

private int mNotificationIconSize;
private float mBgPadding;
private float mBgPaddingScale;

// 异步保存截图的AsyncTask
private AsyncTask<Void, Void, Void> mSaveInBgTask;

// 截屏时发出模拟快门的声音
private MediaActionSound mCameraSound;
```

#### 2.3 构造方法

```java
public GlobalScreenshot(Context context) {
    Resources r = context.getResources();
    mContext = context;
    LayoutInflater layoutInflater = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

    mDisplayMatrix = new Matrix();
    
    // 填充截屏布局
    mScreenshotLayout = layoutInflater.inflate(R.layout.global_screenshot, null);
    // 绑定View
    mBackgroundView = (ImageView) mScreenshotLayout.findViewById(R.id.global_screenshot_background);
    mScreenshotView = (ImageView) mScreenshotLayout.findViewById(R.id.global_screenshot);
    mScreenshotFlash = (ImageView) mScreenshotLayout.findViewById(R.id.global_screenshot_flash);
    mScreenshotSelectorView = (ScreenshotSelectorView) mScreenshotLayout.findViewById(
            R.id.global_screenshot_selector);
    // 令此布局获取焦点
    mScreenshotLayout.setFocusable(true);
    mScreenshotSelectorView.setFocusable(true);
    mScreenshotSelectorView.setFocusableInTouchMode(true);
    mScreenshotLayout.setOnTouchListener(new View.OnTouchListener() {
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            // 拦截并抛弃所有触摸事件
            return true;
        }
    });

    // 设置将要使用的window
    mWindowLayoutParams = new WindowManager.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT, 0, 0,
            WindowManager.LayoutParams.TYPE_SCREENSHOT,
            WindowManager.LayoutParams.FLAG_FULLSCREEN
                | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED,
            PixelFormat.TRANSLUCENT);
    mWindowLayoutParams.setTitle("ScreenshotAnimation");
    mWindowLayoutParams.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    mNotificationManager =
        (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    mDisplay = mWindowManager.getDefaultDisplay();
    mDisplayMetrics = new DisplayMetrics();
    mDisplay.getRealMetrics(mDisplayMetrics);

    // 获取各自的目标尺寸
    mNotificationIconSize =
        r.getDimensionPixelSize(android.R.dimen.notification_large_icon_height);

    // 范围占背景的两边
    mBgPadding = (float) r.getDimensionPixelSize(R.dimen.global_screenshot_bg_padding);
    mBgPaddingScale = mBgPadding /  mDisplayMetrics.widthPixels;

    // 确定最优化的预览尺寸
    int panelWidth = 0;
    try {
        panelWidth = r.getDimensionPixelSize(R.dimen.notification_panel_width);
    } catch (Resources.NotFoundException e) {
    }
    // panelWidth在上述异常出现时为0
    if (panelWidth <= 0) {
        // includes notification_panel_width==match_parent (-1)
        panelWidth = mDisplayMetrics.widthPixels;
    }
    mPreviewWidth = panelWidth;
    mPreviewHeight = r.getDimensionPixelSize(R.dimen.notification_max_height);

    // 加载快门声音
    mCameraSound = new MediaActionSound();
    mCameraSound.load(MediaActionSound.SHUTTER_CLICK);
}
```

#### 2.4 saveScreenshotInWorkerThread

调用方法时截图已经保存在内存中。此方法会创建新工作任务，并在子线程把截图保存到媒体存储。__SaveImageInBackgroundTask__ 继承 __AsyncTask__。

```java
private void saveScreenshotInWorkerThread(Runnable finisher) {
    SaveImageInBackgroundData data = new SaveImageInBackgroundData();
    data.context = mContext;
    data.image = mScreenBitmap; // 截图
    data.iconSize = mNotificationIconSize;
    data.finisher = finisher;
    data.previewWidth = mPreviewWidth;
    data.previewheight = mPreviewHeight;
    if (mSaveInBgTask != null) {
        mSaveInBgTask.cancel(false);
    }
    // 由此.execute()可知任务在AsyncTask是串行执行的
    mSaveInBgTask = new SaveImageInBackgroundTask(mContext, data, mNotificationManager)
            .execute();
}
```

系统很多任务通过 __AsyncTask__ 而不是子线程的方式执行后台任务，类似上面的截图写入到存储的场景。所以一定不能把长耗时任务放入 __AsyncTask__ 导致任务阻塞，或依赖 __AsyncTask__ 完成实时性要求高的工作。

#### 2.5 getDegreesForRotation

获取屏幕旋转角度

```java
private float getDegreesForRotation(int value) {
    switch (value) {
    case Surface.ROTATION_90:
        return 360f - 90f;
    case Surface.ROTATION_180:
        return 360f - 180f;
    case Surface.ROTATION_270:
        return 360f - 270f;
    }
    return 0f;
}
```

#### 2.6 takeScreenshot

__GlobalScreenshot__ 初始化完成后，即可截取当前屏幕并展示动画。方法使用 __private__ 修饰，由同类的其他方法调用。

```java
private void takeScreenshot(Runnable finisher, boolean statusBarVisible, boolean navBarVisible,
        Rect crop) {
    // 获取屏幕的旋转角度、宽、高
    int rot = mDisplay.getRotation();
    int width = crop.width();
    int height = crop.height();

    // 截取屏幕
    mScreenBitmap = SurfaceControl.screenshot(crop, width, height, rot);

    // 检查获取的屏幕截图是否为空
    if (mScreenBitmap == null) {
        notifyScreenshotError(mContext, mNotificationManager,
                R.string.screenshot_failed_to_capture_text);
        finisher.run();
        return;
    }

    // 截图设置为非透明图片，能提升绘制速度
    mScreenBitmap.setHasAlpha(false);
    mScreenBitmap.prepareToDraw();

    // 开始截屏后的动画
    startAnimation(finisher, mDisplayMetrics.widthPixels, mDisplayMetrics.heightPixels,
            statusBarVisible, navBarVisible);
}
```

以下方法截取全屏图片，调用了上面的方法。根据屏幕宽高创建一个 __Rect__。

```java
void takeScreenshot(Runnable finisher, boolean statusBarVisible, boolean navBarVisible) {
    mDisplay.getRealMetrics(mDisplayMetrics);
    takeScreenshot(finisher, statusBarVisible, navBarVisible,
            new Rect(0, 0, mDisplayMetrics.widthPixels, mDisplayMetrics.heightPixels));
}
```

#### 2.7 takeScreenshotPartial

__takeScreenshot()__ 截取全屏，此方法能截取屏幕的部分区域。

```java
void takeScreenshotPartial(final Runnable finisher, final boolean statusBarVisible,
        final boolean navBarVisible) {
    // 向WindowManager添加选择器布局
    mWindowManager.addView(mScreenshotLayout, mWindowLayoutParams);
    // 处理选择器的点击事件
    mScreenshotSelectorView.setOnTouchListener(new View.OnTouchListener() {
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            ScreenshotSelectorView view = (ScreenshotSelectorView) v;
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    view.startSelection((int) event.getX(), (int) event.getY());
                    return true;
                case MotionEvent.ACTION_MOVE:
                    view.updateSelection((int) event.getX(), (int) event.getY());
                    return true;
                case MotionEvent.ACTION_UP:
                    view.setVisibility(View.GONE);
                    mWindowManager.removeView(mScreenshotLayout);
                    // 获取选择的矩形区域
                    final Rect rect = view.getSelectionRect();
                    if (rect != null) {
                        if (rect.width() != 0 && rect.height() != 0) {
                            // 在view消失之后需要mScreenshotLayout处理截图保存的任务
                            mScreenshotLayout.post(new Runnable() {
                                public void run() {
                                    // 把选中的矩形区域作为依据从全屏截取部分图像
                                    takeScreenshot(finisher, statusBarVisible, navBarVisible,
                                            rect);
                                }
                            });
                        }
                    }

                    view.stopSelection();
                    return true;
            }

            return false;
        }
    });
    mScreenshotLayout.post(new Runnable() {
        @Override
        public void run() {
            mScreenshotSelectorView.setVisibility(View.VISIBLE);
            mScreenshotSelectorView.requestFocus();
        }
    });
}
```

#### 2.8 stopScreenshot

停止截屏

```java
void stopScreenshot() {
    // 如果选择器图层依然呈现在屏幕上，则将其移除并重置其状态
    if (mScreenshotSelectorView.getSelectionRect() != null) {
        mWindowManager.removeView(mScreenshotLayout);
        mScreenshotSelectorView.stopSelection();
    }
}
```

#### 2.9 startAnimation

截图动画

```java
private void startAnimation(final Runnable finisher, int w, int h, boolean statusBarVisible,
        boolean navBarVisible) {
    // 手机处于省电模式的话显示一个toast提示用于已截屏
    PowerManager powerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
    if (powerManager.isPowerSaveMode()) {
        Toast.makeText(mContext, R.string.screenshot_saved_title, Toast.LENGTH_SHORT).show();
    }

    // 添加动画视图
    mScreenshotView.setImageBitmap(mScreenBitmap);
    mScreenshotLayout.requestFocus();

    // 使用刚拍摄的屏幕截图设置动画，如动画已启动则需要结束
    if (mScreenshotAnimation != null) {
        if (mScreenshotAnimation.isStarted()) {
            mScreenshotAnimation.end();
        }
        mScreenshotAnimation.removeAllListeners();
    }

    mWindowManager.addView(mScreenshotLayout, mWindowLayoutParams);
    // 通过代码构建动画集合并组装
    ValueAnimator screenshotDropInAnim = createScreenshotDropInAnimation();
    ValueAnimator screenshotFadeOutAnim = createScreenshotDropOutAnimation(w, h,
            statusBarVisible, navBarVisible);
    mScreenshotAnimation = new AnimatorSet();
    mScreenshotAnimation.playSequentially(screenshotDropInAnim, screenshotFadeOutAnim);
    mScreenshotAnimation.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            // 动画播放结束时启动保存截图的任务
            saveScreenshotInWorkerThread(finisher);
            // 截屏的布局也可以从屏幕移除了
            mWindowManager.removeView(mScreenshotLayout);

            // 清除位图的引用，避免内存泄漏
            mScreenBitmap = null;
            mScreenshotView.setImageBitmap(null);
        }
    });
    mScreenshotLayout.post(new Runnable() {
        @Override
        public void run() {
            // 播放快门声通知用户已截屏
            mCameraSound.play(MediaActionSound.SHUTTER_CLICK);
            // 通过硬件加速播放动画
            mScreenshotView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
            mScreenshotView.buildLayer();
            mScreenshotAnimation.start();
        }
    });
}
```

通过代码构造动画

```java
private ValueAnimator createScreenshotDropInAnimation() {
    final float flashPeakDurationPct = ((float) (SCREENSHOT_FLASH_TO_PEAK_DURATION)
            / SCREENSHOT_DROP_IN_DURATION);
    final float flashDurationPct = 2f * flashPeakDurationPct;
    final Interpolator flashAlphaInterpolator = new Interpolator() {
        @Override
        public float getInterpolation(float x) {
            // Flash the flash view in and out quickly
            if (x <= flashDurationPct) {
                return (float) Math.sin(Math.PI * (x / flashDurationPct));
            }
            return 0;
        }
    };
    final Interpolator scaleInterpolator = new Interpolator() {
        @Override
        public float getInterpolation(float x) {
            // We start scaling when the flash is at it's peak
            if (x < flashPeakDurationPct) {
                return 0;
            }
            return (x - flashDurationPct) / (1f - flashDurationPct);
        }
    };
    ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);
    anim.setDuration(SCREENSHOT_DROP_IN_DURATION);
    anim.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationStart(Animator animation) {
            mBackgroundView.setAlpha(0f);
            mBackgroundView.setVisibility(View.VISIBLE);
            mScreenshotView.setAlpha(0f);
            mScreenshotView.setTranslationX(0f);
            mScreenshotView.setTranslationY(0f);
            mScreenshotView.setScaleX(SCREENSHOT_SCALE + mBgPaddingScale);
            mScreenshotView.setScaleY(SCREENSHOT_SCALE + mBgPaddingScale);
            mScreenshotView.setVisibility(View.VISIBLE);
            mScreenshotFlash.setAlpha(0f);
            mScreenshotFlash.setVisibility(View.VISIBLE);
        }
        @Override
        public void onAnimationEnd(android.animation.Animator animation) {
            mScreenshotFlash.setVisibility(View.GONE);
        }
    });
    anim.addUpdateListener(new AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            float t = (Float) animation.getAnimatedValue();
            float scaleT = (SCREENSHOT_SCALE + mBgPaddingScale)
                - scaleInterpolator.getInterpolation(t)
                    * (SCREENSHOT_SCALE - SCREENSHOT_DROP_IN_MIN_SCALE);
            mBackgroundView.setAlpha(scaleInterpolator.getInterpolation(t) * BACKGROUND_ALPHA);
            mScreenshotView.setAlpha(t);
            mScreenshotView.setScaleX(scaleT);
            mScreenshotView.setScaleY(scaleT);
            mScreenshotFlash.setAlpha(flashAlphaInterpolator.getInterpolation(t));
        }
    });
    return anim;
}
```

通过代码构造动画

```java
private ValueAnimator createScreenshotDropOutAnimation(int w, int h, boolean statusBarVisible,
        boolean navBarVisible) {
    ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);
    anim.setStartDelay(SCREENSHOT_DROP_OUT_DELAY);
    anim.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            mBackgroundView.setVisibility(View.GONE);
            mScreenshotView.setVisibility(View.GONE);
            mScreenshotView.setLayerType(View.LAYER_TYPE_NONE, null);
        }
    });

    if (!statusBarVisible || !navBarVisible) {
        // 没有状态栏或导航栏，截屏提示直接淡出屏幕
        anim.setDuration(SCREENSHOT_FAST_DROP_OUT_DURATION);
        anim.addUpdateListener(new AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float t = (Float) animation.getAnimatedValue();
                float scaleT = (SCREENSHOT_DROP_IN_MIN_SCALE + mBgPaddingScale)
                        - t * (SCREENSHOT_DROP_IN_MIN_SCALE - SCREENSHOT_FAST_DROP_OUT_MIN_SCALE);
                mBackgroundView.setAlpha((1f - t) * BACKGROUND_ALPHA);
                mScreenshotView.setAlpha(1f - t);
                mScreenshotView.setScaleX(scaleT);
                mScreenshotView.setScaleY(scaleT);
            }
        });
    } else {
        // In the case where there is a status bar, animate to the origin of the bar (top-left)
        // 屏幕上有状态栏，动画仅状态栏的左上方
        final float scaleDurationPct = (float) SCREENSHOT_DROP_OUT_SCALE_DURATION
                / SCREENSHOT_DROP_OUT_DURATION;
        final Interpolator scaleInterpolator = new Interpolator() {
            @Override
            public float getInterpolation(float x) {
                if (x < scaleDurationPct) {
                    // Decelerate, and scale the input accordingly
                    return (float) (1f - Math.pow(1f - (x / scaleDurationPct), 2f));
                }
                return 1f;
            }
        };

        // Determine the bounds of how to scale
        float halfScreenWidth = (w - 2f * mBgPadding) / 2f;
        float halfScreenHeight = (h - 2f * mBgPadding) / 2f;
        final float offsetPct = SCREENSHOT_DROP_OUT_MIN_SCALE_OFFSET;
        final PointF finalPos = new PointF(
            -halfScreenWidth + (SCREENSHOT_DROP_OUT_MIN_SCALE + offsetPct) * halfScreenWidth,
            -halfScreenHeight + (SCREENSHOT_DROP_OUT_MIN_SCALE + offsetPct) * halfScreenHeight);

        // 截图通过动画移动到status bar
        anim.setDuration(SCREENSHOT_DROP_OUT_DURATION);
        anim.addUpdateListener(new AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float t = (Float) animation.getAnimatedValue();
                float scaleT = (SCREENSHOT_DROP_IN_MIN_SCALE + mBgPaddingScale)
                    - scaleInterpolator.getInterpolation(t)
                        * (SCREENSHOT_DROP_IN_MIN_SCALE - SCREENSHOT_DROP_OUT_MIN_SCALE);
                mBackgroundView.setAlpha((1f - t) * BACKGROUND_ALPHA);
                mScreenshotView.setAlpha(1f - scaleInterpolator.getInterpolation(t));
                mScreenshotView.setScaleX(scaleT);
                mScreenshotView.setScaleY(scaleT);
                mScreenshotView.setTranslationX(t * finalPos.x);
                mScreenshotView.setTranslationY(t * finalPos.y);
            }
        });
    }
    return anim;
}
```

#### 2.10 notifyScreenshotError 

截屏出现错误通知用户

```java
static void notifyScreenshotError(Context context, NotificationManager nManager, int msgResId) {
    Resources r = context.getResources();
    String errorMsg = r.getString(msgResId);

    // 重新利用现有通知以通知用户错误信息
    Notification.Builder b = new Notification.Builder(context, NotificationChannels.ALERTS)
        .setTicker(r.getString(R.string.screenshot_failed_title))
        .setContentTitle(r.getString(R.string.screenshot_failed_title))
        .setContentText(errorMsg)
        .setSmallIcon(R.drawable.stat_notify_image_error)
        .setWhen(System.currentTimeMillis())
        .setVisibility(Notification.VISIBILITY_PUBLIC) // ok to show outside lockscreen
        .setCategory(Notification.CATEGORY_ERROR)
        .setAutoCancel(true)
        .setColor(context.getColor(
                    com.android.internal.R.color.system_notification_accent_color));

    final DevicePolicyManager dpm = (DevicePolicyManager) context.getSystemService(
            Context.DEVICE_POLICY_SERVICE);
    final Intent intent = dpm.createAdminSupportIntent(
            DevicePolicyManager.POLICY_DISABLE_SCREEN_CAPTURE);
    if (intent != null) {
        final PendingIntent pendingIntent = PendingIntent.getActivityAsUser(
                context, 0, intent, 0, null, UserHandle.CURRENT);
        b.setContentIntent(pendingIntent);
    }

    SystemUI.overrideNotificationAppName(context, b, true);

    Notification n = new Notification.BigTextStyle(b)
            .bigText(errorMsg)
            .build();
    nManager.notify(SystemMessage.NOTE_GLOBAL_SCREENSHOT, n);
}
```

#### 2.11 ScreenshotActionReceiver

代理分享或编辑intent的Receiver

```java
public static class ScreenshotActionReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        try {
            ActivityManager.getService().closeSystemDialogs(SYSTEM_DIALOG_REASON_SCREENSHOT);
        } catch (RemoteException e) {
        }

        Intent actionIntent = intent.getParcelableExtra(SHARING_INTENT);

        // If this is an edit & default editor exists, route straight there.
        String editorPackage = context.getResources().getString(R.string.config_screenshotEditor);
        if (actionIntent.getAction() == Intent.ACTION_EDIT &&
                editorPackage != null && editorPackage.length() > 0) {
            actionIntent.setComponent(ComponentName.unflattenFromString(editorPackage));
            final NotificationManager nm =
                    (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
            nm.cancel(SystemMessage.NOTE_GLOBAL_SCREENSHOT);
        } else {
            PendingIntent chooseAction = PendingIntent.getBroadcast(context, 0,
                    new Intent(context, GlobalScreenshot.TargetChosenReceiver.class),
                    PendingIntent.FLAG_CANCEL_CURRENT | PendingIntent.FLAG_ONE_SHOT);
            actionIntent = Intent.createChooser(actionIntent, null,
                    chooseAction.getIntentSender())
                    .addFlags(Intent.FLAG_ACTIVITY_CLEAR_TASK | Intent.FLAG_ACTIVITY_NEW_TASK);
        }

        ActivityOptions opts = ActivityOptions.makeBasic();
        opts.setDisallowEnterPictureInPictureWhileLaunching(true);

        context.startActivityAsUser(actionIntent, opts.toBundle(), UserHandle.CURRENT);
    }
}
```

#### 2.12 TargetChosenReceiver

选择分享或编辑目标后移除截图的通知

```java
public static class TargetChosenReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // 移除通知
        final NotificationManager nm =
                (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
        nm.cancel(SystemMessage.NOTE_GLOBAL_SCREENSHOT);
    }
}
```

#### 2.13 DeleteScreenshotReceiver

从存储里移除截图

```java
public static class DeleteScreenshotReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (!intent.hasExtra(SCREENSHOT_URI_ID)) {
            return;
        }

        // 移除通知，先获取NotificationManager
        final NotificationManager nm =
                (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
        // 获取SCREENSHOT_URI_ID，构建Uri
        final Uri uri = Uri.parse(intent.getStringExtra(SCREENSHOT_URI_ID));
        // 移除截屏通知
        nm.cancel(SystemMessage.NOTE_GLOBAL_SCREENSHOT);

        // 从媒体存储中删除图片，后台任务串行执行
        new DeleteImageInBackgroundTask(context).execute(uri);
    }
}
```

