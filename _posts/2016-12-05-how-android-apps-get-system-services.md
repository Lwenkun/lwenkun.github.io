---
layout:     post
title:      "安卓应用获取系统服务的过程"
subtitle:   "how android apps get system services"
date:       2016-12-05
author:     "lwenkun"
header-img: "img/post-bg-android-how-apps-get-system-services.jpg"
tags:
- android
- system service
- IPC
- Binder
---


# 安卓应用获取系统服务的过程 #

在安卓系统中存在着各种各样的系统服务， 例如 [`ActivityManagerService`](http://androidxref.com/7.0.0_r1/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java), [`WindowManagerService`](http://androidxref.com/7.0.0_r1/xref/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java), [`ClipboardService`](http://androidxref.com/7.0.0_r1/xref/frameworks/base/services/core/java/com/android/server/clipboard/ClipboardService.java) 等。这些系统服务大都运行在单独的进程中，为每个应用提供服务。而应用程序运行于自己的默认进程当中，因此，想要获取系统服务，必定需要进行进程间的通信。而安卓中的进程间通信大多通过 `Binder` 机制进行，因此，要想深入理解应用是如何获取的系统服务的，必须先要了解安卓的 `Binder` 机制。如果对 `Binder` 机制不太了解的同学可以参考我的[这篇博客](http://liwenkun.xyz/2016/10/28/android-IPC-AIDL/)。

安卓中的系统服务作用各不相同，所以，如果对每个服务的内部细节做过多的探讨，不利于我们从宏观的角度去理解安卓的系统服务的大致原理。如果你想知道各个服务的具体实现，那么这篇文章并不适合你。

## Context#getSystemService ##

大家对这方法应该很熟悉了，开发应用的时候我们就是通过这个接口获取系统服务的。系统设计人员总是想为应用开发者提供各种便利，减轻他们的负担。以至于对于开发者来说，系统服务的获取简单得用一行代码就能搞定。那么，我们看看系统到底是如何简化服务的获取过程的。首先从这个方法入手：

```java
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

它的实现也是如此简单，以至于我们必须转入 `SystemServiceRegistry` 一探究竟。

## SystemServiceRegistry ##

源码太多，我就不贴出来了，大家点击[这里](http://androidxref.com/7.0.0_r1/xref/frameworks/base/core/java/android/app/SystemServiceRegistry.java)自己看看。

这个类逻辑很简洁，它最引人注目的地方就是开头的那一大坨静态初始化代码块。这块代码异常整齐，清一色的 `registerService()`：

```java 
......

registerService(Context.WIFI_SERVICE, WifiManager.class,
                new CachedServiceFetcher<WifiManager>() {
    @Override
    public WifiManager createService (ContextImpl ctx){
        IBinder b = ServiceManager.getService(Context.WIFI_SERVICE);
        IWifiManager service = IWifiManager.Stub.asInterface(b);
        return new WifiManager(ctx.getOuterContext(), service,
                ConnectivityThread.getInstanceLooper());
    }
});

registerService(Context.WIFI_P2P_SERVICE, WifiP2pManager.class,
                new StaticServiceFetcher<WifiP2pManager>() {
    @Override
    public WifiP2pManager createService () {
        IBinder b = ServiceManager.getService(Context.WIFI_P2P_SERVICE);
        IWifiP2pManager service = IWifiP2pManager.Stub.asInterface(b);
        return new WifiP2pManager(service);
    }
});

......
```

我们似乎明白，这里注册了一系列的系统服务。而且因为是写在静态代码块中，所以在类加载的时候这些服务就注册了。但是，为什么要对这些系统服务进行注册呢？且看 `registerService()` 这个方法：

```java
/**
 * Statically registers a system service with the context.
 * This method must be called during static initialization only.
 */
private static <T> void registerService(String serviceName, Class<T> serviceClass,
                                       ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```

又一个陌生的类 `ServiceFetcher` 出现了：

```java
static abstract interface ServiceFetcher<T> {
       T getService(ContextImpl ctx); 
}
```

`registerService()` 这个方法并没有将真正的服务注册进去，而是注册了一个 `ServiceFetcher` 对象，这个对象定义了服务获取接口，这样看来， `ServiceFetcher` 相当是一个服务获取策略。所以我们有理由推测，服务的注册实际上注册的不是实际的服务，而是服务的获取策略。当调用 `SystemServiceRegistry.getSystemService()` 获取某个的服务的时候，它会通过根据这个服务的获取策略返回它的一个实例。实际情况是不是这样呢，我们往下看。

`ServiceFetcher` 有三个抽象的实现类，我们只看其中一个：

```java
/**
 * Override this class when the system service constructor needs a
 * ContextImpl and should be cached and retained by that context.
 */
static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
    private final int mCacheIndex;

    public CachedServiceFetcher() {
        mCacheIndex = sServiceCacheSize++;
    }

    @Override
    @SuppressWarnings("unchecked")
    public final T getService(ContextImpl ctx) {
        final Object[] cache = ctx.mServiceCache;
        synchronized (cache) {
            // Fetch or create the service.
            Object service = cache[mCacheIndex];
            if (service == null) {
                service = createService(ctx);
                cache[mCacheIndex] = service;
            }
            return (T)service;
        }
    }

    public abstract T createService(ContextImpl ctx);
}
```

这个抽象类实现了服务获取策略 `getService()`，就算不看这个方法的具体实现，从这个类的名字我们也容易知道，这是一个能够对获取过的服务进行缓存的 `ServiceFetcher`。它将缓存策略封装在 `getService()` 方法中，并且抽象出 `createService()` 这个方法用来创建没有缓存过的对象。`CacheServiceFetcher` 将获取过的服务缓存在 `Context` 的 `mServiceCache` 这个对象中，这个对象是一个 `Object` 类型的数组。你可能会问，为什么选择用数组来缓存，数组的容量是固定的，你怎么知道要缓存多少服务。我们看看 `Context` 的 `mServiceCache`：

```java
final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();
```

又回到了 `SystemServiceRegister`：

```java
public static Object[] createServiceCache() {
     return new Object[sServiceCacheSize];
}
```

结果发现这个数组的大小为 `sServiceCacheSize`，`sServiceCacheSize` 在上面 `CachedServiceFetcher` 的构造函数中进行了自增运算，而 `CachedServiceFetcher` 是在注册的时候创建的，因此我们创注册了多少个 `CachedServiceFetcher`，`cache` 就有多大，因此我们不必担心缓存空间不够用。

貌似有点扯远了，我们回过头分析 `registerService()`，发现它做了两件事：

- 把该服务的 `class` 对象和服务名对应起来
- 把该服务的服务名和获取该服务的策略对应起来

现在我们来看看 `SystemServiceRegistry` 的 `getSystemService()` 方法：

```java
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

果然和我们前面的推断是一样的，获取系统服务时，系统是通过调用某个服务的获取策略来获取这个服务的。前面分析了， `CachedServiceFetcher` 获取服务的策略是先从缓存中找，如果没有就创建一个。那么服务是怎样创建的呢？通过观察那一系列的 `registerService()`，我们发现大多数的服务是这样创建的：

```java
IBinder b = ServiceManager.getService(service_name);
ServiceInterface service = ServiceInterface.Stub.asInterface(b);
return new xxxManager(ctx, ServiceInterface);
```

有没有种似曾相识的感觉？这和我们使用 AIDL 实现进程间通信时客户端的行为是一样的：

- 获取服务端传过来的 `Binder`（对于进程间通信，其实它是一个 `BinderProxy` 对象）
- 将这个 `Binder` 通过 `asInterface()` 转换成相应 AIDL 接口的客户端代理

我们知道，`Binder` 的作用相当于一把钥匙，客户端拿到这个 `Binder` 用 `asInterface()` 将其转换成相应 AIDL 接口的客户端代理后就可以 “随意使唤” 服务端了。只不过在这里是服务接口(如 `IActivityManager`，`IWindowManager`)而不是 AIDL 接口，并且在服务接口的客户端代理上又包装了一层，但实际上还是通过操作代理对象进行进程间通信的。我们发现所有服务的客户端 `Binder`（`BinderProxy`） 都有同一个来源，那就是 `ServiceManager`。接下来就有请 `ServiceManager` 登场。

## ServiceManager ##

不用我过多解释估计你也明白这个类的作用，它就是服务的管理类。安卓的系统服务种类繁多，自然需要一个管理者对它们进行统一的管理，`ServiceManager` 就充当了这样一个角色。那么，它是怎样对那些系统服务进行管理的呢？我们可以看看它的[源码](http://androidxref.com/7.0.0_r1/xref/frameworks/base/core/java/android/os/ServiceManager.java)。源码不多，我们首先看看这个方法：

```java
public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name);
        if (service != null) {
            return service;
        } else {
            return getIServiceManager().getService(name);
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
```

这个方法的作用很明显，就是根据服务名获取一个服务对应的客户端 `Binder`（`BinderProxy`）对象。获取 `Binder` 的过程是先从缓存 `sCache` 中查找是否存在缓存过的 `Binder` 对象，不存在就通过 `getIServiceManager().getService(name)` 来获取并返回。这里我们只对 `sCache` 和 `getIServiceManager()` 感兴趣。先看 `sCache`：

- sCache

通过上面的代码我们发现，当缓存中没有需要的 `Binder`（`BinderProxy`）时，会通过 `getIServiceManager().getService()` 获取，但是我们并没有把获取到的 `Binder`（`BinderProxy`） 进行缓存，这样一来，`sCache` 岂不是一直都是空的？我们看看它是怎样得到的：

```java
public static void initServiceCache(Map<String, IBinder> cache) {
        if (sCache.size() != 0) {
            throw new IllegalStateException("setServiceCache may only be called once");
        }
        sCache.putAll(cache);
}
```

那么我们只要找到 `initServiceCache()` 的调用者就可以知道 `sCache` 是怎样被初始化的。在源码中遨游一小阵子之后发现最终它是这样被初始化的（[ActivityManagerService#getCommonServicesLocked](http://androidxref.com/7.0.0_r1/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#getCommonServicesLocked)）：

```java
private HashMap<String, IBinder> getCommonServicesLocked(boolean isolated) {
    if (mAppBindArgs == null) {
        mAppBindArgs = new HashMap<>();

        // Isolated processes won't get this optimization, so that we don't
        // violate the rules about which services they have access to.
        if (!isolated) {
            // Setup the application init args
            mAppBindArgs.put("package", ServiceManager.getService("package"));
            mAppBindArgs.put("window", ServiceManager.getService("window"));
            mAppBindArgs.put(Context.ALARM_SERVICE,
                    ServiceManager.getService(Context.ALARM_SERVICE));
        }
    }
    return mAppBindArgs;
}
```

这就说明只有 `PackageManagerService`，`WindowManagerService` 和 `AlarmManagerService` 对应的客户端 `Binder`（`BinderProxy`） 被缓存在 `sCache` 中的，其他服务的都是现用现取。至于为什么只对这三个 `BinderProxy` 进行缓存，我也不是太清楚，大家有兴趣可以去研究研究。

- getIServiceManager()

```java
private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
}
```

这是典型的单例模式，并且我们还发现 `sServiceManager` 也是通过 IPC 得到的（源码中 `xxxManagerNative` 其实就相当于 AIDL 中的 `Stub`），说明这个 `ServiceManager` 其实也是一个代理类。也就是说我们获取系统服务的过程，就是通过 `ServiceManager` 这个代理获取其他服务的客户端代理的过程。

OK，安卓获取系统服务的过程就介绍到这里，感谢大家的阅读，有什么不对的地方还望大家不吝赐教。
