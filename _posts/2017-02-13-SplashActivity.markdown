---
layout:     post
title:      "简单Android闪屏页控制逻辑"
date:       2017-02-13
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android
---

```java
public final class SplashActivity extends Activity {

    public static final String SHOW_SPLASH = "SHOW_SPLASH";
    private static final int START_ACTIVITY = 0x01;
    private static final int SPLASH_DURATION = 3000; // 闪屏展示时长，ms
    private static final int NO_SPLASH_DURATION = 0; // 闪屏时长0ms

    private SplashHandler mHandler;

    /**
     * 软件启动是否展示闪屏页，startSplash:
     *     true  :  跳过闪屏页
     *     false :  不跳过闪屏（default）
     * 建议在App设置中控制此SharedPrefs变量，参考Bilibili Android客户端
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_splash);
        mHandler = new SplashHandler(Looper.getMainLooper());

        boolean startSplash = SharedPrefsUtil.getBoolean(SHOW_SPLASH, false, this); // 使用SharedPrefs保存变量
        if (startSplash) {
            mHandler.sendEmptyMessageDelayed(START_ACTIVITY, NO_SPLASH_DURATION);
        } else {
            mHandler.sendEmptyMessageDelayed(START_ACTIVITY, SPLASH_DURATION);
        }
    }

    // 此自定义Handle修改SplashActivity跳转目的地
    class SplashHandler extends Handler {
        SplashHandler(Looper mLooper) {
            super(mLooper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Intent intent = new Intent(SplashActivity.this, MainActivity.class);
            startActivity(intent);
            overridePendingTransition(android.R.anim.fade_in, android.R.anim.fade_out); // 两个Activity跳转过程的渐变过度效果
            SplashActivity.this.finish();
        }
    }

    /**
     * 点击闪屏任一个位置直接跳过闪屏页
     * 一般在实机测试时不等待闪屏点击跳过
     */
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (event.getAction() == MotionEvent.ACTION_DOWN) {
            mHandler.removeMessages(START_ACTIVITY);  // 移除有延迟的Message，加入无等待时间Message
            mHandler.sendEmptyMessage(START_ACTIVITY);
        }
        return true;
    }

    // 配合点击跳过闪屏页功能使用
    @Override
    public void onBackPressed() {
    }
}
```

