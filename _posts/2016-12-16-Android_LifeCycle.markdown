---
layout:     post
title:      "重现onPause到onResume生命周期"
date:       2016-12-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

最近重看《Android开发艺术探索》一书，其中第3页`Activity`生命周期`onPause`到`onResume`过程的确如作者所说“在一般的开发中用不上”，但是作为开发者还是有研究的必要。

`onResume`的状态是`Activity`前台可见正在活动，`onPause`是置于前台可见停止活动。从后者到前者的变化场景，可以通过一个透明的`Dialog`弹出遮蔽`MainActivity`重现。

不过，这是个“特殊”的`Dialog`。一般`Dialog`弹出时，背景可见`MainActivity`依然是`onResume`，表明这个`Activity`的生命周期并没有因为这个`Dialog`的弹出而变化。

但是，我们使用`Activity`实现一个`Dialog`时，由于是`MainActivity`启动`DialogActivity`，所以`MainActivity`生命周期必然会变化。


__MainActivity.java代码__


```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 图方便直接给findViewById加监听
        findViewById(R.id.btn_dialog).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent i = new Intent(MainActivity.this, DialogActivity.class);
                startActivity(i);
            }
        });
        System.out.println("onCreate");
    }
    // 实现所有生命周期方法Log
    @Override
    protected void onStart() {
        super.onStart();
        System.out.println("onStart");
    }

    @Override
    protected void onRestart() {
        super.onRestart();
        System.out.println("onRestart");
    }

    @Override
    protected void onResume() {
        super.onResume();
        System.out.println("onResume");
    }

    @Override
    protected void onPause() {
        super.onPause();
        System.out.println("onPause");
    }

    @Override
    protected void onStop() {
        super.onStop();
        System.out.println("onStop");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        System.out.println("onDestroy");
    }
}
```

__MainActivity.java对应activity_main.xml__

`Button`只是用来启动`Dialog_Activity`

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/btn_dialog"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Button" />
</RelativeLayout>
```

__DialogActivity.java代码__

```java
public class DialogActivity extends Activity {
    
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.test_dialog);
        // 这里图个方便直接给findViewById加监听
        findViewById(R.id.returnButton).setOnClickListener(new View.OnClickListener() {
            public void onClick(View v) {
                DialogActivity.this.finish();
            }
        });
    }
}
```

__test_dialog.xml__


```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/returnButton"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:text="点击退出"/>

</LinearLayout>
```


__AndroidManifest.xml__

注意!!! `DialogActivity`使用的主题是`@android:style/Theme.Dialog`，不然上面的代码就白瞎了

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.phantomvk.exampleapp">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity android:name=".DialogActivity"
            android:theme="@android:style/Theme.Dialog"/>
    </application>

</manifest>
```

下面是结果

```java
// MainActivity到可见可操作的三个周期变化
12-16 23:59:35.806 7477-7477/com.phantomvk.exampleapp I/System.out: onCreate
12-16 23:59:35.811 7477-7477/com.phantomvk.exampleapp I/System.out: onStart
12-16 23:59:35.811 7477-7477/com.phantomvk.exampleapp I/System.out: onResume

// 点击Button后，MainActivity仅仅是暂停(onPause)，没有去停止(onStop)
12-16 23:59:50.731 7477-7477/com.phantomvk.exampleapp I/System.out: onPause

// 在DialogActivity里面退出，MainActivity状态变为onResume
12-17 00:00:00.161 7477-7477/com.phantomvk.exampleapp I/System.out: onResume

// 退出应用: onPause -> onStop -> onDestroy
12-17 00:00:30.576 7477-7477/com.phantomvk.exampleapp I/System.out: onPause
12-17 00:00:31.151 7477-7477/com.phantomvk.exampleapp I/System.out: onStop
12-17 00:00:31.151 7477-7477/com.phantomvk.exampleapp I/System.out: onDestroy
```

上面的代码已经成功重现`MainActivity`里`onPause` -> `onResume`。

运行截图如下

![img](/img/android/onPause_onResume.png)

提醒一点，直接使用Dialog组件的MainActivity生命周期是不会变化的，这样应用更加高效且性能更好。这篇文章只是用来重现题目所说的目标，不是用于实践。


