---
layout:     post
title:      "Android学习笔记，持续更新"
date:       2017-02-15
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android
---

## 一、TabLayout

### 1.1 TabLayout常见XML属性

属性 	|说明
---- | ----
app:tabIndicatorColor 	|   tab滚动指示横线颜色
app:tabSelectedTextColor | 被选中tab子项的文本颜色
app:tabTextColor | tab子项默认的文本颜色

### 1.2 Java调用

```java
TabLayout mTabLayout = (TabLayout) findViewById(R.id.tabLayout);
TabLayout.Tab mContactsTab = mTabLayout.newTab().setText("Contacts");
mTabLayout.addTab(mContactsTab, true); // 添加Tab且默认选中
```

### 1.3 滚动模式

```java
mTabLayout.setTabMode(TabLayout.MODE_FIXED);
```

一共两种模式:

* TabLayout.MODE_FIXED (默认)
* TabLayout.MODE_SCROLLABLE


### 1.4 添加监听

```java
mTabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener() {
    @Override
    public void onTabSelected(TabLayout.Tab tab) {}
    
    @Override
    public void onTabUnselected(TabLayout.Tab tab) {}
    
    @Override
    public void onTabReselected(TabLayout.Tab tab) {}
});
```

### 1.5 TabGravity设置

只在TabMode为`TabLayout.MODE_FIXED`时才会生效。
`GRAVITY_FILL`占满`TabLayout`空间，`GRAVITY_CENTER`居中`Wrap_Content`。

```java
mTabLayout.setTabGravity(TabLayout.GRAVITY_FILL);
mTabLayout.setTabGravity(TabLayout.GRAVITY_CENTER);
```

### 1.6 ViewPager和TabLayout

在`mPagerAdapter`添加页面，绑定到`mViewPager`上。

```java
private ViewPagerAdapter mPagerAdapter;
private ViewPager mViewPager = (ViewPager) findViewById(R.id.viewPager);
private TabLayout mTabLayout = (TabLayout) findViewById(R.id.tabLayout);

mPagerAdapter = new ViewPagerAdapter(getSupportFragmentManager());
mPagerAdapter.addTab(new MainFragment(), "所有书签");
mPagerAdapter.addTab(new ReadingFragment(), "正在阅读");
mPagerAdapter.addTab(new ReadFragment(), "完成阅读");

mViewPager.setAdapter(mPagerAdapter);
mTabLayout.setupWithViewPager(viewPager, false);
```

使用的ViewPagerAdaptor

```java
public class ViewPagerAdapter extends FragmentPagerAdapter {

    private final List<Fragment> fragments = new ArrayList<>();
    private final List<String> titles = new ArrayList<>();

    public ViewPagerAdapter(FragmentManager fm) {
        super(fm);
    }

    public void addTab(Fragment fragment, String title) {
        fragments.add(fragment);
        titles.add(title);
    }

    public Fragment getFragments(int position) {
        return fragments.get(position);
    }

    @Override
    public Fragment getItem(int position) {
        return fragments.get(position);
    }

    @Override
    public int getCount() {
        return fragments.size();
    }

    @Override
    public CharSequence getPageTitle(int position) {
        return titles.get(position);
    }
}
```

## 二、DrawerLayout

### 2.1 DrawerLayout常用功能

获取抽屉状态和控制抽屉开闭

```java
mDrawerLayout.isDrawerOpen(GravityCompat.START); // boolean, 抽屉是否展开

mDrawerLayout.closeDrawer(GravityCompat.START);  // void, 收起抽屉
mDrawerLayout.openDrawer(GravityCompat.START);   // void, 打开抽屉
```

用法示例

```java
private DrawerLayout mDrawerLayout; // 声明控件
mDrawerLayout = (DrawerLayout) findViewById(R.id.drawer_layout); // 初始化控件

if (mDrawerLayout.isDrawerOpen(GravityCompat.START)) {
    mDrawerLayout.closeDrawer(GravityCompat.START);
}
```

### 2.2 DrawerLayout与startActivity冲突

参考自 [Optimizing drawer and activity launching speed](http://stackoverflow.com/questions/18343018/optimizing-drawer-and-activity-launching-speed)

```java
class SmoothDrawerToggle extends ActionBarDrawerToggle {
    private Runnable runnable;

    SmoothDrawerToggle(Activity activity, DrawerLayout drawerLayout, Toolbar toolbar) {
        super(activity, drawerLayout, toolbar, R.string.navigation_drawer_open, R.string.navigation_drawer_close);
    }

    @Override
    public void onDrawerOpened(View drawerView) {
        super.onDrawerOpened(drawerView);
        invalidateOptionsMenu();
    }

    @Override
    public void onDrawerClosed(View view) {
        super.onDrawerClosed(view);
        invalidateOptionsMenu();
    }

    @Override
    public void onDrawerStateChanged(int newState) {
        super.onDrawerStateChanged(newState);
        if (runnable != null && newState == DrawerLayout.STATE_IDLE) {
            runnable.run();
            runnable = null;
        }
    }

    void runWhenIdle(Runnable runnable) {
        this.runnable = runnable;
    }
}
```

```java
private SmoothDrawerToggle mToggle;
private DrawerLayout drawerLayout;
private NavigationView navView;
private Toolbar toolBar;

// 在onCreate()中初始化
navView.setNavigationItemSelectedListener(this);
// NavigationView.OnNavigationItemSelectedListener 
mToggle = new SmoothDrawerToggle(this, drawerLayout, toolBar);
// Activity.Context, 设置该抽屉xml的ID, toolBar用于抽屉开闭图标 
drawerLayout.addDrawerListener(mToggle);
// 给抽屉加入自定义的Listener
mToggle.syncState();
```
### 2.3 箭头颜色

```xml
<resources>
    <!-- DrawerArrowStyle -->
    <style name="DrawerArrowStyle" parent="Widget.AppCompat.DrawerArrowToggle">
        <item name="spinBars">true</item>
        <item name="color">#FFFFFF</item>
    </style>

    <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="drawerArrowStyle">@style/DrawerArrowStyle</item>
    </style>
</resources>
```


## 三、SplashActivity

### 闪屏页代码

```java
public class SplashActivity extends Activity {

    public static final String SHOW_SPLASH = "SHOW_SPLASH";
    private static final int START_ACTIVITY = 0x01;
    private static final int SPLASH_DURATION = 3000;
    private static final int NO_SPLASH_DURATION = 0;

    private SplashHandler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_splash);

        mHandler = new SplashHandler(Looper.getMainLooper());
        boolean startSplash = SharedPrefsUtil.getBoolean(SHOW_SPLASH, false, this);

        if (startSplash) {
            mHandler.sendEmptyMessageDelayed(START_ACTIVITY, NO_SPLASH_DURATION);
        } else {
            mHandler.sendEmptyMessageDelayed(START_ACTIVITY, SPLASH_DURATION);
        }
    }

    class SplashHandler extends Handler {
        SplashHandler(Looper mLooper) {
            super(mLooper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);

            Intent intent = new Intent(SplashActivity.this, MainActivity.class);
            startActivity(intent);
            //切换Activity的过程中渐变效果
            overridePendingTransition(android.R.anim.fade_in, android.R.anim.fade_out);
            SplashActivity.this.finish();
        }
    }
    
    // 单击提早关闭SplashActivity，选用
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (event.getAction() == MotionEvent.ACTION_DOWN) {
            mHandler.removeMessages(START_ACTIVITY);
            mHandler.sendEmptyMessage(START_ACTIVITY);
        }
        return true;
    }
    
    // 屏蔽返回按键
    @Override
    public void onBackPressed() {
    }
}
```

## 四、Utils

### 4.1 ToastUtil

避免多次连续使用Toast时一直显示Toast，为了避免内存泄漏，使用的Context请调用ApplicationContext.

```java
public final class ToastUtil {

    private static Toast mToast;
    private static String mLastMsg;
    private static long mFirstTime = 0;
    private static long mSecondTime = 0;

    public static void makeText(Context context, String msg, int duration) {
        if (mToast == null) {
            mToast = Toast.makeText(context, msg, duration);
            mToast.show();
            mFirstTime = System.currentTimeMillis();
        } else {
            mSecondTime = System.currentTimeMillis();
            if (msg.equals(mLastMsg)) {
                if (mSecondTime - mFirstTime > 2500) {
                    mToast.show();
                }
            } else {
                mLastMsg = msg;
                mToast.setText(msg);
                mToast.show();
            }
        }
        mFirstTime = mSecondTime;
    }

    public static void makeText(Context context, int text, int duration) {
        makeText(context, context.getString(text), duration);
    }

    public static void cancel(Context context) {
        if (mToast != null) {
            mToast.cancel();
        }
    }
}
```

### 4.2 DoubleClickUtil

双击返回键退出应用

```java
public class ClickHelper {

    private static long mLastClick = 0L;
    private static final int THRESHOLD = 2000;

    public static boolean check() {
        long now = System.currentTimeMillis();
        boolean exit = now - mLastClick < THRESHOLD;
        mLastClick = now;
        return exit;
    }
}
```

### 4.3 GlideCacheUtil

提供Glide磁盘缓存占用大小和清理功能

```java
public class GlideCacheUtil {
    /**
     * 公开的缓存清理方法
     *
     * @param context Activity context
     */
    public static void clearAll(Context context) {
        clearDisk(context);
        clearMemory(context);
        String cacheDir = context.getExternalCacheDir() + ExternalCacheDiskCacheFactory.DEFAULT_DISK_CACHE_DIR;
        clearFile(cacheDir, true);
    }

    /**
     * 获取Glide缓存大小的字符串
     *
     * @param context Activity context
     * @return 包含Glide缓存大小的字符串，如"324.49KB"
     */
    public static String getCacheSize(Context context) {
        File file = new File(context.getCacheDir() + "/" + InternalCacheDiskCacheFactory.DEFAULT_DISK_CACHE_DIR);
        double folderSize = getFolderSize(file);
        return formatSize(folderSize);
    }

    /**
     * 清除Glide磁盘缓存内容，必须运行在子线程，避免阻塞主线程
     *
     * @param context Activity context
     */
    private static void clearDisk(final Context context) {
        if (Looper.myLooper() == Looper.getMainLooper()) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Glide.get(context).clearDiskCache();
                }
            }).start();
        } else {
            Glide.get(context).clearDiskCache();
        }
    }

    /**
     * 清除Glide内存的缓存内容，必须运行在主线程
     *
     * @param context Activity context
     */
    private static void clearMemory(final Context context) {
        if (Looper.myLooper() == Looper.getMainLooper()) {
            Glide.get(context).clearMemory();
        }
    }

    /**
     * 统计文件夹大小
     *
     * @param file 文件夹路径
     * @return 返回文件夹大小，单位字节
     */
    private static double getFolderSize(File file) {
        File[] list = file.listFiles();

        if (list == null) {
            return 0;
        } else {
            double size = 0;

            for (File f : list) {
                size += (f.isDirectory() ? getFolderSize(f) : f.length());
            }
            return size;
        }
    }

    /**
     * 清除缓存文件
     *
     * @param filePath       缓存路径
     * @param deleteThisPath 是否删除缓存根目录
     * @return 删除结果布尔标志
     */
    private static boolean clearFile(String filePath, boolean deleteThisPath) {
        boolean flag = false;
        if (!Utils.isEmpty(filePath)) {
            try {
                File file = new File(filePath);
                if (file.isDirectory()) {
                    File files[] = file.listFiles();
                    for (File f : files) {
                        clearFile(f.getAbsolutePath(), deleteThisPath);
                    }
                }
                if (deleteThisPath) {
                    if (!file.isDirectory()) {
                        flag = file.delete();
                    } else {
                        if (file.listFiles().length == 0) {
                            flag = file.delete();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return flag;
    }

    /**
     * 返回表示Glide缓存大小的字符串，如"324.49KB"、"3.72MB"
     *
     * @param size 缓存文件夹的大小，单位为字节
     * @return 缓存大小，自适应单位KB/MB
     */
    private static String formatSize(double size) {
        size /= 1024; // from B to KB

        if (size >= 1024) {
            BigDecimal sizeMB = new BigDecimal(Double.toString(size / 1024));
            return sizeMB.setScale(2, BigDecimal.ROUND_HALF_UP).toPlainString() + "MB";
        } else {
            BigDecimal sizeKB = new BigDecimal(Double.toString(size));
            return sizeKB.setScale(2, BigDecimal.ROUND_HALF_UP).toPlainString() + "KB";
        }
    }
}
```

## 五、获取库图片路径

```java
//调用的Intent
Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
intent.setType("image/*");
intent.addCategory(Intent.CATEGORY_OPENABLE);
startActivityForResult(intent, FOR_RESULT_SUCCESS);
              
@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == FOR_RESULT_SUCCESS && data != null) {
        Uri uri = data.getData();
        String[] project = {MediaStore.Images.Media.DATA};
        CursorLoader loader = new CursorLoader(this, uri, project, null, null, null);
        Cursor cursor = loader.loadInBackground();
        int index = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);
        cursor.moveToFirst();
        String path = cursor.getString(index);
    }
}
```

## 六、ProgressBar 进度条

进度条布局代码

```xml
<ProgressBar
    android:id="@+id/pb_progressbar"
    style="@style/StyleProgressBarMini"
    android:layout_width="fill_parent"
    android:layout_height="wrap_content"
    android:layout_margin="30dp"
    android:background="@drawable/shape_progressbar_bg"
    android:max="1000"
    android:progress="768" />
```

@style/StyleProgressBarMini

```xml
<style name="StyleProgressBarMini" parent="@android:style/Widget.ProgressBar.Horizontal"> 
    <item name="android:maxHeight">50dp</item> 
    <item name="android:minHeight">10dp</item> 
    <item name="android:indeterminateOnly">false</item> 
    <item name="android:indeterminateDrawable">@android:drawable/progress_indeterminate_horizontal</item> 
    <item name="android:progressDrawable">@drawable/shape_progressbar_mini</item> 
</style>
```

@drawable/shape_progressbar_mini

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >
    <!-- 背景 -->
    <item android:id="@android:id/background">
        <shape>
            <corners android:radius="5dip" />
            <gradient
                android:angle="270"
                android:centerY="0.75"
                android:endColor="#FFFFFF"
                android:startColor="#FFFFFF" />
        </shape>
    </item>
    <item android:id="@android:id/secondaryProgress">
        <clip>
            <shape>
                <corners android:radius="0dip" />
 
                <gradient
                    android:angle="270"
                    android:centerY="0.75"
                    android:endColor="#df0024"
                    android:startColor="#df0024" />
            </shape>
        </clip>
    </item>
    <item android:id="@android:id/progress">
        <clip>
            <shape>
                <corners android:radius="5dip" />
                <gradient
                    android:angle="270"
                    android:centerY="0.75"
                    android:endColor="#de42ec"
                    android:startColor="#de42ec" />
            </shape>
        </clip>
    </item>
</layer-list>
```

@drawable/shape_progressbar_bg

```xml
<?xml version="1.0" encoding="UTF-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle" >
    <!-- 边框填充的颜色 -->
    <solid android:color="#cecece" />
    <!-- 设置进度条的四个角为弧形的半径 -->
    <corners android:radius="90dp" />
    <!-- padding：边界的间隔 -->
    <padding
        android:bottom="1dp"
        android:left="1dp"
        android:right="1dp"
        android:top="1dp" />
</shape>
```

## 七、supportInvalidateOptionsMenu() deprecated

```
'supportInvalidateOptionsMenu(): Unit' is deprecated. Overrides deprecated member in 'android.support.v4.app.FragmentActivity'. Deprecated in Java
:projectName:javaPreCompileDebug
```

__修复方法:__

使用以下方法替换`supportInvalidateOptionsMenu()`

```java
invalidateOptionsMenu()
```

## 八、避免使用android.media.ExifInterface

Android Studio提示使用`android.support.media.ExifInterface`替换`android.media.ExifInterface`，但是直接修改没法找到对应包，需要导入：`compile "com.android.support:exifinterface:27.0.2"`


