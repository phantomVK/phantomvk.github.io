---
layout:     post
title:      "TakeScreenshotService"
date:       2019-01-08
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

Android28

## SaveImageInBackgroundData

```java
class SaveImageInBackgroundData {
    Context context;
    // 需要保存的截屏位图
    Bitmap image;
    // 图片保存的路径
    Uri imageUri;
    // 保存完成后调用finisher
    Runnable finisher;
    // 图标尺寸
    int iconSize;
    // 预览图宽度
    int previewWidth;
    // 预览图高度
    int previewheight;
    // 错误信息id
    int errorMsgResId;

    // 清空对象数据
    void clearImage() {
        image = null;
        imageUri = null;
        iconSize = 0;
    }
    
    // 移除保存的Context
    void clearContext() {
        context = null;
    }
}
```

## SaveImageInBackgroundTask

截图的 [AsyncTask](/2018/10/21/AsyncTask/) 任务，在后台把截取的图片保存到媒体存储

```java
class SaveImageInBackgroundTask extends AsyncTask<Void, Void, Void>
```

#### 2.1 静态变量

```java
private static final String TAG = "SaveImageInBackgroundTask";

// 截图保存的文件夹名称
private static final String SCREENSHOTS_DIR_NAME = "Screenshots";

// 截图文件名模板
private static final String SCREENSHOT_FILE_NAME_TEMPLATE = "Screenshot_%s.png";

private static final String SCREENSHOT_SHARE_SUBJECT_TEMPLATE = "Screenshot (%s)";
```

#### 2.2 数据成员

```java
private final SaveImageInBackgroundData mParams;
private final NotificationManager mNotificationManager;
private final Notification.Builder mNotificationBuilder, mPublicNotificationBuilder;

// 保存截图的文件夹路径
private final File mScreenshotDir;

// 截图文件名
private final String mImageFileName;

// 截图文件路径
private final String mImageFilePath;

// 截图事件
private final long mImageTime;
private final BigPictureStyle mNotificationStyle;

// 截图宽度
private final int mImageWidth;

// 截图高度
private final int mImageHeight;
```

#### 2.3 构造方法

```java
SaveImageInBackgroundTask(Context context, SaveImageInBackgroundData data,
        NotificationManager nManager) {
    Resources r = context.getResources();

    // Prepare all the output metadata
    mParams = data;
    // 获取系统时间戳
    mImageTime = System.currentTimeMillis();
    // 构造截图的时间
    String imageDate = new SimpleDateFormat("yyyyMMdd-HHmmss").format(new Date(mImageTime));
    // 构造截图名称，形如：Screenshot_20181201-235132
    mImageFileName = String.format(SCREENSHOT_FILE_NAME_TEMPLATE, imageDate);

    // 并获取外部存储保存图片文件夹的路径
    mScreenshotDir = new File(Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_PICTURES), SCREENSHOTS_DIR_NAME);
    // 构造截图路径
    mImageFilePath = new File(mScreenshotDir, mImageFileName).getAbsolutePath();

    // 构建大的通知图标
    mImageWidth = data.image.getWidth();
    mImageHeight = data.image.getHeight();
    int iconSize = data.iconSize;
    int previewWidth = data.previewWidth;
    int previewHeight = data.previewheight;

    Paint paint = new Paint();
    ColorMatrix desat = new ColorMatrix();
    desat.setSaturation(0.25f);
    paint.setColorFilter(new ColorMatrixColorFilter(desat));
    Matrix matrix = new Matrix();
    int overlayColor = 0x40FFFFFF;

    matrix.setTranslate((previewWidth - mImageWidth) / 2, (previewHeight - mImageHeight) / 2);
    Bitmap picture = generateAdjustedHwBitmap(data.image, previewWidth, previewHeight, matrix,
            paint, overlayColor);

    // Note, we can't use the preview for the small icon, since it is non-square
    float scale = (float) iconSize / Math.min(mImageWidth, mImageHeight);
    matrix.setScale(scale, scale);
    matrix.postTranslate((iconSize - (scale * mImageWidth)) / 2,
            (iconSize - (scale * mImageHeight)) / 2);
    Bitmap icon = generateAdjustedHwBitmap(data.image, iconSize, iconSize, matrix, paint,
            overlayColor);

    mNotificationManager = nManager;
    final long now = System.currentTimeMillis();

    // 构建通知
    mNotificationStyle = new Notification.BigPictureStyle()
            .bigPicture(picture.createAshmemBitmap());

    // The public notification will show similar info but with the actual screenshot omitted
    mPublicNotificationBuilder =
            new Notification.Builder(context, NotificationChannels.SCREENSHOTS_HEADSUP)
                    .setContentTitle(r.getString(R.string.screenshot_saving_title))
                    .setSmallIcon(R.drawable.stat_notify_image)
                    .setCategory(Notification.CATEGORY_PROGRESS)
                    .setWhen(now)
                    .setShowWhen(true)
                    .setColor(r.getColor(
                            com.android.internal.R.color.system_notification_accent_color));
    SystemUI.overrideNotificationAppName(context, mPublicNotificationBuilder, true);

    mNotificationBuilder = new Notification.Builder(context,
            NotificationChannels.SCREENSHOTS_HEADSUP)
        .setContentTitle(r.getString(R.string.screenshot_saving_title))
        .setSmallIcon(R.drawable.stat_notify_image)
        .setWhen(now)
        .setShowWhen(true)
        .setColor(r.getColor(com.android.internal.R.color.system_notification_accent_color))
        .setStyle(mNotificationStyle)
        .setPublicVersion(mPublicNotificationBuilder.build());
    mNotificationBuilder.setFlag(Notification.FLAG_NO_CLEAR, true);
    SystemUI.overrideNotificationAppName(context, mNotificationBuilder, true);

    mNotificationManager.notify(SystemMessage.NOTE_GLOBAL_SCREENSHOT,
            mNotificationBuilder.build());

    /**
     * NOTE: The following code prepares the notification builder for updating the notification
     * after the screenshot has been written to disk.
     */

    // On the tablet, the large icon makes the notification appear as if it is clickable (and
    // on small devices, the large icon is not shown) so defer showing the large icon until
    // we compose the final post-save notification below.
    mNotificationBuilder.setLargeIcon(icon.createAshmemBitmap());
    // But we still don't set it for the expanded view, allowing the smallIcon to show here.
    mNotificationStyle.bigLargeIcon((Bitmap) null);
}
```

```java
/**
 * Generates a new hardware bitmap with specified values, copying the content from the passed
 * in bitmap.
 */
private Bitmap generateAdjustedHwBitmap(Bitmap bitmap, int width, int height, Matrix matrix,
        Paint paint, int color) {
    Picture picture = new Picture();
    Canvas canvas = picture.beginRecording(width, height);
    canvas.drawColor(color);
    canvas.drawBitmap(bitmap, matrix, paint);
    picture.endRecording();
    return Bitmap.createBitmap(picture);
}
```

#### 2.4 后台保存

当执行到 __doInBackground__ 时，图片其实已经成功截取并保存在内存里。

```java
@Override
protected Void doInBackground(Void... params) {
    if (isCancelled()) {
        return null;
    }

    // By default, AsyncTask sets the worker thread to have background thread priority, so bump
    // it back up so that we save a little quicker.
    Process.setThreadPriority(Process.THREAD_PRIORITY_FOREGROUND);

    Context context = mParams.context;
    Bitmap image = mParams.image;
    Resources r = context.getResources();

    try {
        // 创建截屏文件夹的目录
        mScreenshotDir.mkdirs();

        // media provider uses seconds for DATE_MODIFIED and DATE_ADDED, but milliseconds
        // for DATE_TAKEN
        long dateSeconds = mImageTime / 1000;

        // 截图位图写入文件流
        OutputStream out = new FileOutputStream(mImageFilePath);
        image.compress(Bitmap.CompressFormat.PNG, 100, out);
        out.flush();
        out.close();

        // 配置ContentValues
        ContentValues values = new ContentValues();
        ContentResolver resolver = context.getContentResolver();
        values.put(MediaStore.Images.ImageColumns.DATA, mImageFilePath);
        values.put(MediaStore.Images.ImageColumns.TITLE, mImageFileName);
        values.put(MediaStore.Images.ImageColumns.DISPLAY_NAME, mImageFileName);
        values.put(MediaStore.Images.ImageColumns.DATE_TAKEN, mImageTime);
        values.put(MediaStore.Images.ImageColumns.DATE_ADDED, dateSeconds);
        values.put(MediaStore.Images.ImageColumns.DATE_MODIFIED, dateSeconds);
        values.put(MediaStore.Images.ImageColumns.MIME_TYPE, "image/png");
        values.put(MediaStore.Images.ImageColumns.WIDTH, mImageWidth);
        values.put(MediaStore.Images.ImageColumns.HEIGHT, mImageHeight);
        values.put(MediaStore.Images.ImageColumns.SIZE, new File(mImageFilePath).length());
        // 把截屏保存到MediaStore
        Uri uri = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values);

        // 创建分享的Intent
        String subjectDate = DateFormat.getDateTimeInstance().format(new Date(mImageTime));
        String subject = String.format(SCREENSHOT_SHARE_SUBJECT_TEMPLATE, subjectDate);
        Intent sharingIntent = new Intent(Intent.ACTION_SEND);
        sharingIntent.setType("image/png");
        sharingIntent.putExtra(Intent.EXTRA_STREAM, uri);
        sharingIntent.putExtra(Intent.EXTRA_SUBJECT, subject);
        sharingIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);

        // Create a share action for the notification. Note, we proxy the call to
        // ScreenshotActionReceiver because RemoteViews currently forces an activity options
        // on the PendingIntent being launched, and since we don't want to trigger the share
        // sheet in this case, we start the chooser activity directly in
        // ScreenshotActionReceiver.
        PendingIntent shareAction = PendingIntent.getBroadcast(context, 0,
                new Intent(context, GlobalScreenshot.ScreenshotActionReceiver.class)
                        .putExtra(SHARING_INTENT, sharingIntent),
                PendingIntent.FLAG_CANCEL_CURRENT);
        Notification.Action.Builder shareActionBuilder = new Notification.Action.Builder(
                R.drawable.ic_screenshot_share,
                r.getString(com.android.internal.R.string.share), shareAction);
        mNotificationBuilder.addAction(shareActionBuilder.build());

        Intent editIntent = new Intent(Intent.ACTION_EDIT);
        editIntent.setType("image/png");
        editIntent.setData(uri);
        editIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        editIntent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);

        // Create a edit action for the notification the same way.
        PendingIntent editAction = PendingIntent.getBroadcast(context, 1,
                new Intent(context, GlobalScreenshot.ScreenshotActionReceiver.class)
                        .putExtra(SHARING_INTENT, editIntent),
                PendingIntent.FLAG_CANCEL_CURRENT);
        Notification.Action.Builder editActionBuilder = new Notification.Action.Builder(
                R.drawable.ic_screenshot_edit,
                r.getString(com.android.internal.R.string.screenshot_edit), editAction);
        mNotificationBuilder.addAction(editActionBuilder.build());

        // 给通知创建删除截屏的操作
        PendingIntent deleteAction = PendingIntent.getBroadcast(context, 0,
                new Intent(context, GlobalScreenshot.DeleteScreenshotReceiver.class)
                        .putExtra(GlobalScreenshot.SCREENSHOT_URI_ID, uri.toString()),
                PendingIntent.FLAG_CANCEL_CURRENT | PendingIntent.FLAG_ONE_SHOT);
        Notification.Action.Builder deleteActionBuilder = new Notification.Action.Builder(
                R.drawable.ic_screenshot_delete,
                r.getString(com.android.internal.R.string.delete), deleteAction);
        mNotificationBuilder.addAction(deleteActionBuilder.build());

        mParams.imageUri = uri;
        mParams.image = null;
        mParams.errorMsgResId = 0;
    } catch (Exception e) {
        // 外部存储没有挂载会抛出IOException/UnsupportedOperationException
        Slog.e(TAG, "unable to save screenshot", e);
        mParams.clearImage();
        mParams.errorMsgResId = R.string.screenshot_failed_to_save_text;
    }

    // 销毁位图
    if (image != null) {
        image.recycle();
    }

    return null;
}
```

#### 2.5 onPostExecute

```java
@Override
protected void onPostExecute(Void params) {
    if (mParams.errorMsgResId != 0) {
        // 图片保存到存储失败，弹出一个通知告知用户
        GlobalScreenshot.notifyScreenshotError(mParams.context, mNotificationManager,
                mParams.errorMsgResId);
    } else {
        // Show the final notification to indicate screenshot saved
        // 图片保存到存储成功，弹出通知指明已保存的截屏
        Context context = mParams.context;
        Resources r = context.getResources();

        // Create the intent to show the screenshot in gallery
        Intent launchIntent = new Intent(Intent.ACTION_VIEW);
        launchIntent.setDataAndType(mParams.imageUri, "image/png");
        launchIntent.setFlags(
                Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_GRANT_READ_URI_PERMISSION);

        final long now = System.currentTimeMillis();

        // Update the text and the icon for the existing notification
        mPublicNotificationBuilder
                .setContentTitle(r.getString(R.string.screenshot_saved_title))
                .setContentText(r.getString(R.string.screenshot_saved_text))
                .setContentIntent(PendingIntent.getActivity(mParams.context, 0, launchIntent, 0))
                .setWhen(now)
                .setAutoCancel(true)
                .setColor(context.getColor(
                        com.android.internal.R.color.system_notification_accent_color));
        mNotificationBuilder
            .setContentTitle(r.getString(R.string.screenshot_saved_title))
            .setContentText(r.getString(R.string.screenshot_saved_text))
            .setContentIntent(PendingIntent.getActivity(mParams.context, 0, launchIntent, 0))
            .setWhen(now)
            .setAutoCancel(true)
            .setColor(context.getColor(
                    com.android.internal.R.color.system_notification_accent_color))
            .setPublicVersion(mPublicNotificationBuilder.build())
            .setFlag(Notification.FLAG_NO_CLEAR, false);

        mNotificationManager.notify(SystemMessage.NOTE_GLOBAL_SCREENSHOT,
                mNotificationBuilder.build());
    }
    mParams.finisher.run();
    mParams.clearContext();
}
```

#### 2.6 onCancelled

```java
@Override
protected void onCancelled(Void params) {
    // If we are cancelled while the task is running in the background, we may get null params.
    // The finisher is expected to always be called back, so just use the baked-in params from
    // the ctor in any case.
    mParams.finisher.run();
    mParams.clearImage();
    mParams.clearContext();

    // Cancel the posted notification
    mNotificationManager.cancel(SystemMessage.NOTE_GLOBAL_SCREENSHOT);
}
```

##DeleteImageInBackgroundTask

在后台删除媒体存储的图片，此类继承 __AsyncTask__。

```java
class DeleteImageInBackgroundTask extends AsyncTask<Uri, Void, Void> {
    private Context mContext;

    DeleteImageInBackgroundTask(Context context) {
        mContext = context;
    }

    @Override
    protected Void doInBackground(Uri... params) {
        if (params.length != 1) return null;

        // 从参数获取截图的资源路径Uri
        Uri screenshotUri = params[0];
        // 获取ContentResolver
        ContentResolver resolver = mContext.getContentResolver();
        //  从ContentResolver移除指定图片
        resolver.delete(screenshotUri, null, null);
        // 给方法返回值类型Void返回null
        return null;
    }
}
```

