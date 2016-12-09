---
layout:     post
title:      "Context 获取目录和路径"
subtitle:   "通过 Context 获取目录、路径"
date:       2016-10-29
author:     "lwenkun"
header-img: "img/post-bg-android-context-dirs.png"
tags:
    - android
    - Context
---

# 通过 Context 获取目录、路径 #

```java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Context c = getApplicationContext();

        Log.d("getDatabasePath", c.getDatabasePath("data_base_path").toString());
        Log.d("getPackageCodePath", c.getPackageCodePath().toString());
        Log.d("getCacheDir", c.getCacheDir().toString());
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.N)
            Log.d("getDataDir", c.getDataDir().toString());
        Log.d("getExternalCacheDir", c.getExternalCacheDir().toString());
        Log.d("getFilesDir", c.getFilesDir().toString());
        Log.d("getObbDir", c.getObbDir().toString());
        Log.d("getObbDirs", c.getObbDirs()[0].toString());
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.LOLLIPOP)
            Log.d("getNoBackupFilesDir", c.getNoBackupFilesDir().toString());
        Log.d("getExternalFilesDirs", c.getExternalFilesDirs("external_files_dirs")[0].toString());
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.LOLLIPOP)
            Log.d("getExternalMediaDirs", c.getExternalMediaDirs()[0].toString());
        Log.d("getDir", c.getDir("get_dir", MODE_PRIVATE).toString());
        Log.d("getPackageResourcePath", c.getPackageResourcePath());
        Log.d("getFileStreamPath", c.getFileStreamPath("file_system_stream_path").toString());

}
```

Log ：

```java
10-30 01:35:09.998 15213-15213/net.bingyan.hustpass.demo D/getDatabasePath: /data/user/0/net.bingyan.hustpass.demo/databases/data_base_path
10-30 01:35:09.998 15213-15213/net.bingyan.hustpass.demo D/getPackageCodePath: /data/app/net.bingyan.hustpass.demo-1/base.apk
10-30 01:35:09.998 15213-15213/net.bingyan.hustpass.demo D/getCacheDir: /data/user/0/net.bingyan.hustpass.demo/cache
10-30 01:35:10.006 15213-15213/net.bingyan.hustpass.demo D/getExternalCacheDir: /storage/emulated/0/Android/data/net.bingyan.hustpass.demo/cache
10-30 01:35:10.006 15213-15213/net.bingyan.hustpass.demo D/getFilesDir: /data/user/0/net.bingyan.hustpass.demo/files
10-30 01:35:10.012 15213-15213/net.bingyan.hustpass.demo D/getObbDir: /storage/emulated/0/Android/obb/net.bingyan.hustpass.demo
10-30 01:35:10.016 15213-15213/net.bingyan.hustpass.demo D/getObbDirs: /storage/emulated/0/Android/obb/net.bingyan.hustpass.demo
10-30 01:35:10.016 15213-15213/net.bingyan.hustpass.demo D/getNoBackupFilesDir: /data/user/0/net.bingyan.hustpass.demo/no_backup
10-30 01:35:10.021 15213-15213/net.bingyan.hustpass.demo D/getExternalFilesDirs: /storage/emulated/0/Android/data/net.bingyan.hustpass.demo/files/external_files_dirs
10-30 01:35:10.028 15213-15213/net.bingyan.hustpass.demo D/getExternalMediaDirs: /storage/emulated/0/Android/media/net.bingyan.hustpass.demo
10-30 01:35:10.029 15213-15213/net.bingyan.hustpass.demo D/getDir: /data/user/0/net.bingyan.hustpass.demo/app_get_dir
10-30 01:35:10.029 15213-15213/net.bingyan.hustpass.demo D/getPackageResourcePath: /data/app/net.bingyan.hustpass.demo-1/base.apk
10-30 01:35:10.029 15213-15213/net.bingyan.hustpass.demo D/getFileStreamPath: /data/user/0/net.bingyan.hustpass.demo/files/file_system_stream_path
```
通过日志很容易能够看出各个方法获取 `File(s)` 对象的路径。<br>
其中 `/data/user/0` == `/data/data`。

