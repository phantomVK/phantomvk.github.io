---
layout:     post
title:      "Fragment打开页面"
date:       2019-04-02
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---

在 __Fragment__ 内部调用自有方法 __startActivityForResult()__

```java
public void startActivityForResult(Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

该方法辗转调用同名重载方法，方法内调用名为 __mHost__ 变量的方法，该变量的类型为 __FragmentHostCallback__。

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    // 如果Fragment没有绑定到Activity，即宿主实例不存在，会抛出异常
    if (mHost == null) {
        throw new IllegalStateException("Fragment " + this + " not attached to Activity");
    }
    // 调用方法
    mHost.onStartActivityFromFragment(this, intent, requestCode, options);
}
```

__FragmentHostCallback__ 是抽象类，继承抽象父类 __FragmentContainer__，由 __Fragment__ 所依附的宿主实现。


```java
FragmentHostCallback mHost;
```

实现该抽象类表示具体实现类具有保存和展示Fragment的能力。

```java
public abstract class FragmentHostCallback<E> extends FragmentContainer {
    public void onStartActivityFromFragment(
            Fragment fragment, Intent intent, int requestCode, @Nullable Bundle options) {
        if (requestCode != -1) {
            throw new IllegalStateException(
                    "Starting activity with a requestCode requires a FragmentActivity host");
        }
        mContext.startActivity(intent);
    }
}
```

__FragmentHostCallback__ 的实际实现类是 __FragmentActivity__ 的内部类 __HostCallbacks__。实现抽象类的同时还复写了父类的实现逻辑，当 __Fragment__ 调用该方法时，实际调用 __FragmentActivity__ 实现的成员方法。

```java
public class FragmentActivity extends BaseFragmentActivityApi16 implements
        ViewModelStoreOwner,
        ActivityCompat.OnRequestPermissionsResultCallback,
        ActivityCompat.RequestPermissionsRequestCodeValidator {
            
    .....
        
    // 面试考点：非静态内部类隐式持有外部类的实例引用
    class HostCallbacks extends FragmentHostCallback<FragmentActivity> {
        .....

        @Override
        public void onStartActivityFromFragment(
                Fragment fragment, Intent intent, int requestCode, @Nullable Bundle options) {
            // 内部类调用了宿主的成员方法，相当于做了一层桥接
            FragmentActivity.this.startActivityFromFragment(fragment, intent, requestCode, options);
        }
    }
}
```

看宿主 __FragmentActivity__ 的方法实现：方法内实参 __requestCode__ 高16位保存 __(requestIndex+1)__ 的值，低16位保存来自 __Fragment__ 的 __requestCode__ 值。

```java
public void startActivityFromFragment(Fragment fragment, Intent intent,
        int requestCode, @Nullable Bundle options) {
    mStartedActivityFromFragment = true;
    try {
        if (requestCode == -1) {
            ActivityCompat.startActivityForResult(this, intent, -1, options);
            return;
        }
        checkForValidRequestCode(requestCode);
        // Activity给Fragment的请求生成requestIndex，用于后续匹配返回的Fragment
        int requestIndex = allocateRequestIndex(fragment);
        // requestIndex放在requestCode实参高16位，Fragment提供的requestCode放在实参低16位
        ActivityCompat.startActivityForResult(
                this, intent, ((requestIndex + 1) << 16) + (requestCode & 0xffff), options);
    } finally {
        mStartedActivityFromFragment = false;
    }
}
```

当结果从其他 __Activity__ 回到 __Fragment__ 所在宿主 __Activity__ 时，宿主页面先检查 __requestIndex__ 的值，判断原始请求是否来自 __Fragment__。

__requestCode__ 高16位非零表示该请求来自 __Fragment__，所以请求要转发给源 __Fragment__。

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    mFragments.noteStateNotSaved();
    // 取出requestCode高16位
    int requestIndex = requestCode>>16;

    // requestIndex非0表示请求来自Fragment
    if (requestIndex != 0) {
        requestIndex--;

        // 用requestIndex找对应的请求Fragment名称
        String who = mPendingFragmentActivityResults.get(requestIndex);
        // 请求完成转发，可以移除该requestIndex的记录
        mPendingFragmentActivityResults.remove(requestIndex);
        if (who == null) {
            Log.w(TAG, "Activity result delivered for unknown Fragment.");
            return;
        }
        // 从Activity的Fragment栈查找接收数据的目标实例
        Fragment targetFragment = mFragments.findFragmentByWho(who);
        // 如果此时Fragment已被销毁，返回的结果直接丢弃
        if (targetFragment == null) {
            Log.w(TAG, "Activity result no fragment exists for who: " + who);
        } else {
            // 进行与操作后就是请求的原始requestCode，即低16位数据
            // 作为参数和返回结果调用Fragment.onActivityResult()
            targetFragment.onActivityResult(requestCode & 0xffff, resultCode, data);
        }
        return;
    }

    ActivityCompat.PermissionCompatDelegate delegate =
            ActivityCompat.getPermissionCompatDelegate();
    if (delegate != null && delegate.onActivityResult(this, requestCode, resultCode, data)) {
        // Delegate has handled the activity result
        return;
    }
  
    // 请求交给Activity执行
    super.onActivityResult(requestCode, resultCode, data);
}
```

由于上述调用位于 __FragmentActivity__ 中，如果重写该方法时没调用 __super.onActivityResult(int requestCode, int resultCode, Intent data)__，页面返回结果不会得到正确处理，对应 __Fragment__ 也不会收到该结果通知。

可知，__Activity__ 自己调用 __onActivityResult__ 方法时传入的 __requestCode__ 不能大于 __0xFFFF__，否则会和来自 __Fragment__ 的请求 __requestCode__ 混淆。