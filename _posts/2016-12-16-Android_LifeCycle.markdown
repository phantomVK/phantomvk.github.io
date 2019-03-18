---
layout:     post
title:      "透明Activity生命周期变化"
date:       2016-12-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Android
---

最近重看《Android开发艺术探索》一书，其中第3页 __Activity__ 生命周期 __onPause__ 到 __onResume__ 过程的确如作者所说“在一般的开发中用不上”，但是作为开发者还是有研究的必要。

__onResume__ 的状态是 __Activity__ 前台可见正在活动，__onPause__ 是置于前台可见停止活动。从后者到前者的变化场景，可以通过一个透明的 __Dialog__ 弹出遮蔽 __MainActivity__ 重现。

不过，这是个“特殊”的 __Dialog__。一般 __Dialog__ 弹出时，背景可见 __MainActivity__ 依然是 __onResume__，表明这个 __Activity__ 的生命周期并没有因为这个 __Dialog__ 的弹出而变化。

但是，我们使用 __Activity__ 实现 __Dialog__ 时，由于是 __MainActivity__ 启动 __DialogActivity__，所以 __MainActivity__ 生命周期必然会变化。

__MainActivity.java代码__

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

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

__Button__ 只是用来启动 __Dialog_Activity__

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

注意： __DialogActivity__ 使用主题 __@android:style/Theme.Dialog__，不然无法实现上述生命周期。

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

结果：

```java
// MainActivity到可见可操作的三个周期变化
12-16 23:59:35.806 7477-7477/com.phantomvk.exampleapp I/System.out: onCreate
12-16 23:59:35.811 7477-7477/com.phantomvk.exampleapp I/System.out: onStart
12-16 23:59:35.811 7477-7477/com.phantomvk.exampleapp I/System.out: onResume

// 点击Button后，MainActivity仅仅是暂停(onPause)，没有停止(onStop)
12-16 23:59:50.731 7477-7477/com.phantomvk.exampleapp I/System.out: onPause

// 在DialogActivity里面退出，MainActivity状态变为onResume
12-17 00:00:00.161 7477-7477/com.phantomvk.exampleapp I/System.out: onResume

// 退出应用: onPause -> onStop -> onDestroy
12-17 00:00:30.576 7477-7477/com.phantomvk.exampleapp I/System.out: onPause
12-17 00:00:31.151 7477-7477/com.phantomvk.exampleapp I/System.out: onStop
12-17 00:00:31.151 7477-7477/com.phantomvk.exampleapp I/System.out: onDestroy
```

上述代码已经成功重现 onPause -> onResume。运行截图如下

![img](/img/android/images/onPause_onResume.png)
