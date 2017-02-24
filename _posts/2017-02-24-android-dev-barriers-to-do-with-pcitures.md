---
layout:     post
title:      "android 记坑（图片相关）"
subtitle:   "android 记坑"
date:       2017-02-24
author:     "lwenkun"
header-img: "img/post-bg-android-dev-barriers-to-do-with-pcitures.png"
tags:
- android
- barriers
---

# android 记坑（图片相关）#
## 调用相机获取图片 ##
有两种方法获取图片，分别对应不同的需求：一种是获取缩略图，适合用来做头像或者其他比较小的 icon ；另一种是获取原图，如果有保存或查看原图的需求，就应该用这种方法。

### 获取缩略图 ###

```java
Intent getThumbnail = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
File file = createEmptyFile();
if (getThumbnail.resolveActivity(getPackageManager()) != null)
    startActivityForResult(getThumbnail, REQUEST_GET_THUMBNAIL);
```

这样就调用了相机拍照，然后我们在 onActivityResult() 中获取图片：

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_GET_THUMBNAIL) {
        if (resultCode == RESULT_OK) {
            Bitmap bitamp;
            if (data != null) 
                bitmap = (Bitmap) data.getExtras().get("data");
            thumbnail.setImageBitmap(bitmap);
        }
    }
}
```

### 获取原图 ###

```java
Intent getBigPhoto = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
File file = createEmptyFile(); //用于输出原图
currentFile = file;
Uri uri = Uri.fromFile(file);//将 file 转换成 uri 格式
getBigPhoto.putExtra(MediaStore.EXTRA_OUTPUT, file);
startActivityForResult(Intent, REQUEST_GET_BIG_PHOTO);
```
然后图片就会自动输出到你指定的文件，如果同时你又想获取原图，你就会这样做：

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_GET_BIG_PHOTO) {
        if (resultCode == RESULT_OK) {
            Bitmap bitamp;
            if (data != null) 
                bitmap = (Bitmap) data.getExtras().get("data");
            thumbnail.setImageBitmap(bitmap);
        }
    }
}
```
好了，看似一切正常，但是总有一些你意想不到的异常发生，我来指出这之中有问题的几个坑吧（权限问题就不讲了）吧：

- file 转换成 uri 格式这一步时，如果你是上面那样转换的，那么在 android N 上你就会得到 FileUriExposedException，因为这样做是不安全的，你将文件的真实 uri 暴露出去了，其他应用可能对这个文件有完全的控制权。安全的做法是通过 [FileProvider.getUriForFile(Context, String, File)](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html#getUriForFile(android.content.Context, java.lang.String, java.io.File)) 暴露所要提供的 Uri。这样提供给其他的应用的 Uri 不会暴露出文件的真实位置，因为这只是这是文件的一个 "镜像地址"，其他应用只能调用系统功能才能将此 Uri 定位到实际位置。
- 如果你想同时获取原图和缩略图，那么你就错了，这二者不可兼得。每次调用相机，只能获取缩略图或者原图，这和你是否指定 MediaStore.EXTRA_OUTPUT 附加字段有关：如果没有指定，说明你并不想将获取的图片输出（即保存为原图)，此时 onActivityResult() 的第三个参数就不为 null，你可以从中取出缩略图；但是如果指定了，说明你想将照片输出为原图，那么第三个参数就为 null，你并不能从中获取缩略图。

## 将图片添加至媒体库 ##

我们的应用经常有将图片保存到磁盘的需求，但是保存的图片在下一次开机前媒体库是检测不到的，也就是说如果我们在图库中是看不到保存的图片的，因此就无法和其他应用共享。要将保存的图片加入系统的媒体库中需要做一点额外的工作，下面介绍实现方法：

### 发送广播通知媒体库扫描此图片 ###
这一方法在安卓官方培训[教程](https://developer.android.google.cn/training/camera/photobasics.html)中有提到，它是这样做的：

```java
private void galleryAddPic() {
    Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    File f = new File(mCurrentPhotoPath);
    Uri contentUri = Uri.fromFile(f);
    mediaScanIntent.setData(contentUri);
    this.sendBroadcast(mediaScanIntent);
}
```

### 和系统进程通信将图片添加至媒体库 ###
另外一种是通过 MediaScannerConnection 实现的，这一方法的原理就是和系统进程进行通信，具体做法是这样的：

```java
public Class Client implements MediaScannerConnectionClient {

    private MediaScannerConnection conn;
    private String filePath;
    private String mimeType;
    
    public Client(Context c, String filePath, String mimeType) {
        conn = new MediaScannrConnection(context, this);
    }

    @Override
    public void onMediaScannerConnected() {
        conn.scanFile(filePath, mimeType);
    }

    @Override
    public void onScanCompleted(String path, Uri uri) {
        Log.d("Client", "path:" + path + "; uri:" + uri);
    }
    
    public void scanFile(String path, String mimeType) {
        this.filePath = filePath;
        this.mimeType = mimeType;
        conn.connect();
    }
}
```

### 直接操作媒体库的数据库 ###
第三种方法是直接将图片信息直接添加媒体库的数据库中，其实前面两种方法的底层实现也是这样的，只是屏蔽了相关细节，避免我们直接接触数据库。因为要和操作其他应用的数据数据，所以要用到 ContentResolver，具体实现是这样的：

```java
ContentValues values = new ContentValues();

values.put(Images.Media.DATE_TAKEN, System.currentTimeMillis());
values.put(Images.Media.MIME_TYPE, "image/jpeg");
values.put(MediaStore.MediaColumns.DATA, filePath);

context.getContentResolver().insert(Images.Media.EXTERNAL_CONTENT_URI, values);
```
下面分别介绍其中遇到的坑：

- 第一种方法在 android 4.4 以下时是可以的，但是有个问题就是通过广播将单个文件添加至媒体库比较浪费资源；在 android 4.4 会出现 SecurityException，原因是 android 4.4 禁止非系统应用发送系统广播，而扫描媒体文件的广播在此之类，因此运行时会崩溃；但是我在 android 5.0 至 7.1 上实测却可行，或许系统开发了此权限。
- 第二种和第三种方法没有版本限制，是通用的方法。
- 前两种方法有一点要注意，如果该图片存放至系统的私有文件夹（无论内部存储还是外部存储），媒体库都无法扫描到该文件（没有权限）；第三种方法则没有这种限制，因为它并不是通过扫描，而是直接操作数据库的方式添加的。

--- 未完待续
