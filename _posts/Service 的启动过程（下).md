# Service 的启动过程（下）#
在上一篇文章中我介绍了通过 startService() 启动一个 Servcie 的过程，这篇文章我们就来探究一下 bindService() 是如何启动 Service 的，看看这两种方式有什么异同。


## 通过 bindService() 启动并绑定一个服务 ##
 
 和 startService() 一样

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
            Process.myUserHandle());
}
```

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
        handler, UserHandle user) {
    IServiceConnection sd;
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    } else {
        throw new RuntimeException("Not supported in system context");
    }
    validateServiceIntent(service);
    try {
        IBinder token = getActivityToken();
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        service.prepareToLeaveProcess(this);
        int res = ActivityManagerNative.getDefault().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
bindService() 会调用 bindServiceCommon() ，这个方法的实现分两步：第一步是在本地实现的，它会通过 LoadedApk#getServiceDispatcher() 获取这个 conn 对应的 IServiceConnection 对象； 第二步在服务端 AMS 中实现，向 AMS 发出绑定服务的请求。下面我们分析这两步分别做了哪些事。

### LoadedApk#getServiceDispatcher() ###

我们知道，绑定 Service 时，我们需要实现 ServiceConnection 这个接口，当 Service 绑定或者解绑时会回调其中相应的方法。我们要清楚一点，不管 Service 和我们的发出绑定请求的进程是否为同一进程，绑定这个过程都是需要和系统进行 IPC 通信的（其实就是和 AMS 进行通信），由 AMS 来进行调度。按照一般逻辑，当服务成功绑定后，系统会回调接口中的 onServiceConnected() 方法，但很遗憾，这个接口并不是 Binder 对象，并不能被跨进程回调，因此系统便通过 ServiceDispatcher 这个类辅助实现接口的跨进程回调。ServiceDispatcher 也不是 Binder 类，但是它有一个 Binder 类型的静态内部类 InnnerConnection，这个类的定义如下：

```java
private static class InnerConnection extends IServiceConnection.Stub {
        final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

        InnerConnection(LoadedApk.ServiceDispatcher sd) {
            mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
        }

        public void connected(ComponentName name, IBinder service) throws RemoteException {
            LoadedApk.ServiceDispatcher sd = mDispatcher.get();
            if (sd != null) {
                sd.connected(name, service);
            }
        }
}
```

这个内部类 继承自 IServiceConnection.Stub, 它把外层类做了个简单的包装。当系统绑定好服务后，会跨进程回调这个类的 connected() 方法。这个 connected() 方法具体做了那些事稍后再作分析。我们先看看 getServiceDispatche() 做了什么：

```java
public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                sd = map.get(c);
            }
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (map == null) {
                    map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler);
            }
            return sd.getIServiceConnection();
        }
}
```
这个方法中出现两张映射表，分别是 mServices 和 map。mService 记录着当前 Context 对象（即发出绑定请求的那个 Context, 一般是 Service 或者 Activity）到 map 的映射关系。map 记录着 ServiceConnection 对象到 ServiceDispatcher 的映射关系。这个方法逻辑如下：

- 首先从 mService 中取出当前 Context 对象对应的 map。
- 判断 map 是否为空，如果不为空，那么根据 ServiceConnection:c 取出对应的 ServiceDispatcher:sd。如果取出的 sd 为空，就创建一个 ServiceDispatcher 赋给 sd，然后把 c 和 sd 的对应关系放入 map；如果 sd 不为空就更新 sd。
- 如果 map 为空，先创建一个 ServiceDispatcher 给 sd，再创建一张空表给 map，然后把 c 和 sd 的对应关系放入 map，最后把 context 和 map 的对应关系放入 mServices。
- 最后调用 sd 的 getIServiceConnection() 方法返回一个 IServiceConnection()，这个 IServiceConnection 就是 InnerConnection 对象。

通过这个方法得到的 IServiceConnction 会通过 Binder 机制传至 AMS。

### ActivityManager#bindService() ###

ActivityManagerNative.getDefault() 获取的是 ActivityManager，这个 AMS 在客户端的代理，它的 bindService() 方法会向 AMS 发起一个远程调用。简单说明下这个类的几个重要参数：第一个参数是 ApplicationThread 对象，是一个 Binder 对象，用来和 AMS 通信；第二个参数是 token，如果 Service 的绑定是 Activity 发起的，token 就不为空，否则为空，它用来标识 Activity；第三个是 Intent 对象，告诉 AMS 我们要启动那个服务。这个远程调用的具体实现在 AMS 的同名方法 bindServie() 中，这个方法对参数做了些检查后就直接把任务交给了 ActiveServices:mServices 的 bindServiceLocked() 方法，接下来我们看看 ActiveServices#bindServiceLocked()：

