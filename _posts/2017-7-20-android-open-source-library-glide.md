---
layout:     post
title:      "Glide 源码探究"
subtitle:   "Glide 源码探究"
date:       2017-07-20
author:     "lwenkun"
catalog:  true
header-img: "img/post-bg-how-does-messenger-work.png"
tags:
    - Android
    - 开源库
    - Glide
---


# Glide 源码探究
## 前言
Glide 是安卓平台上媒体管理和图片加载框架，它内部封装了媒体解码工具、内存和磁盘缓存以及资源池等，并向用户暴露简单易用的接口。我们可以用它来获取、解码、并展示视频、图片和 GIF 动画。如果大家有用过 Picasso 应该知道，Glide 的使用方式和 Picasso 非常相似，甚至很多 API 的名称都一样。但是相比 Picasso，Glide 的功能更加强大，内部实现也更加复杂。接下来我们就从源码的角度来探究一下 Glide 的内部原理。

## Glide 的简单使用
在探究 Glide 的原理之前，我们先熟悉一下它的常见 API，这样有助于我们后面的分析。Glide 的 API 使用了流式 API 风格，加载图片一行代码就能搞定：

```java
Glide.with(context).load(url).into(imageView);
```
当然，以上只是 Glide 最基础、最常见的用法，它的功能远不止于此，基本上关于图片加载的需求它都能满足。下面我们就顺着这个 API 进入到 Glide 内部一探究竟。

## Glide 原理
### with()
首先从 Glide 的 with(Context) 静态方法看起，需要注意的是，这个方法返回的不是 Glide 实例，而是一个 RequestManager 对象，用来管理图片加载请求。with() 方法有多个重载，除了接受 Context 对象，它还可以接受 Activity，Fragment，View 等作为它的参数，我们来看下这些重载方法：

```java
public static RequestManager with(Context context) {
    return getRetriever(context).get(context);
}

public static RequestManager with(Activity activity) {
    return getRetriever(activity).get(activity);
}

public static RequestManager with(android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);   
}

public static RequestManager with(View view) {
    return getRetriever(view.getContext()).get(view);
}
```
上面这几个重载方法长得都差不多，并且都调用了 getRetriver(Context) 方法：

```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    ......
    return Glide.get(context).getRequestManagerRetriever();
}
```

在这个方法中，创建了 Glide 单例，然后通过它的 getRequestManagerRetriever() 方法获取并返回了一个 RequesetManagerRetriever 对象。RequestManagerRetriever 是 ReqeustManager 的构建者，它会根据传入 get() 方法的参数类型来创建不同的 RequestManager 实例。

先来分析下如果传入的参数类型是 Context 时，RequestManager 是怎样构建的：

```java
public RequestManager get(Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }
  return getApplicationManager(context);
}
```
这个方法的逻辑是：首先对参数进行检查，如果为空抛出异常；如果当前线程不是主线程或者 context 为 Application 对象，那么就反回 getApplicatinManager(Context) 的调用结果；如果前面的条件都不成立，就根据 context 的实际类型调用相应的 get() 重载方法，get() 方法可接受的参数类型和 with() 是一致的。现在进一步假设我们传入的 Context 的实际类型为 Activity，这样就进入到了 get(Activity) 方法内：

```java
public RequestManager get(Activity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    android.app.FragmentManager fm = activity.getFragmentManager();
    return fragmentGet(activity, fm, null /*parentHint*/);
  }
}
```
这里会先判断当前线程是不是后台线程：如果是，就会调用 get(Context) 方法，因为传入的 Context 是 ApplicationContext，因此这个方法最终返回的是 getApplicationManager(Context) 的调用结果；否则，在 Activity 还没销毁的前提下，会返回 fragmentGet() 的调用结果。

通过上面两个 get() 方法我们可以知道，RequestManager 对象的创建会根据 Glide 加载图片时使用的上下文环境和线程的不同而不同。并且，如果线程是后台线程，不管是什么上下文环境，最终都会调用 getApplicatinManager(Context) 来创建 RequestManager 对象。

<!--Glide 之所以这样做，可能是为了节省资源。通常来说，当 UI 不可见或者不可交互时可以停止加载图片。而 UI 视图通常都是和 Activity 和 Fragment 关联的，它们的生命周期反映了 UI 的可见性和可交互性，从而影响了图片加载的必要性。因此，对于有生命周期的上下文环境来说，比如 Activity、Fragment、View，Glide 选择使用获取它-->

假设我们的加载动作发生在主线程，则 fragmentGet() 方法会得到调用，这个方法源码如下：

```java
private RequestManager fragmentGet(Context context, android.app.FragmentManager fm,
    android.app.Fragment parentHint) {
  RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode());
    current.setRequestManager(requestManager);
  }
  return requestManager;
}
```
这个方法会先通过 `getRequestManagerFragment()` 方法来查找当前 FragmentManager 中是否保存着指定的 `Fragment`，其逻辑如下：

```java
RequestManagerFragment getRequestManagerFragment(
    final android.app.FragmentManager fm, android.app.Fragment parentHint) {
  RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
  if (current == null) {
    current = pendingRequestManagerFragments.get(fm);
    if (current == null) {
      current = new RequestManagerFragment();
      current.setParentFragmentHint(parentHint);
      pendingRequestManagerFragments.put(fm, current);
      fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
      handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
    }
  }
  return current;
}
```
如果指定的 Fragment（RequestManagerFragment） 不存在，那么就会创建一个并返回。这里有一个比较奇怪的地方：为什么要创建 Fragment ？其实 Glide 在这里使用了 “奇技淫巧”，我们看下 RequestManagerFragment 的文档说明：

>A view-less {@link android.app.Fragment} used to safely store an {@link
 com.bumptech.glide.RequestManager} that can be used to start, stop and manage Glide requests
 started for targets the fragment or activity this fragment is a child of.
 
通过文档可知，Glide 借助 Fragment 来跟踪 Activity 的生命周期，以达到启动，停止和管理请求的目的。因为系统并没有提供注册 Activity 生命周期监听器的接口，而要开发者手动在 Activity 的生命周期方法里去管理图片的加载，势必会使代码逻辑混乱，不易维护，因此 Glide 将 Fragment 嵌入 Activity，借助 Fragment 的生命周期来驱动和管理图片加载，从而减少开发者的工作。
 
### load()
前面我们分析了 with() 方法的内部细节，现在我们来看 load() 方法，load() 定义在 ReqeustManager 中：

```java
public RequestBuilder<Drawable> load(@Nullable Object model) {
  return asDrawable().load(model);
}

public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class).transition(new DrawableTransitionOptions());
}

public <ResourceType> RequestBuilder<ResourceType> as(Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass);
}
```

RequestBuilder#load()：

```java
public RequestBuilder<TranscodeType> load(@Nullable Object model) {
  return loadGeneric(model);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```
model 一般情况下就是图片的 URL。总的来说，load() 调用栈内做的事很简单，就是构造出RequestBuilder。而 RequestBuilder 是用来收集配置参数并构造出 Request 的。关于它内部的细节，这里暂时不做深入分析，我们接着往下看。

### into()

into() 是 RequestBuilder 的一个方法，其调用栈比较长，接受的是一个 ImageView 对象：

```java
public Target<TranscodeType> into(ImageView view) {
  ......
  return into(context.buildImageViewTarget(view, transcodeClass));
}

// buildIamgeViewTarget() 实际上会调用 ImageViewTargetFactory 的 buildTarget() 方法：
public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
  if (Bitmap.class.equals(clazz)) {
    return (Target<Z>) new BitmapImageViewTarget(view);
  } else if (Drawable.class.isAssignableFrom(clazz)) {
    return (Target<Z>) new DrawableImageViewTarget(view);
  } else {......}
}
```

它把工作交给了它的一个重载方法：

```java
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
  ......
  Request previous = target.getRequest();
  if (previous != null) {
    requestManager.clear(target);
  }
  requestOptions.lock();
  Request request = buildRequest(target);
  target.setRequest(request);
  requestManager.track(target, request);
  return target;
}
```

真正处理加载请求的方法是 RequestManager 的 track() 方法：

```java
void track(Target<?> target, Request request) {
  targetTracker.track(target);
  requestTracker.runRequest(request);
}
```
这里包含两个方法调用，其中 track() 方法将这个 Target 添加进了一个 Set 集合中，使得这些 target 可以接受生命周期回调，关于这一块的稍后会有更详细的分析，这里暂时不谈。而 runRequest(Reqeust) 触发了加载请求：

```java
public void runRequest(Request request) {
  requests.add(request);
  if (!isPaused) {
    request.begin(); // 执行请求
  } else {
    pendingRequests.add(request);
  }
}
```
这个方法的逻辑很清晰：首先将 request 加入请求列表 requests 中，这个列表保存的是待发起的请求；然后判断是否需要暂停加载（受生命周期的控制），如果是，则开始请求，否则将这个请求放入待处理队列中。

Request 是一个接口，它包含多了很多控制方法和状态查询方法。Request 在当前 Glide 版本中只有一个实现者：SingleRequest。SingleRequest 的 begin() 方法如下：

```java
public void begin() {
  ......
  if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    onSizeReady(overrideWidth, overrideHeight);
  } else {
    target.getSize(this); 
  }
  if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
      && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
  }
  ......
}
```

这段代码的重点是 onSizeReady()，它会调用 Engine#load() 方法来执行图片资源的加载。它的工作流程是：

构建出标识这个请求的 key，并根据这个 key 从内存缓存中获取资源并返回；如果内存中不存在该资源，就根据该 key 从当前正被使用资源中获取并返回；如果还是不存在，那么看是否存在正在请求该资源的后台任务，如果存在该任务，就向该任务中添加一个回调，当后台任务获取资源时就会通过该回调将资源传给请求者，这避免了重复发送请求；如果不存在该任务，没办法，只能发起新的任务来获取该资源。

开始的时候我有这样的疑问，为什么从缓存获取资源只是从内存获取而没有考虑磁盘缓存？其实，Glide 确实考虑了磁盘缓存，只是因为从磁盘获取和从网络获取都是耗时的操作，因此统一将它们放在后台任务中去执行了，后台任务是通过 EngineJob#start(DecodeJob) 方法开启的，其中 DecodeJob 是后台任务单元。

```java
public void start(DecodeJob<R> decodeJob) {
  this.decodeJob = decodeJob;
  GlideExecutor executor = decodeJob.willDecodeFromCache()
      ? diskCacheExecutor
      : getActiveSourceExecutor();
  executor.execute(decodeJob);
}
```

GlideExecutor 是一个线程池，start 方法会将 decodeJob 投入线程池中去执行。DecodeJob 的入口方法 run 的定义如下：

```java
public void run() {
   ......
  try {
    if (isCancelled) {
      notifyFailed();
      return;
    }
    runWrapped();
  } catch (RuntimeException e) {
    ......
  } finally {
    ......
  }
}
```
其执行逻辑是：判断任务状态，如果任务被取消了，那么就发出任务失败的通知；否则执行 runWrapper() 方法：

```java
private void runWrapped() {
   switch (runReason) {
    case INITIALIZE:
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}
```

runGenerators()：

```java
private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled && currentGenerator != null
      && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();
    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  // We've run out of stages and generators, give up.
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }
  // Otherwise a generator started a new load and we expect to be called back in
  // onDataFetcherReady.
}
```

DecodeJob 里面维护了一个状态机，而 runGenerators() 方法就是在获取不同状态下的 DataFetcherGenerator 对象，然后根据此对象的 startNext() 方法来获取相应的资源。getNextState() 方法定义了状态转移条件，它会根据磁盘缓存策略来决定下一个状态：

```java
private Stage getNextStage(Stage current) {
  switch (current) {
    case INITIALIZE:
      return diskCacheStrategy.decodeCachedResource()
          ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
    case RESOURCE_CACHE:
      return diskCacheStrategy.decodeCachedData()
          ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
    case DATA_CACHE:
      // Skip loading from source if the user opted to only retrieve the resource from cache.
      return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
    case SOURCE:
    case FINISHED:
      return Stage.FINISHED;
    default:
      throw new IllegalArgumentException("Unrecognized stage: " + current);
  }
}
```
不同状态下获取到的 DataFetcherGenerator 对象通过 getNextGenerator() 指定：

```java
private DataFetcherGenerator getNextGenerator() {
  switch (stage) {
    case RESOURCE_CACHE:
      return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
      return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
      return new SourceGenerator(decodeHelper, this);
    case FINISHED:
      return null;
    default:
      throw new IllegalStateException("Unrecognized stage: " + stage);
  }
}
```
其中 ResourceCacheGenerator 和 DataCacheGenerator 的资源获取方式都是从磁盘缓存中获取，而 SourceGenerator 获取资源的方式则是从资源的初始位置获取，比如网络。这里我们只分析 SourceGenerator 的 startNext() 方法：

```java
@Override
public boolean startNext() {
  ......
  loadData = null;
  boolean started = false;
  while (!started && hasNextModelLoader()) {
    loadData = helper.getLoadData().get(loadDataListIndex++);
    if (loadData != null
        && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
        || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
      started = true;
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}
```

loadData 的类型是 LoadData，它是一个定义在 ModelLoader 接口内部的类，由此可见它应该和 ModelLoader 有非常紧密的联系。ModelLoader 相当于一个转换器，它将 model 转换成 resource，这个 model 和前面 load() 方法传入的 model 是同一个对象。model 和 resource 都可以是任意类型的，它们的类型分别由类型参数 Model 和 Data 指定：

```java
public interface ModelLoader<Model, Data> {
  @Nullable
  LoadData<Data> buildLoadData(Model model, int width, int height, Options options);
  boolean handles(Model model);
}
```

LoadData 定义如下：

```java
class LoadData<Data> {
  public final Key sourceKey;
  public final List<Key> alternateKeys;
  public final DataFetcher<Data> fetcher;
  ......
}
```
ModelLoader 的工作原理是：通过 model 构造一个 LoadData，然后通过 LoadData 中的 fetcher:DataFetcher 对象加载和 model 相关的资源。DataFetcher 是一个接口，它有很多实现类。一般来说，ModelLoader 的实现类和 DataFetcher 的实现类是成对出现的，通常会吧 DataFetcher 实现类嵌套在对应的 ModelLoader 实现类内部。关于 ModelLoader 和 DataFetcher 的更多细节，我们后面在分析。

假设 model 为 http 协议的 URI 字符串的话，最终就会用到 HttpGlideUriLoader 和 HttpUrlFetcher 这两个类：HttpGlideUriLoader 根据 model 构造出一个 LoadData 对象，该对象的 fetcher:DataFetcher 为 HttpUrlFetcher 字段；SourceGenerator#startNext() 方法中，调用 DataFetcher#loadData() 执行资源加载动作。现在来看看 HttpUrlFetcher 是如何加载资源的：

```java
@Override
public void loadData(Priority priority, DataCallback<? super InputStream> callback) {
  long startTime = LogTime.getLogTime();
  final InputStream result;
  try {
    result = loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/,
        glideUrl.getHeaders());
  } catch (IOException e) {
    ......
    callback.onLoadFailed(e);
    return;
  }
  ......
  callback.onDataReady(result);
}
```
逻辑很清晰：调用 loadDataWithRedirects() 方法获取资源的输入流，如果这个过程发生异常，那么回调 onLoadFail() 方法；如果成功，那么回调 onDataReady() 方法将输入流传给上层。loadDataWithRedirects() 方法如下：

```java
private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
    Map<String, String> headers) throws IOException {
  .....
  urlConnection = connectionFactory.build(url);
  for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
    urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
  }
  urlConnection.setConnectTimeout(timeout);
  urlConnection.setReadTimeout(timeout);
  urlConnection.setUseCaches(false);
  urlConnection.setDoInput(true);
  ......
  urlConnection.setInstanceFollowRedirects(false);
  // Connect explicitly to avoid errors in decoders if connection fails.
  urlConnection.connect();
  if (isCancelled) {
    return null;
  }
  final int statusCode = urlConnection.getResponseCode();
  if (statusCode / 100 == 2) {
    return getStreamForSuccessfulRequest(urlConnection);
  } else if (statusCode / 100 == 3) {
    String redirectUrlString = urlConnection.getHeaderField("Location");
    if (TextUtils.isEmpty(redirectUrlString)) {
      throw new HttpException("Received empty or null redirect url");
    }
    URL redirectUrl = new URL(url, redirectUrlString);
    return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
  } else if (statusCode == -1) {
    throw new HttpException(statusCode);
  } else {
    throw new HttpException(urlConnection.getResponseMessage(), statusCode);
  }
}
```
这里就是 Glide 和网络交互的地方，Glide 通过 HttpUrlConnection 进行网络连接，并且对可能存在的重定向情况进行了处理，最终 Glide 会将获取到的输入流解码成客户端期望的资源。

## 总结

Glide 的加载逻辑还是很清晰的，下面我们从头来梳理一下 Glide 是如何将一个 URL 转换成一张图片的：Glide 先会收集各种必要的配置信息将请求构造成一个 Request 对象，然后将该 Request 对象进一步包装成后台任务并将其投入线程池中运行。运行过程中会获取所有能够处理该 model 的 ModelLoader，通过该 ModelLoader 构造出 LoadData 对象，再用 LoadData 的 DataFetcher 对象去加载 model 对应的资源的二进制流，解码后就得到了最终所期望的资源。Glide 加载的大致流程就是这样的，接下来还会对它的一些细节作更深入的分析。