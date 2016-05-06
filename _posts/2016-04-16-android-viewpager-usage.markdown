---
layout:     post
title:      "ViewPager 的使用"
subtitle:   "ViewPager"
date:       2016-04-16
author:     "lwenkun"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - android
    - View
    - ViewPager
---
>什么是ViewPager

图告诉你一切：

<table border="0px">
<tbody>
<tr>
<td><a href="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_1.png" rel="attachment wp-att-123"><img class="alignnone wp-image-123" src="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_1.png" alt="ViewPager" width="112" height="198" /></a></td>
<td><a href="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_2.png" rel="attachment wp-att-127"><img class="alignnone wp-image-127" src="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_2.png" alt="Screenshot_2016-04-04-17-00-18" width="114" height="203" /></a></td>
<td><a href="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_3.png" rel="attachment wp-att-126"><img class="alignnone wp-image-126" src="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_3.png" alt="Screenshot_2016-04-04-17-00-28" width="113" height="200" /></a></td>
<td><a href="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_4.png" rel="attachment wp-att-128"><img class="alignnone wp-image-128" src="/img/in-post/post_android_viewpager_usage/android_viewpager_usage_4.png" alt="Screenshot_2016-04-04-16-59-21" width="112" height="199" /></a></td>
</tr>
</tbody>
</table>

如图，白色矩形区域就是我们的 `ViewPager`，正如我们所熟悉的，`ViewPager` 通常会配合 `tab` 使用，什么是 `tab`？就是“网易新闻，网易体育，网易财经，网易女人”（page title) 这四个标题所在的那个区域，每个 page title 代表着一个 `tab` 。这个区域再上面一点就是 `ToolBar` 了。 `tab` 的实现方法有很多种，但现在最受欢迎并且最美观的就是谷歌MD提供的 `TabLayout` 了。现在我们主要讲下 `TabLayout` + `ViewPager` 的实现和使用吧。

<strong>布局文件</strong>：

```xml
<android.support.design.widget.TabLayout
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:tabIndicatorColor="#ffffff" 
        app:tabSelectedTextColor="#ffffff"
        app:tabTextColor="#aaffffff"
        app:tabBackground="@drawable/tab_ripple"/>

<android.support.v4.view.ViewPager
      android:id="@+id/viewpager"
      android:layout_width="match_parent"
      android:layout_height="match_parent"/>
```

这就是 `TabLayout` + `ViewPager` 的布局文件 `TabLayout` 在上，`ViewPager` 在下。

<strong>java代码</strong>：

```java
    ViewPager pager = null;
    TabLayout tabs = null;
    ArrayList<View> viewContainer = new ArrayList<>();
    ArrayList<String> titleContainer = new ArrayList<>();

     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
    }

    public void initView() {
       //其他布局初始化代码省略
        pager = (ViewPager) findViewById(R.id.viewpager);
        tabs = (TabLayout) findViewById(R.id.tabs);
    }
```

在代码中找到这两个控件之后，我们就要给他们添加数据了。总不能两个控件都是空内容吧。对于 `TabLayout` 来说，我们要给它添加的数据当然就是page title了；而对于 `ViewPager` 来说， `ViewPager` 是一个 `ViewGroup` 类，我们需要给他添加的数据就是添加四个子 `View`，这四个子`View`对应的就是我们的四个页面。怎么添加呢？继续看。

对于 `TabLayout`，我们可以将数据源放在一个 `ArrayList` 中：

```java
ArrayList<String> titleContainer = new ArrayList<>();

titleContainer.add("网易新闻");
titleContainer.add("网易体育");
titleContainer.add("网易财经");
titleContainer.add("网易女人");
```

对于 `ViewPager`，我们同样将数据源放入一个 `ArrayList` 中：

```java
ArrayList<View> viewContainer = new ArrayList<>();

TextView view1,view2,view3,view4;
(view1 = new TextView(this)).setText("页面一");
(view2 = new TextView(this)).setText("页面二");
(view3 = new TextView(this)).setText("页面三");
(view4 = new TextView(this)).setText("页面四");

viewContainer.add(view1);
viewCOntainer.add(view2);
viewContainer.add(view3);
viewContainer.add(view4);
```

这里我们给这个 `ViewPager` 添加的四个子 `View` 是四个 `TextView`;

好了数据源都准备好了，那么我们应该怎样将数据送到相应的控件中呢？对于 `TabLayout`，方法有两种，我们先讲一种比较容易理解的：

```java
for(String title : titleContainer) {
   tabs.add(tabs.newTab().setText(title));  //setText()的返回值是new出来的tab
}
```

然后我们得将 `viewContainer` 中的那四个 `view` 添加进 `ViewPager` 中，当然，不是我们手动的 `addView` 去添加，而是像那些包含 `itemView` 的 `AdapterView` 那样去设置 `Adapter`。适配器模式有很多好处，首先能够将视图展示和数据绑定分离，其次能够很好的管理子 `View`，如动态的添加移除，还可以设置动画，并且能够实现视图回收和重用，如果手动添加这些是很难做到的。

安卓提供了一个 `PagerAdapter` 来给 `ViewPager` 设置适配器，我们的子 `View` 就是通过这个适配器添加进 `ViewPager` 的：

新建一个 `PagerAdapter` 通常要重写和实现的方法有：

```java
//获取页面的数量
public int getCount();

//判断某个子view是否和object关联，不理解这个方法没关系，官方推荐直接返回 view == object 
//就可以了 
public boolean isViewFromObject(View view, Object object); 

//添加一个页面并返回关联这个页面的object,对应上面的我们只需这样做： 
//return viewContainer.get(position),即把子View返回 
public Object instantiateItem(ViewGroup container, int position); 

//销毁一个页面 
public void destroyItem(ViewGroup container, int position, Object object); 

//获取当前页面的page title，这个方法可以不重写，因为我们之前手动添加tab并且设置了 
//page title
public CharSequence getPageTitle(int position）;
```

一个典型的 `PagerAdapter` 可以这样写：

```java
pager.setAdapter( new PagerAdapter() {

            @Override
            public int getCount() {
                return  viewContainer.size();
            }

            @Override
            public boolean isViewFromObject(View view, Object object) {
                return  view == object;
            }

            @Override
            public Object instantiateItem(ViewGroup container, int position) {
                View view;
                container.addView(view = viewContainer.get(position));
                return view;
            }

            @Override
            public void destroyItem(ViewGroup container, int position,  Object
            object) {                                         
                container.removeView(viewContainer.get(position));
            }

            //可以不重写
            @Override
            public CharSequence getPageTitle(int position) {
                return titleContainer.get(position);
            }
       });
```

如果我们想要监听 `ViewPager` 页面滑动的事件，我们可以添加监听器：

```java
pager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
       @Override
        public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
             Log.d("onPageSrolled：", "我监听页面滚动事件");
        }

        @Override
        public void onPageSelected(int position) {
              Log.d("onPageSelected: ", "我监听页面选择事件");
        }

        @Override
        public void onPageScrollStateChanged(int state) {
             Log.d("onPageSrollStateChanged: ", "我监听页面滚动状态改变的事件");
        }
    });
```

现在你肯定会问怎么将 `ViewPager` 和 `TabLayout` 关联起来呢？代码中两者之间没有任何关联，但我们通常看到的是指示条会跟着 `ViewPager` 页面的切换滚动到相应的 page title 下，而且当前页面对应的 page title 会更“亮”。对，这也就是我当时碰到的一个坑， 少写了这一步：

```java
pager.addOnPageChangeListener(new   TabLayout.TabLayoutOnPagerChangeListener(tabs));
```

有了这一步，`tabs` 就能够监听 `ViewPager` 切换的事件了，从而能够做出相应改变。

前面讲了，给 `TabLayout` 设置 page title 的方法有两种，其中一种是手动添加 ，那么现在就讲另一种方法：

这种方法是把 `TabLayout` 设置 page title 的的任务交给 `PagerAdapter` 来做，最后将 `TabLayout` 和 `ViewPager` 绑定，这样 `TabLayout` 就和 `PagerAdapter` 产生某种关联，从而 `TabLayout` 能够获取标题信息。具体的做法是这样的：

首先重写 `PagerAdapter` 中的 `getPageTitle()` 方法，因为我们没有手动添加 page title，而是交给 `PagerAdapter` 来处理，所以我们必须重写这个方法来获取 page title：

```java
@Override
public CharSequence getPageTitle(int position) {
    return titleContainer.get(position);
}
```

之后，我们就要将 `TabLayout` 和 `ViewPager` 绑定：

```java
tabs.setupWithViewPager(pager);
```


这样也能达到我们预期的效果了，而且我们连这一步：

```java
pager.addOnPageChangeListener(new   TabLayout.TabLayoutOnPagerChangeListener(tabs));
```

也不用写了，但是有一点要记住这个方法一定要是在 `ViewPager ` 设置 `PagerAdapter` 之后调用，否则程序运行时会崩溃。

如果你的手机是安卓5.0以上，你的页面可能会出现下图这种情况：

![](/img/in-post/post_android_viewpager_usage/android_viewpager_usage_5.png)

在 `ToolBar` 和 `TabLayout` 之间出现了一条阴影，这是因为 `Toolbar` 在安卓5.0以上默认 `elevation` 是不为零的，而 `TabLayout` 的高度为零，所以在 `TabLayout` 上造成了阴影，解决办法是将 `TabLayout` 放进 `AppBarLayout` 这个容器里，如下：

```java
<android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/AppTheme.AppBarOverlay">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/AppTheme.PopupOverlay" />

        <android.support.design.widget.TabLayout
            android:id="@+id/tabs"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:tabIndicatorColor="#ffffff"
            app:tabSelectedTextColor="#ffffff"
            app:tabTextColor="#aaffffff"
            app:tabBackground="@drawable/tab_ripple"/>

</android.support.design.widget.AppBarLayout>
```

这样就不会出现那条难看的阴影了。