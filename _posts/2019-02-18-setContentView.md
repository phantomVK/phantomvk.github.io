---
layout:     post
title:      "Android源码系列(20) -- setContentView"
date:       2019-02-18
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、Activity

__mWindow__ 是 __Activity__ 的数据成员，源码来自 __Android 27.1.1__

```java
public class Activity extends ContextThemeWrappers
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient {
        
    private Window mWindow;

    .....
}
```

__PhoneWindow__ 是 __Window__ 的具体实现。__mDecor__ 为 __DecorView__ 类型的成员 ，是界面的根布局。

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback{
    .....

    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    .....
}
```

## 二、AppCompatActivity

__AppCompatActivity__ 继承关系：

![AppCompatActivity](/img/android/Activity/AppCompatActivity.png)

__AppCompatActivity__ 重写 __Activity.setContentView()__ 方法，把相关工作交给代理类完成。本文分析流程将沿着代理类的实现进行解释。

```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}
```

通过单例模式获取代理类 __AppCompatDelegate__ 实例。因为获取操作一定在主线程执行，所以不需要增加线程保护等操作。

```java
@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}
```

## 三、AppCompatDelegate

在代理实现类中没有看见 __AppCompatDelegateImplV9__ 的分支

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

__AppCompatDelegateImplV14__ 继承自 __AppCompatDelegateImplV9__，可知v23以下工作都交给v14处理。

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
    // 先初始化SubDecor
    ensureSubDecor();
    // mSubDecor里获取名为android.R.id.content的ViewGroup
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    // 移除contentParent里所有视图
    contentParent.removeAllViews();
    // 传入的resId在这里填充并添加到contentParent
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    // 通知Window
    mOriginalWindowCallback.onContentChanged();
}
```

#### 3.1 ensureSubDecor()

标记 __Window__ 的 __subDecor__ 布局是否已经装载标志位，变量位于 __AppCompatDelegateImplV9__

```java
private boolean mSubDecorInstalled;
```

上述 __setContentView()__ 调用 __ensureSubDecor()__，里面最重要的调用方法是 __createSubDecor()__ 。如果多次调用 __setContentView(int resId)__ 方法，则后续 __mSubDecorInstalled__ 标志位为 __true__ 而不初始化 __SubDecor__。

```java
private void ensureSubDecor() {
    // 先检查标志位，确定SubDecor是否已装载
    if (!mSubDecorInstalled) {
        // 构建SubDecor
        mSubDecor = createSubDecor();

        // 如果在decor之前已经配置标题，则在decor装载完毕后使用这个标题
        CharSequence title = getTitle();
        if (!TextUtils.isEmpty(title)) {
            onTitleChanged(title);
        }

        applyFixedSizeWindow();

        // 空实现，里面没有逻辑
        onSubDecorInstalled(mSubDecor);

        // 标记SubDecor已装载
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

#### 3.2 createSubDecor()

上述 __mSubDecorInstalled__ 为空则创建 __SubDecor__

```java
private ViewGroup createSubDecor() {
    TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

    if (!a.hasValue(R.styleable.AppCompatTheme_windowActionBar)) {
        a.recycle();
        // AppCompat需配合Theme.AppCompat使用，否则会抛出以下异常
        throw new IllegalStateException(
                "You need to use a Theme.AppCompat theme (or descendant) with this activity.");
    }

    // 从主题获取样式，并通过requestWindowFeature()把对应属性标为true
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


    // 由主题配置决定使用的布局，填充视图赋值给subDecor
    if (!mWindowNoTitle) {
        if (mIsFloating) {
                // 类似这种根据样式选择布局，并初始化subDecor
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);

                // 悬浮windows没有action bar，重置该标志位
                mHasActionBar = mOverlayActionBar = false;
        } else if (mHasActionBar) {
            .....
        }
    } else {
        .....
    }

    // 上面决策完毕后subDecor不为空
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
        // 把PhoneWindow的视图放入subDecor的contentView
        while (windowContentView.getChildCount() > 0) {
            final View child = windowContentView.getChildAt(0);
            windowContentView.removeViewAt(0);
            contentView.addView(child);
        }

        // 把PhoneWindow名为android.R.id.content视图的id去掉
        windowContentView.setId(View.NO_ID);
        // 设置contentView的id为android.R.id.content
        // 相当于PhoneWindow把同名id让给subDecor的子视图使用
        contentView.setId(android.R.id.content);

        // The decorContent may have a foreground drawable set (windowContentOverlay).
        // Remove this as we handle it ourselves
        if (windowContentView instanceof FrameLayout) {
            ((FrameLayout) windowContentView).setForeground(null);
        }
    }

    // 还要subDecor加到PhoneWindow作为子视图
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


如果什么样式都没有配置，__subDecor__ 会默认选择 __R.layout.screen_simple__ 作为布局。从以下xml布局可见里面id为 __content__ 的视图为 __FrameLayout__，里面保存着我们填充的 __Activity__ 布局。这个也是赋值给 __mContentParent__ 的视图。

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

#### 3.3 requestWindowFeature()

从主题获取样式 __featureId__，由此id决定特性是否开启

```java
// true if this activity has an action bar.
boolean mHasActionBar;
// true if this activity's action bar overlays other activity content.
boolean mOverlayActionBar;
// true if this any action modes should overlay the activity content
boolean mOverlayActionMode;
// true if this activity is floating (e.g. Dialog)
boolean mIsFloating;
// true if this activity has no title
boolean mWindowNoTitle;

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

## 四、PhoneWindow

__Window__ 是 __PhoneWindow__ 的父类，定义一些列绘制窗口的抽象方法，作为顶级视图添加到 __window manager__。

```java
// Abstract base class for a top-level window look and behavior policy.  An
// instance of this class should be used as the top-level view added to the
// window manager. It provides standard UI policies such as a background, title
// area, default key processing, etc.
//
// <p>The only existing implementation of this abstract class is
// android.view.PhoneWindow, which you should instantiate when needing a
// Window.
public abstract class Window {
    .....
}
```

#### 4.1 getDecorView()

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

#### 4.2 installDecor()

如果 __mDecor__ 没有创建，则必须先创建 __DecorView__ 并赋值

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
        // 返回布局内id名为content的布局并赋值到mContentParent
        mContentParent = generateLayout(mDecor);

        .....
        .....
    }
}
```

#### 4.3 generateDecor(featureId)

创建 __DecorView__ 实例

```java
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    // 处理Context和主题相关逻辑，省略
    if (mUseDecorContext) {
        .....
    }
    // 创建新DecorView
    return new DecorView(context, featureId, this, getAttributes());
}
```

#### 4.4 generateLayout(DecorView decor)

__DecorView__ 创建完成赋值给 __mDecor__。本方法根据窗口的风格样式，选择窗口对应的资源根布局文件，作为 __mDecor__ 的子布局进行添加。

然后从这个布局里面，获取id名为 __content__ 的 __FrameLayout__ 赋值给 __mContentParent__ 变量。对比 __mContentRoot__，__mContentRoot__ 其实就是 __mContentParent__ 所在的父布局。

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
    // 用layoutResource构建布局并加入到mDecor
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    // 从mDecor内查找id为content的子视图
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

#### 4.5 onResourcesLoaded

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

前文提到，Activity主题样式决定 __layoutResource__ 所指布局，一般通过 __xml__ 指定样式：

```xml
android:theme="@style/Theme.AppCompat.Light.NoActionBar"
```

__requestWindowFeature()__ 配置相关参数，由 __getLocalFeature()__ 负责处理

```java
requestWindowFeature(Window.FEATURE_NO_TITLE);
```

## 五、填充布局

经过上面 __DecorView__ 创建和初始化的论述，现在回到原来关注的调用点上。随后填充视图并加入到 __contentParent__，这个视图就是日常使用的 __setContentView(int resId)__ 的 __resId__

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

工作完成，回调 __onContentChanged()__ 发出通知，而 __Activity__ 基类实现为空逻辑。

```java
public void onContentChanged() {
}
```

## 六、总结

所有工作完成后，界面布局层次如下：

![Window](/img/android/Activity/Window.png)

__setContentView(int resId)__ 工作流程：

- __Activity__ 的 __mWindow__ 已提前初始化为 __PhoneWindow__ 类型实例；
- __Activity__ 把 __PhoneWindow__ 的配置工作交给代理进行；
- 代理给 __PhoneWindow__ 的 __mDecor__ 创建 __DecorView__ 实例；
- 并根据 __Activity__ 主题样式选择 __layoutResource__，填充后赋值给 __mContentRoot__；
- 把 __mContentRoot__ 加到 __PhoneWindow__ 的 __mDecor__ 作为子视图；
- 在 __mDecor__ 找一个 __id__ 为 __android.R.id.content__ 的视图赋值给  __contentParent__；
- 最后，根据开发者定义的布局 __resId__，实例化后加到 __contentParent__ 内.

## 七、参考链接

- [Android应用setContentView与LayoutInflater加载解析机制源码分析](https://blog.csdn.net/yanbober/article/details/45970721)

- [Android Window 机制探索](https://juejin.im/entry/5a123c31f265da430d579cda)

