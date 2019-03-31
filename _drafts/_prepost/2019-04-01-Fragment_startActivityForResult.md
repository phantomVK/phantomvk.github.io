---
layout:     post
title:      "Fragment.startActivityForResult"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

在 __Fragment__ 内部调用自有方法 __startActivityForResult__

```java
public void startActivityForResult(Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

该方法辗转调用同名重载方法，方法内调用名为 __mHost__ 变量的方法，类型为 __FragmentHostCallback__。

```java
// fragment所依附的宿主
FragmentHostCallback mHost;

public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    // 如果Fragment没有绑定到Activity，就会排除异常
    if (mHost == null) {
        throw new IllegalStateException("Fragment " + this + " not attached to Activity");
    }
    // 调用方法
    mHost.onStartActivityFromFragment(this /*fragment*/, intent, requestCode, options);
}
```

__FragmentHostCallback__ 是一个抽象类，继承抽象类 __FragmentContainer__。实现该抽象类表示具体实现类具有保存和展示Fragment的能力。

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

__FragmentHostCallback__ 的实际实现类，是 __FragmentActivity__ 的内部类 __HostCallbacks__。实现抽象类的同时还重写了父类的实现逻辑，当 __Fragment__ 调用该方法时，实际是调用了__FragmentActivity__ 实现的成员方法。

```java
public class FragmentActivity extends BaseFragmentActivityApi16 implements
        ViewModelStoreOwner,
        ActivityCompat.OnRequestPermissionsResultCallback,
        ActivityCompat.RequestPermissionsRequestCodeValidator {
            
    .....

    class HostCallbacks extends FragmentHostCallback<FragmentActivity> {
        .....

        @Override
        public void onStartActivityFromFragment(
                Fragment fragment, Intent intent, int requestCode, @Nullable Bundle options) {
            // 内部类又调用了宿主的成员方法，相当于做了一层桥接
            FragmentActivity.this.startActivityFromFragment(fragment, intent, requestCode, options);
        }
    }
}
```

那看看宿主 __FragmentActivity__ 的方法实现。方法内的实参 __requestCode__ 高16位保存 __(requestIndex+1)__ 的值，低16位保存来自 __Fragment__ 的 __requestCode__ 值。

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
        // Activity给Fragment的请求生成请求索引值，用于在后续匹配返回时的Fragment
        int requestIndex = allocateRequestIndex(fragment);
        // 请求索引放在requestCode实参高16位，Fragment提供的requestCode放在实参低16位
        ActivityCompat.startActivityForResult(
                this, intent, ((requestIndex + 1) << 16) + (requestCode & 0xffff), options);
    } finally {
        mStartedActivityFromFragment = false;
    }
}
```

当结果从其他 __Activity__ 回到 __Fragment__ 所在宿主 __Activity__ 时，宿主页面先检查 __requestIndex__ 的值，判断原始请求是否来自 __Fragment__。

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    mFragments.noteStateNotSaved();
    // 取出requestCode高16位
    int requestIndex = requestCode>>16;
    if (requestIndex != 0) {
        requestIndex--;

        // 用requestCode查找对应的请求Fragment
        String who = mPendingFragmentActivityResults.get(requestIndex);
        mPendingFragmentActivityResults.remove(requestIndex);
        if (who == null) {
            Log.w(TAG, "Activity result delivered for unknown Fragment.");
            return;
        }
        // 从Activity的Fragment栈中查找返回数据的最终归宿，即Fragment实例
        Fragment targetFragment = mFragments.findFragmentByWho(who);
        // Fragment如果已经被销毁则可能为空
        if (targetFragment == null) {
            Log.w(TAG, "Activity result no fragment exists for who: " + who);
        } else {
            // 进行与操作之后就是请求中原始的requestCode
            // 作为参数和返回结果让Fragment自行处理逻辑
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

    super.onActivityResult(requestCode, resultCode, data);
}
```

由于上述调用位于 __FragmentActivity__ 中，如果重写该方法的时候没有调用 __super.onActivityResult(int requestCode, int resultCode, Intent data)__，则页面内的 __Fragment__ 无法接受到该结果通知。

同时可知，__Activity__ 自己调用 __onActivityResult__ 方法时传入的 __requestCode__ 不可能大于 __0xffff__ ，否则就会和来自 __Fragment__ 的请求混淆。