---
layout:     post
title:      "开启前台Service"
date:       2018-11-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android
---

## Service

将 __DaemonService__ 设置为前台服务，主要为了减小 __oom_adj__ 值(oom_adj越小优先级越高)，增加应用存活几率。在API为18或更高的版本，此方法会显示可见通知。为避免打扰用户，一般会启动另一个Service把出现的通知移除。最终达到通知栏没有常驻通知，但是oom_adj值减少的目的。

```java
class DaemonService : Service() {

    override fun onCreate() {
        super.onCreate()

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            val builder = Notification.Builder(this)
            builder.setSmallIcon(R.mipmap.ic_launcher)
            builder.setContentTitle(TAG)
            builder.setContentText(DESCRIPTION)
            startForeground(NOTIFICATION_ID, builder.build())
            startService(Intent(this, CancelNoticeService::class.java))
        } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
            startForeground(NOTIFICATION_ID, Notification())
        }
    }

    // 返回START_STICKY
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int) = START_STICKY

    override fun onDestroy() {
        super.onDestroy()

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            val manager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
            manager.cancel(NOTIFICATION_ID)
        }

        startService(Intent(applicationContext, DaemonService::class.java))
    }

    override fun onBind(intent: Intent?) = null

    companion object {
        const val TAG = "DaemonService"
        const val NOTIFICATION_ID = 1024
        const val NOTIFICATION_ID_STR = "DaemonService"
        const val DESCRIPTION = "DaemonService is running."
    }
}
```
在 __DaemonService__ 启动后会触发 __CancelNoticeService__ 的启动。 __CancelNoticeService__ 会在子线程中通过相同 __NotificationId__ 移除已经出现的通知栏，达到隐藏的效果。由于通知栏出现和隐藏的时间间隔很短，所以用户一般不会察觉。

```java
class CancelNoticeService : Service() {

    override fun onBind(intent: Intent?) = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.JELLY_BEAN_MR2) {
            val builder = Notification.Builder(this)
            builder.setSmallIcon(R.mipmap.ic_launcher)
            startForeground(DaemonService.NOTIFICATION_ID, builder.build())

            Thread(Runnable {
                stopForeground(true)
                val manager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
                manager.cancel(DaemonService.NOTIFICATION_ID)

                // 结束服务
                stopSelf()
            }).start()
        }

        return super.onStartCommand(intent, flags, startId)
    }
}
```

## 调用

在 __AndroidManifest.xml__ 注册以上两个Service。__DaemonService__ 要放在主进程，通过 __DaemonService__ 的前台可见属性提升主进程优先级。

```xml
<service
    android:name=".services.DaemonService"
    android:enabled="true"
    android:exported="true" />

<service
    android:name=".services.CancelNoticeService"
    android:enabled="true"
    android:exported="true" />
```

在 __MainActivity__ 启动 __DaemonService__

```java
val i = Intent(this, DaemonService::class.java)
ContextCompat.startForegroundService(this, i)
```

## 结果

根据实际测试，以上方法只能在 __Android 7.0__ 或以下版本使用。 __Android 7.1.1__ 真机和模拟器测试通知均会出现，可知该方法从 __Android 7.1.1__ 开始不起效。

以下结果在 __Android 5.0__ 上测试：

```shell
user@MacBook-Pro:/ # adb shell
shell@generic_x86:/ # su
root@generic_x86:/ # ps | grep playground
u0_a55    2919  1263  1283232 45972 SyS_epoll_ b7377fa5 S com.phantomvk.playground
```

应用进入后台，Service尚未启动

```shell
root@generic_x86:/ # cat /proc/2919/oom_adj
6
```

应用对用户可见，并启动Service。前台界面对用户可见就为0，不受Service影响。

```shell
root@generic_x86:/ # cat /proc/2919/oom_adj
0
```

应用进入后台，Activity所在进程 __oom_adj__ 从6变为1

```shell
root@generic_x86:/ # cat /proc/2919/oom_adj
1
```

经过一段时间测试，虽然主进程 __oom_adj__ 能保持1，但在设备可用内存低时，还是出现应用被回收的现象。不过，对比没有保护的应用，使用以上方法后被杀的时间点明显延后很多。
