---
layout:     post
title:      "安卓中 Service 的启动过程（上）"
subtitle:   "Process of starting a Service in android"
date:       2016-12-15
author:     "lwenkun"
catalog:  true
header-img: "img/post-bg-android-how-a-service-in-a-android-app-starts.jpg"
tags:
    - Android
    - 四大组件
---

# Service 的启动过程（上）#

作为四大组件的 `Service`，由于没有 UI 界面，只能默默无闻地在后台工作。虽然我们说他是后台工作的，但是他还是默认进程的主线程中运行的，除非我们给它指定了单独的进程。`Service` 的启动过程有两种，一种是 `startService()`，另一种是 `bindService()`。我会在接下来的两篇文章中分别来介绍着两种启动方式，首先我们来探究 `startService()` 启动服务的过程。

## 通过 startService 启动一个 Service ##
`Activity` 和 `Service` 都是从 `ContextWrapper` 继承而来，而 `ContextWrapper` 继承自 `Context` 这个抽象类，所以我们通常把 `Activity` 和 `Service` 看作是 `Context` 对象。`ContextWrapper` 的实现用的是典型的装饰者模式，它的一系列的方法都靠内部的被装饰者的同名方法来实现，这和代理模式有点类似。而这个被装饰者通常是一个 `ContextImpl`，它是 `Context` 的真正实现者。

`startService()` 是 `Context` 中定义的一个方法，由于 `Activity` 和 `Service` 属于 `ContextWrapper` 类型，所以这个方法真正的实现在 `ContextImpl#startService()` 中：

```java
@Override
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, mUser);
}
```
`startService()` 会调用 `startServiceCommon()` 方法：

```java
private ComponentName startServiceCommon(Intent service, UserHandle user) {
    try {
        ......
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), getOpPackageName(), user.getIdentifier());   
        ......
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
`ActivityManagerNative.getDefault()` 返回的是一个 `ActivityManagerProxy` 对象，它作为 AMS 在本进程的代理，如果我们的应用程序要和 AMS 打交道，必须要以它为媒介。这样看来 `ActvitiyManagerService` 并不像它名字所暗示的那样只管理 `Activity`，`Servcie` 也同样归它管。显然，这是一个远程调用，具体的实现在 `ActivityManagerService#startService()` 中：

```java
@Override
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, String callingPackage, int userId)
        throws TransactionTooLargeException {
    ......
    synchronized(this) {
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid, callingPackage, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```
这个方法并没有做什么，它把事情交给了 `mService`， `mService` 是一个 `ActiveServices` 对象。在早期的安卓版本中并没有这个类，后来重构时抽出这个类专门用来管理 `Service`，`ActiveServices#startServiceLocked()` 有点长，我们挑重要部分来看：

```java 
ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false);
......
ServiceRecord r = res.record;
......
return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
```
这个方法根据这个 `Service` 对应的 `Intent` 解析出一个 `ServiceRecord`，然后把事情交给了 `startServiceInnerLocked()`，这个方法的核心就是调用了 `bringUpServiceLocked()`，因此我们进入 `ActiveService#bringUpServiceLocked()` 看看：

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
        
        ......
        
        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;
        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            ......
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }
             }
        } else {
            app = r.isolatedProc;
        }
        
        ......
        
        if (app == null && !permissionsReviewRequired) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                ......
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
      
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }
        ......
 }
```
这段代码中有一个分支：

- 如果 `Service` 要运行在启动服务的进程中（默认情况），就直接转到 `realStartServiceLocked()` 方法；
- 如果 `Service` 需要运行在指定进程（注册时给 `Service` 指定进程名）中就会通知 AMS 新开一个进程（如果这个进程不存在的话），并把这个 `Service` 对应的 `ServiceRecord` 对象放进等待列表 `mPendingServices` 中，当进程开启成功后会从列表中取出 `ServiceRecord` 对象，然后把 `ServiceRecord` 对应的 `Service` 在该进程中创建并运行。

下面我们分别对这两种情况进行研究。

### 在启动服务的进程中创建 Service ###

前面说了，如果在启动服务的进程中启动 `Service` 会进入到 `ActiveService#realStartServiceLocked()` 方法中，它有两行代码值得关注：

##### 第一行代码 #####

```java
app.thread.scheduleCreateService(r, r.serviceInfo,
       mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
       app.repProcState);
```
关于这行代码说明两点：

- `app` 是要运行 `Service` 的进程对应的 `ProcessRecord` 对象，系统源码很多地方用 `ProcessRecord` 代表一个应用进程，这和用 `ActivityRecord` 代表一个 `Activity` 以及用 `ServiceRecord` 代表一个 `Service` 是同样的道理。
- `thread` 是一个 `ApplicationThreadProxy` 对象，它是应用进程的 `ApplicatonThread` 对象在 AMS 端的代理，AMS 靠它来和应用进程进行通信。你可能会感到奇怪，前面不是说了应用进程和 AMS 通信靠的是 `ActivityManagerProxy` 吗，这里怎么出来了一个 `ApplicationThreadProxy` ？我们要清楚，`Binder` 实现的进程间通信是单向的，其信息传递方向是 `BinderProxy` —> `Binder`。但是很显然，应用进程和 AMS 是需要双向通信的，所以要想实现双向通信，必须有两对 `Binder`-`BinderProxy` 才行，这就是 `ApplicationThread` 和 `ApplicationThreadProxy` 存在的的原因。应用进程和 AMS 的通信可以用下图来表示：

![](/img/in-post/post_android_process_of_starting_a_service/post-1.png)

虽然在这里是 `ActiveServices` 与应用进程通信，但 `ActiveServices` 也是用来辅助 AMS 管理 `Service` 的，所以也可以把这个过程看作是 AMS 与 应用进程的通信。（要知道早期的安卓版本没有 `ActiveServices` 这个类，这些逻辑都是在 AMS 中的）

这行代码的作用是通知应用进程根据已知信息创建一个 `Service`，那么应用进程是怎样创建这个 `Service` 的呢？进入到 `ApplicationThread#scheduleCreateService()`：

```java
public final void scheduleCreateService(IBinder token,
        ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
    updateProcessState(processState, false);
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;
    sendMessage(H.CREATE_SERVICE, s);
}
```

在 `scheduleCreateService()` 中，先是把 AMS 传来的信息封装成一个 `CreateServcieData` 对象，然后调用 `sendMessage()` 把信息发送出去。注意，`sendMessage()` 是 `ActivitytThead` 的方法，因为 `ApplicationThread` 是 `ActivityThread` 的内部类，所以对 `ActivityThread` 有完全的访问权限。这样一来消息就从 `ApplicationThread` 传到了 `ActivityThread`，我们来看看 `ActivityThread#sendMessage()`：

```java
private void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}
```
转到另一个重载方法：

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```
这里把信息传给了 `mH` 这个 `Handler` 对象，这个 `Handler` 对象是在应用进程的主线程中创建的，所以最终的结果是把创建 `Service` 的消息传到了主线程。现在你终于明白了为什么 `Service` 会在主线程运行吧？看看 `mH` 是怎样处理这个消息的：

```java
case CREATE_SERVICE:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
    handleCreateService((CreateServiceData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```

百转千回，消息最终传到了 `handleCreateService()`：

```java
private void handleCreateService(CreateServiceData data) {
    ......
    
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
    } catch (Exception e) {
        ......
    }
    try {
        if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
        mServices.put(data.token, service);
        try {
            ActivityManagerNative.getDefault().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        ......
    }
}
```
这里是 `Service` 真正被创建的地方，这个方法做了以下几件事：

- 根据类名用应用程序的类加载器加载并实例化一个 `Service`。
- 创建一个 `ContextImpl` 对象，把 `Service` 作为它的额外层 `Context` 对象。
- 为这个 `Service` 创建一个 `Application` 对象，当然，如果应用进程已经存在 `Application` 就不会重复创建，具体大家可以看看 `LoadedAPK#makeApplication()` 的实现，这里就不详细讲了。
- 用 `Service` 的 `attach()` 方法把相应信息附加到 `Service`，其中，`context` 是之前创建的 `ContextImpl` 对象，在 `attach()` 方法中会把它作为 `Service` 内部的被装饰对象；`this` 代表本进程中的 `ActivityThread` 对象；`data.info.name` 是这个 `Service` 的名称；`data.token` 是从 AMS 传过来的 `ServiceRecord` 对象，它是一个 `Binder`，作为这个 `Service` 的标识。
- 调用 `Service` 的 `onCreate()` 方法。
- 把 `token` 和 `Service` 的映射关系保存在 `mServices` 中。
- 通过 `ActivityManagerProxy` 告知 AMS `Service` 已经创建好了，让其完成后续的工作。

关于后续 AMS 做了哪些事，我们就不深究了，大家有兴趣可以自行阅读源码。现在我们看看 `ActiveService#realStartServiceLocked()` 中的另一行代码。

#### 第二行代码 ####

```java
sendServiceArgsLocked(r, execInFg, true);
```

这个方法的核心部分在这里：

```java
r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
```
这里又通过 `ApplicationThreadProxy` 和应用进程进行了通信，我们看看应用进程是怎样响应这个方法调用的：

```java
public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
    int flags ,Intent args) {
    ServiceArgsData s = new ServiceArgsData();
    s.token = token;
    s.taskRemoved = taskRemoved;
    s.startId = startId;
    s.flags = flags;
    s.args = args;
    sendMessage(H.SERVICE_ARGS, s);
}
```
和前面一样，同样是把信息封装好后通过安卓的消息机制投递到主线程中，我们看看 `Handler:mH` 是怎样处理这个消息的：

```java
case SERVICE_ARGS:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
    handleServiceArgs((ServiceArgsData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```
转入 `ActivityThread#handleServiceArgs`：

```java
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            ......
            int res;
            if (!data.taskRemoved) {
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }
            QueuedWork.waitToFinish();
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
            ensureJitEnabled();
        } catch (Exception e) {
            ......
        }
    }
}
```
这里先根据 `token` 取出保存的 `Service`，然后根据 `data.taskRemoved` 的值回调 `Service` 中的相应方法，一般情况下这个值是为 `false` 的，也就是说 `Service` 的 `onStartCommand()` 方法会得到调用。至此在服务启动者的进程中创建服务的过程就分析完了，现在我们看看指定进程中启动服务的过程是怎样的。

### 在指定进程中创建并启动 Service 的过程 ###

如果注册 `Service` 的时候我们给 `Service` 指定了进程名，那么 `Service` 就会在那个进程中被创建并运行，这个时候启动过程就会走向另一条分支，还记得出现分支的地方吗？它在 `ActiveService#bringUpServiceLocked()` 中，这条分支会执行以下代码：

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
        ......        
        if (app == null && !permissionsReviewRequired) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                ......
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
      
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }
        ......
}
```
注意到这行代码 `mAm.startProcessLocked()`，`mAm` 就是 AMS，这里通过 AMS 开启了一个进程。现在我们进入 `ActivityManagerService#startProcessLocked()` 这个方法：

```java
private final void startProcessLocked(ProcessRecord app,
        String hostingType, String hostingNameStr) {
    startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
            null /* entryPoint */, null /* entryPointArgs */);
}
```
转向它的一个重载方法，这个重载方法有点长，但它的核心代码也就一句话：

```java
Process.ProcessStartResult startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, debugFlags, mountExternal,
        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
        app.info.dataDir, entryPointArgs);
```
`Process#start()` 会利用这些参数向 native 层 fork 一个进程。注意到这个方法的第一参数 `String:entryPoint`，顾名思义，它是进程的入口点，其值为 `"android.app.ActivityThread"`。对于 `ActivityThread` 我们应该很熟悉了，每个应用进程都有一个 `ActivityThread` 对象，它代表着应用进程的主线程，处理着内部类 `ApplicationThread` 发送过来的来自 AMS 的各种消息。进程创建好后 native 层代码后就会调用这个类的静态方法 `main()`，它的实现如下：

```java
public static void main(String[] args) {
    ......
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
    Process.setArgV0("<pre-initialized>");
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
它很简短，主要做了这几件事：

- 为主线程准备消息循环
- 为应用进程创建一个 `ActivityThread` 对象
- 调用 `ActivityThread` 的 `attach()` 方法
- 开启消息循环

这样我们的应用进程就进入到一个死循环中了，不断的接受消息并执行。所以我们会说安卓应用是消息驱动的。我们重点关注 `attach()` 方法，它有这样一段代码值得关注：

```java
final IActivityManager mgr = ActivityManagerNative.getDefault();
try {
    mgr.attachApplication(mAppThread);
} catch (RemoteException ex) {
    throw ex.rethrowFromSystemServer();
}
```
这里通过 `ActivityManagerProxy` 远程调用了 AMS 的 `attachApplication()` 方法，参数是 `mAppThread`，而它是这样初始化的：

```java
final ApplicationThread mAppThread = new ApplicationThread();
```
说明它就是 `ApplicationThread` 对象，前面说了，AMS 要想向应用进程发送消息，需要借助 `ApplicationThreadProxy` 对象。而通过 `Binder` 机制 `ApplicationThread` 在 AMS 那边就转化成了 `ApplicationThreadProxy` 对象，所以这个对象就是在此时传给 AMS 的。在 AMS 中，`attachApplication()` 会直接调用 `attachApplicationLocked()`，对于这个方法，我们挑需要的代码段来看：

```java
if (!badApp) {
    try {
        didSomething |= mServices.attachApplicationLocked(app, processName);
    } catch (Exception e) {
        Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
        badApp = true;
    }
}
```
`mServices` 就是 `ActiveServices`，进入 `ActiveServices#attachApplicationLocked()`：

```java
boolean attachApplicationLocked(ProcessRecord proc, String processName)
        throws RemoteException {
    boolean didSomething = false;
    // Collect any services that are waiting for this process to come up.
    if (mPendingServices.size() > 0) {
        ServiceRecord sr = null;
        try {
            for (int i=0; i<mPendingServices.size(); i++) {
                sr = mPendingServices.get(i);
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                mPendingServices.remove(i);
                i--;
                proc.addPackage(sr.appInfo.packageName, sr.appInfo.versionCode,
                        mAm.mProcessStats);
                realStartServiceLocked(sr, proc, sr.createdFromFg);
                didSomething = true;
                if (!isServiceNeeded(sr, false, false)) {
                    bringDownServiceLocked(sr);
                }
            }
        } catch (RemoteException e) {
           ......
        }
    }
    ......
}
```
大家还记得 `mPendingServices` 吗？前面说了，如果在指定进程当中运行一个 `Service`，会先创建一个进程，然后在把该 `Service` 对应的 `ServiceRecord` 对象放入 `mPendingServices` 中，待进程创建好了就会从中取出 `ServiceRecord`，然后根据它在进程中创建 `Service`。具体过程是怎样的呢？我们看看这个方法主要作了哪些事：

遍历 `mPendingServices`，根据 `ProcessRecord:proc` 和 `String:processName` 提供的进程信息找出要运行在这个进程的 `ServiceRecord`，然后调用 `realStartServiceLocked()` 方法并把找到的 `ServiceRecord` 作为参数传入其中。`realStartServiceLocked()` 之后的流程前面已经有介绍，这里就不重复讲了。

至此，通过 `startService()` 启动服务的整个过程就介绍完了，在接下来的一篇文章中我会介绍通过 `bindService()` 创建并绑定一个 `Service` 的详细流程。感谢大家的阅读，有什么不对的地方还望大家不吝赐教。

