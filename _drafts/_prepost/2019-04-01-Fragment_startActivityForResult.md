---
layout:     post
title:      "Fragment startActivityForResult"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

__Fragment__

在 __Fragment__ 内部调用自有方法 __startActivityForResult__

```java
public void startActivityForResult(Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

该方法辗转调用到同名的重载方法

```java
// Host this fragment is attached to.
FragmentHostCallback mHost;

public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mHost == null) {
        throw new IllegalStateException("Fragment " + this + " not attached to Activity");
    }
    mHost.onStartActivityFromFragment(this /*fragment*/, intent, requestCode, options);
}
```

__FragmentHostCallback__ 是一个抽象类，继承抽象类 __FragmentContainer__。如果具体类实现抽象类 __FragmentContainer__，则表示这个具体类具有保存和展示Fragment的能力。

```java
public abstract class FragmentHostCallback<E> extends FragmentContainer {
   /**
     * Starts a new {@link Activity} from the given fragment.
     * See {@link FragmentActivity#startActivityForResult(Intent, int, Bundle)}.
     */
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

__FragmentHostCallback__ 的实际实现类，是 __FragmentActivity__ 的内部类 __HostCallbacks__

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

__FragmentActivity__

```java
/**
 * Called by Fragment.startActivityForResult() to implement its behavior.
 */
public void startActivityFromFragment(Fragment fragment, Intent intent,
        int requestCode, @Nullable Bundle options) {
    mStartedActivityFromFragment = true;
    try {
        if (requestCode == -1) {
            ActivityCompat.startActivityForResult(this, intent, -1, options);
            return;
        }
        checkForValidRequestCode(requestCode);
        int requestIndex = allocateRequestIndex(fragment);
        ActivityCompat.startActivityForResult(
                this, intent, ((requestIndex + 1) << 16) + (requestCode & 0xffff), options);
    } finally {
        mStartedActivityFromFragment = false;
    }
}
```

当结果从其他Activity回到Fragment所在宿主的Activity时，宿主页面先检查requestIndex的值，判断原始请求是否来自Fragment

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    mFragments.noteStateNotSaved();
    int requestIndex = requestCode>>16;
    if (requestIndex != 0) {
        requestIndex--;

        String who = mPendingFragmentActivityResults.get(requestIndex);
        mPendingFragmentActivityResults.remove(requestIndex);
        if (who == null) {
            Log.w(TAG, "Activity result delivered for unknown Fragment.");
            return;
        }
        // 从自己的Fragment栈中查找返回数据的最终归宿
        Fragment targetFragment = mFragments.findFragmentByWho(who);
        // Fragment如果已经被销毁则可能为空
        if (targetFragment == null) {
            Log.w(TAG, "Activity result no fragment exists for who: " + who);
        } else {
            // 进行与操作之后就是原始请求的requestCode，让Fragment自行处理逻辑
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

由于上述调用位于 __FragmentActivity__ 中，如果重写该方法的时候没有调用 super.onActivityResult(int requestCode, int resultCode, Intent data)，则页面内的Fragment无法接受到该通知