---
layout:     post
title:      "setContentView源码解析"
date:       2019-02-15
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## Activity

__Window__ 是 __Activity__ 的数据成员

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient {
        
    private Window mWindow;
    
    .....
}
```

__PhoneWindow__ 是 __Window__ 的实现。变量为 __DecorView__ 类型的 __mDecor__，这是界面的根布局。

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback{
    .....

    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    .....
}
```


## AppCompatActivity

以下是 __AppCompatActivity__ 的类注释

```
/**
 * Base class for activities that use the
 * <a href="{@docRoot}tools/extras/support-library.html">support library</a> action bar features.
 *
 * <p>You can add an {@link android.support.v7.app.ActionBar} to your activity when running on API level 7 or higher
 * by extending this class for your activity and setting the activity theme to
 * {@link android.support.v7.appcompat.R.style#Theme_AppCompat Theme.AppCompat} or a similar theme.
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 *
 * <p>For information about how to use the action bar, including how to add action items, navigation
 * modes and more, read the <a href="{@docRoot}guide/topics/ui/actionbar.html">Action
 * Bar</a> API guide.</p>
 */
```

![AppCompatActivity](/img/android/Activity/AppCompatActivity.png)

__AppCompatActivity__ 重写 __Activity__ 的 __setContentView()__，把相关工作交给代理类完成，而不是 __Activity.setContentView()__。本文分析流程将沿着代理类的实现进行解释。

```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}
```

获取代理类 __AppCompatDelegate__

```java
@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}
```

## AppCompatDelegate

代理实现类没有看见 __AppCompatDelegateImplV9__

```java
private static AppCompatDelegate create(Context context, Window window,
        AppCompatCallback callback) {
    if (Build.VERSION.SDK_INT >= 24) {
        return new AppCompatDelegateImplN(context, window, callback);
    } else if (Build.VERSION.SDK_INT >= 23) {
        return new AppCompatDelegateImplV23(context, window, callback);
    } else {
        return new AppCompatDelegateImplV14(context, window, callback);
    }
}
```

__AppCompatDelegateImplV14__ 继承自 __AppCompatDelegateImplV9__，V23以下工作都交给它处理。

```java
@RequiresApi(14)
class AppCompatDelegateImplV14 extends AppCompatDelegateImplV9 {
    ....
}
```

代理实现的 __setContentView()__ 位于 __AppCompatDelegateImplV9__，其他子类没有重写这个方法。

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    // mSubDecor里获取名为android.R.id.content的ViewGroup
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    // 移除contentParent里所有视图
    contentParent.removeAllViews();
    // 传入的Activity resId在这里填充并加入到contentParent
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```

#### ensureSubDecor()

标记window的sub-decor布局是否已经装载的标志位，变量位于 __AppCompatDelegateImplV9__

```java
private boolean mSubDecorInstalled;
```

上述的 __setContentView()__ 调用 __ensureSubDecor()__ 。里面最重要的调用方法是 __createSubDecor()__ 

```java
private void ensureSubDecor() {
    // 先检查标志位，确实SubDecor是否已经装载
    if (!mSubDecorInstalled) {
        // 构建SubDecor
        mSubDecor = createSubDecor();

        // 如果在decor之前已经设置标题，则在decor装载完毕后设置这个标题
        CharSequence title = getTitle();
        if (!TextUtils.isEmpty(title)) {
            onTitleChanged(title);
        }

        applyFixedSizeWindow();

        onSubDecorInstalled(mSubDecor);

        mSubDecorInstalled = true;

        // Invalidate if the panel menu hasn't been created before this.
        // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
        // being called in the middle of onCreate or similar.
        // A pending invalidation will typically be resolved before the posted message
        // would run normally in order to satisfy instance state restoration.
        PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
        if (!isDestroyed() && (st == null || st.menu == null)) {
            invalidatePanelMenu(FEATURE_SUPPORT_ACTION_BAR);
        }
    }
}
```

#### createSubDecor()

```java
private ViewGroup createSubDecor() {
    TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

    if (!a.hasValue(R.styleable.AppCompatTheme_windowActionBar)) {
        a.recycle();
        // AppCompat需配合Theme.AppCompat使用，否则会抛出以下异常
        throw new IllegalStateException(
                "You need to use a Theme.AppCompat theme (or descendant) with this activity.");
    }

    // 从主体获取属性值，通过requestWindowFeature()把对应属性标为true
    if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
        requestWindowFeature(Window.FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
        requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
    }
    if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
        requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
    }
    if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
        requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
    }
    mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
    a.recycle();

    // 创建DecorView并装载到Window，实现类是PhoneWindow
    mWindow.getDecorView();

    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;


    if (!mWindowNoTitle) {
        // 配置标题相关代码，省略
    } else {
        // 配置标题相关代码，省略
    }

    if (subDecor == null) {
        throw new IllegalArgumentException(
                "AppCompat does not support the current theme features: { "
                        + "windowActionBar: " + mHasActionBar
                        + ", windowActionBarOverlay: "+ mOverlayActionBar
                        + ", android:windowIsFloating: " + mIsFloating
                        + ", windowActionModeOverlay: " + mOverlayActionMode
                        + ", windowNoTitle: " + mWindowNoTitle
                        + " }");
    }

    if (mDecorContentParent == null) {
        mTitleView = (TextView) subDecor.findViewById(R.id.title);
    }

    // Make the decor optionally fit system windows, like the window's decor
    ViewUtils.makeOptionalFitsSystemWindows(subDecor);

    final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
            R.id.action_bar_activity_content);

    // 从PhoneWindow中获取content布局对象
    final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    if (windowContentView != null) {
        // There might be Views already added to the Window's content view so we need to
        // migrate them to our content view
        while (windowContentView.getChildCount() > 0) {
            final View child = windowContentView.getChildAt(0);
            windowContentView.removeViewAt(0);
            contentView.addView(child);
        }

        // Useful for fragments.
        windowContentView.setId(View.NO_ID);
        // 设置contentView的id为android.R.id.content
        contentView.setId(android.R.id.content);

        // The decorContent may have a foreground drawable set (windowContentOverlay).
        // Remove this as we handle it ourselves
        if (windowContentView instanceof FrameLayout) {
            ((FrameLayout) windowContentView).setForeground(null);
        }
    }

    // Now set the Window's content view with the decor
    mWindow.setContentView(subDecor);

    contentView.setAttachListener(new ContentFrameLayout.OnAttachListener() {
        @Override
        public void onAttachedFromWindow() {}

        @Override
        public void onDetachedFromWindow() {
            dismissPopups();
        }
    });

    return subDecor;
}
```


如果什么样式都没有配置，则会选择 __R.layout.screen_simple__ 作为布局。可见里面id为 __content__ 的视图为 __FrameLayout__，其实就是我们填充 __Activity__ 的父布局。这个也是赋值给 __mContentParent__ 的实例。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />

    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```



```java
@Override
public boolean requestWindowFeature(int featureId) {
    featureId = sanitizeWindowFeatureId(featureId);

    if (mWindowNoTitle && featureId == FEATURE_SUPPORT_ACTION_BAR) {
        return false; // Ignore. No title dominates.
    }
    if (mHasActionBar && featureId == Window.FEATURE_NO_TITLE) {
        // Remove the action bar feature if we have no title. No title dominates.
        mHasActionBar = false;
    }

    switch (featureId) {
        case FEATURE_SUPPORT_ACTION_BAR:
            throwFeatureRequestIfSubDecorInstalled();
            mHasActionBar = true;
            return true;
        case FEATURE_SUPPORT_ACTION_BAR_OVERLAY:
            throwFeatureRequestIfSubDecorInstalled();
            mOverlayActionBar = true;
            return true;
        case FEATURE_ACTION_MODE_OVERLAY:
            throwFeatureRequestIfSubDecorInstalled();
            mOverlayActionMode = true;
            return true;
        case Window.FEATURE_PROGRESS:
            throwFeatureRequestIfSubDecorInstalled();
            mFeatureProgress = true;
            return true;
        case Window.FEATURE_INDETERMINATE_PROGRESS:
            throwFeatureRequestIfSubDecorInstalled();
            mFeatureIndeterminateProgress = true;
            return true;
        case Window.FEATURE_NO_TITLE:
            throwFeatureRequestIfSubDecorInstalled();
            mWindowNoTitle = true;
            return true;
    }

    return mWindow.requestFeature(featureId);
}
```

## PhoneWindow

__PhoneWindow__ 的父类 __Window__。

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {
    .....
}
```

#### getDecorView()

检查是否已经创建 __mDecor__ 

```java
@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```

#### installDecor()

如果 __mDecor__ 没有创建，则必须先创建 __DecorVIew__ 并赋值

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 返回DecorView实例并赋值给mDecor，具体看下文
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        // DecorView保存了PhoneWindow的实例
        mDecor.setWindow(this);
    }

    // Decor已经装载完毕，开始初始化mContentParent
    if (mContentParent == null) {
        // 返回布局内id名为content的布局并复制到mContentParent
        mContentParent = generateLayout(mDecor);

        .....
        .....
    }
}
```

#### generateDecor(featureId)

创建 __DecorView__ 实例

```java
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```

#### generateLayout(DecorView decor)

__DecorView__ 创建完成后就能给 __DecorView__ 添加布局。本方法根据窗口的风格样式，选择窗口对应的资源根布局文件。作为mDecor的根布局添加到mDecor中，然后从这个布局里面，获取id名为 __content__ 的 __FrameLayout__ 赋值给 __mContentParent__ 变量。对比 __mContentRoot__，__mContentRoot__ 其实就是 __mContentParent__ 所在的父布局。

```java
protected ViewGroup generateLayout(DecorView decor) protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

    TypedArray a = getWindowStyle();

    // 读取xml样式里相关配置信息
    mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
    int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
            & (~getForcedWindowFlags());
    if (mIsFloating) {
        setLayout(WRAP_CONTENT, WRAP_CONTENT);
        setFlags(0, flagsToUpdate);
    } else {
        setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
    }

    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
        // Don't allow an action bar if there is no title.
        requestFeature(FEATURE_ACTION_BAR);
    }

    // 这里省略类似上面这种配置处理的代码
    .....
    .....

    if (params.windowAnimations == 0) {
        params.windowAnimations = a.getResourceId(
                R.styleable.Window_windowAnimationStyle, 0);
    }

    // The rest are only done if this window is not embedded; otherwise,
    // the values are inherited from our container.
    if (getContainer() == null) {
        if (mBackgroundDrawable == null) {
            if (mBackgroundResource == 0) {
                mBackgroundResource = a.getResourceId(
                        R.styleable.Window_windowBackground, 0);
            }
            if (mFrameResource == 0) {
                mFrameResource = a.getResourceId(R.styleable.Window_windowFrame, 0);
            }
            mBackgroundFallbackResource = a.getResourceId(
                    R.styleable.Window_windowBackgroundFallback, 0);
        }
        if (mLoadElevation) {
            mElevation = a.getDimension(R.styleable.Window_windowElevation, 0);
        }
        mClipToOutline = a.getBoolean(R.styleable.Window_windowClipToOutline, false);
        mTextColor = a.getColor(R.styleable.Window_textColor, Color.TRANSPARENT);
    }

    // 填充window decor.

    // 通过设置的features决定layoutResource的id
    int layoutResource;
    // 除了xml样式，通过代码也能完成相关配置，这里读取代码实现的配置
    int features = getLocalFeatures();

    // 特性读取完成后，根据设置选择对应layoutResource
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        ..... // 省略类似的条件判断选择
        .....
    }

    mDecor.startChanging();
    // 用layoutResource构建布局
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
    
    .....
    .....

    mDecor.finishChanging();

    // 返回contentParent
    return contentParent;
}
```

__installDecor()__ 中调用 __generateDecor()__ 和 __generateLayout__，完成构建 __DecorView__ 实例，并把其子视图内名为 __content__ 的视图绑定到变量 __mContentParent__ 。

#### onResourcesLoaded

这个方法负责把 __layoutResource__ 构建为视图，加入到 __DecorView__

```java
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    if (mBackdropFrameRenderer != null) {
        loadBackgroundDrawablesIfNeeded();
        mBackdropFrameRenderer.onResourcesLoaded(
                this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
                mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
                getCurrentColor(mNavigationColorViewState));
    }

    mDecorCaptionView = createDecorCaptionView(inflater);
    // 样式决定layoutResource，填充对应视图
    final View root = inflater.inflate(layoutResource, null);
    if (mDecorCaptionView != null) {
        if (mDecorCaptionView.getParent() == null) {
            // mDecorCaptionView加入到DecorView
            addView(mDecorCaptionView,
                    new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        // root加到mDecorCaptionView，所以root是DecorView的子视图，布局参数为MATCH_PARENT
        mDecorCaptionView.addView(root,
                new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
    } else {
        // 填充layoutResource获得的视图添加到DecorView
        // Put it below the color views.
        addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }
    // 赋值给mContentRoot
    mContentRoot = (ViewGroup) root;
    initializeElevation();
}
```

前文提到，Activity主题的样式决定变量 __layoutResource__ 所指明的布局。一般通过以下形式指定样式

```xml
android:theme="@style/Theme.AppCompat.Light.NoActionBar"
```

此外，也常用以下代码配置相关参数，由 __getLocalFeature()__ 负责处理

```java
requestWindowFeature(Window.FEATURE_NO_TITLE);
getWindow().setBackgroundDrawable(null);
getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

## 填充布局

经过上面一圈的论述，回到原来关注的调用点，上面的所有操作都发生在 __ensureSubDecor()__ 内。随后的工作就是实例化界面的视图并加入到 __contentParent__。

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    // mSubDecor里获取名为android.R.id.content的ViewGroup
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    // 移除contentParent里所有已有的视图
    contentParent.removeAllViews();
    // 传入的Activity resId在这里填充并加入到contentParent
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```

完成所有工作后调用 __mOriginalWindowCallback.onContentChanged();__ 发出通知

## 参考链接

- [Android应用setContentView与LayoutInflater加载解析机制源码分析](https://blog.csdn.net/yanbober/article/details/45970721)