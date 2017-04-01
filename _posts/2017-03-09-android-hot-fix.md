---
layout:     post
title:      "动态加载应用之简单热更新"
subtitle:   "动态加载&热更新"
date:       2017-03-09
catalog:  true
author:     "lwenkun"
header-img: "img/post-bg-android-hot-fix.jpg"
tags:
    - android
    - 动态加载
    - 热更新
---

# 动态加载应用之简单热更新 #
在之前的一片文章中，我和大家分享了[安卓动态加载的原理](/2016/11/11/android-load-class-dynamically/)，这篇文章就和大家分享一下动态加载在热更新方面的应用，我会通过一个假想的案例来演示如何在项目利用动态加载实现热更新。

## 案例 ##

某项目需要对某些数据集进行排序，但由于时间原因，开发团队使用了一个比较基础的算法实现该排序功能。上线之后才发现该排序算法的性能太差，影响了用户体验，因此开发团队不得不对该算法进行优化。优化后的新版本经过测试、发布、审核成功上线，但对项目进度造成了一定的影响。经历这次事件之后，开发团队吸取了教训：对于经常变动或者需要后期优化的功能，采用热更新技术避免重新发布新版本，从而降低人力成本。接下来我就用一个 demo 来模拟开发团队这次热更新技术的应用。

## 实践 ##
### 在项目中插入热更新逻辑 ###
因为热更新需要从外部加载代码，而项目中的原本的代码并不能预知外部代码细节，因此我们需要定义一份接口实现项目中的代码和外部代码的对接。定义的接口如下：

```java
public interface Sort {
    String getName(); // 获取排序算法的名称
    void sort(int a[]);
}
```
不同的排序策略可以对该接口采用不同的实现，我们当前在项目采用的排序策略是插入排序，其实现如下：

```java
public class InsertSort implements Sort {
    @Override
    public String getName() {
        return "插入排序";
    }

    @Override
    public void sort(int[] a) {
        for (int i = 1; i < a.length; i ++) {
            int key = a[i];
            int pos = i;
            for(int j = i - 1 ; j >= 0; j--) {
                if (a[j] < key) break;
                a[j + 1] = a[j];
                pos = j;
            }
            a[pos] = key;
        }
        System.out.println("I'm InsertSort");
    }
}
```
把以上两个类放入项目中，同时创建一个 `SortActivity`，在该 `Activity` 中对排序算法进行了应用：

```java
public class SortActivity {
    private static final int[] collection = {21, 70, 1, 88, 4, 54, 22, 10, 9, 104, 37};
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dynamic_loading);
        TextView tvResult = (TextView) findViewById(R.id.tv_result);
        
        Sort sort = getDefaultSort();
        sort.sort(collection);
        // 显示排序结果
        StringBuilder result = new StringBuilder(sort.getName() + ": ");
        for (int i = 0; i < collection.length; i++) {
            result.append(collection[i]);
            if (i != collection.length - 1) result.append(", ");
        }
        tvResult.setText(result.toString());
    }
    
    private Sort getDefaultSort() {
        return new InsertSort();
    }
}
```
上面的代码和采用热更新技术之前的项目代码无异，为了达到热更新的目的，我们要把关于热更新这一部分的逻辑写在程序中：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_dynamic_loading);
    
    Sort sort;
    if (hasUpdate()) { //如果排序算法有更新
        sort = getNewSort(); // 应用新的排序算法
    } else {
        sort = getDefaultSort(); // 否则用默认实现的排序算法
    }
    sort.sort(collection)
    // 显示排序结果
    ......
}

private Sort getNewSort() {
    Sort newSort = null;
    try {
        Class<?> clazz = getNewSortClass(); // 获取新的 Sort 实现类
        newSort = (Sort) clazz.newInstance(); // 创建该实现类的实例
    } catch (Exception e) {
        e.printStackTrace();
    }
    return newSort; // 用新的排序算法实现排序
}

private boolean hasUpdate(){
    // 根据需求自行实现，如询问服务器，这里通过读取配置文件内容模拟向服务器询问的过程
    try {
        // 读取配置文件，默认值为 0，即没有更新
        FileReader reader = new FileReader(Environment.
            getExternalStorageDirectory().getAbsolutePath()
            + File.separator + "has_update.txt");
        int hasUpdate = reader.read();
        System.out.println("hasUpdate:" + hasUpdate);
        return hasUpdate == '1';
    } catch (Exception e) {
        e.printStackTrace();
    }
    return false;
}
private Class<?> getNewSortClass() throws ClassNotFoundException {
    // 从服务器下载到本地磁盘
    downLoadCodeFromServer();
    // 假定下载的 .dex 文件在外部存储的根目录下，文件名为 newsort.dex
    String path =  Environment.getExternalStorageDirectory().getAbsolutePath() 
            + File.separator + "newsort.dex";
    // .dex 文件优化输出后的目录为 “optimized_dir”
    String optimizedDir = getDir("optimized_dir", MODE_PRIVATE).getAbsolutePath();
    // 创建一个 DexClassLoader，用来加载从网络上下载的 .dex 文件
    DexClassLoader loader = new DexClassLoader(path, optimizedDir, null, getClassLoader);
    // 新的排序实现类的全名为 me.liwenkun.demo.NewSort，该名称也可从服务端或通过其他方式动态
    // 获取，这里为了简化我们约定新的实现类的类名必须是 NewSort
    loader.loadClass("me.liwenkun.demo.NewSort")
}
```
对于上面的代码，需要说明几点：

- 为了方便，我把网络请求和文件读取放在了主线程中，实际项目中不可能这样做；
- 我忽略了权限相关的问题，大家在 demo 中一定要注意读写权限的配置，特别是 6.0 之后的动态权限申请，否则很可能导致热更新失败；
- 优化后的输出目录一定要是私有的内部存储目录，否则在高版本的系统中会有运行时错误，因为这被认为是不安全的；从服务器下载的 .dex 的位置虽然没有限制，但安全起见最好也放置在内部私有目录中，本例中为了方便起见直接放在外部存储的根目录下（否则手机得 root）；
- 如果关于 `DexClassLoader` 这个类加载器的原理和用法不熟悉的话，最好先看看我的[这篇文章](/2016/11/11/android-load-class-dynamically/)。

至此，项目中关于热更新的逻辑就写完了。现在我们模拟上线，把应用装进手机并运行，结果如下图：

![](/img/in-post/post_android_dynamically_loading_practice/insert.jpg)

### 实施热更新 ###

如果现在觉得这种排序算法还不够好，只需将新的 `Sort` 实现类编译成 .dex 文件发送给客户端就可以了，不必推送新的版本。比如我想用希尔排序取代插入排序，于是在项目中编写了 `NewSort` 类：

```java
public class NewSort implements Sort {
    @Override
    public String getName() {
        return "希尔排序";
    }

    @Override
    public void sort(int[] a) {
        int h = 1;
        int N;
        for(N = a.length; h < N / 25; h = h * 3 + 1);

        while(h >= 1) {
            for(int i = h; i < N; ++i) {
                for(int j = i; j >= h && a[j] < a[j - h]; j -= h) {
                    exch(a, j, j - h);
                }
            }
            h /= 3;
        }
    }

    private void exch(int[] a, int src, int dest) {
        int temp = a[src];
        a[src] = a[dest];
        a[dest] = temp;
    }
}
```

编译之后，我们需要将此类以 jar 包的形式输出，为之后生成 .dex 文件做好准备。因此我们找到 Android Studio 的编译输出文件夹（我这里的位置是 `Demo/app/build/intermediates/classes/debug/`），找到 NewSort.class，连同 package 将其移出，结果如下图所示：

![](/img/in-post/post_android_dynamically_loading_practice/structure.png)

在 me 的上一级目录处打开终端，输入命令 `jar cf newsort.jar me`，执行完后我们在与 me  同级的位置可以看到打包好的 newsort.jar 文件。（关于 .jar 文件的打包可以参考 [命令行编译、打包并运行一个 java 程序](/2016/11/05/compile-java-through-command-line/) 这篇文章。）

接下来我们进入 sdk 目录，找到 build-tools 文件夹，该文件夹下有各个版本的 build-tools，如下图所示，我们选择最新的 25.0.2，进入之后会发现有一个名为 dx 的工具，它可以将 jar 包转换成 .dex 文件。所以把我们的 newsort.jar 放入 25.0.2 目录下，然后在 25.0.2 这个目录位置打开终端，输入命令 `./dx --dex --output=/Users/lwenkun/desktop/newsort.dex newsort.jar ` ，命令执行完后在 desktop 文件夹下（也就是桌面）可以看到生成了 newsort.dex 文件，这个文件就是我们最终需要的。

![](/img/in-post/post_android_dynamically_loading_practice/build-tools.png)

准备好了 .dex 文件后，我们就可以模拟热更新了。我们假设 .dex 文件已经从服务器下载到了本地，因此直接把 newsort.dex 文件拷贝到手机外部存储的根目录下。然后修改根目录下的 has_update.txt 文件，将其中的值修改为 1。然后我们退出应用，等再次打开应用时，插入排序已经成功替换成了希尔排序：

![](/img/in-post/post_android_dynamically_loading_practice/shell.jpg)

## 感谢阅读 ##
动态加载在热更新方面的应用就分享到这里了，网上关于这方面的文章也有很多，大家可以找写这方面的文章看看。由于水平和理解能力有限，有什么不对的地方还望大家不吝赐教，感谢大家的阅读。

参考资料：[安卓动态加载](https://www.google.com/search?q=%E5%AE%89%E5%8D%93%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD&oq=%E5%AE%89%E5%8D%93%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD&aqs=chrome..69i57j69i61.302j0j9&sourceid=chrome&ie=UTF-8)