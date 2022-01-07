---
layout:     post
title:      "Android 进程启动原理"
subtitle:   "Android 进程启动原理"
catalog:  true
date:       2017-08-29
author:     "lwenkun"
header-img: "img/post-bg-android-event-dispatch.png" 
tags:
    - Android
    - 进程
---

# Android 进程启动原理

## 前言
Android 中的应用是支持多进程的，我们只要在 AndroidManifest 中给四大组件指定 android:process 属性即可让其运行在独立的进程当中。那么应用的主进程又是如何被创建的呢？一般来说，当我们的应用没有任何组件处于运行状态，此时其他应用启动我们应用的组件时，应用的主进程就会被创建，进程相当于是提供了组件运行的空间。最常见的触发应用主进程被创建的方式是应用 A 通过 startActivity 或者 startActivityForResult 启动应用 B，并且应用 B 此时没有任何组件处于运行状态，那么系统先会通过 AMS 为 B 创建一个进程，等进程创建好了再通知应用 B 创建并运行该 Activity。AMS 是运行在 system_server 进程当中的，这个进程并不会 fork 出应用进程，那么应用进程到底是哪个进程创建的呢？这就得从 Zygote 进程说起。
<!-- more -->
## Zygote 进程

### zygote 进程的启动
Zygote 的中文意思是“受精卵”，它的字面意思很形象的概括了 Zygote 进程在 Android 系统中的作用——“分裂”出其他进程。Zygote 进程是所有 Android 应用的父进程，并且还是 system\_server 的父进程。由此看来，Zygote 进程在 Android 系统中发挥了举足轻重的作用。但是 Zygote 并不是 Android  系统中的第一个进程，它也是其他进程 fork 来的，这个进程就是 init 进程，它是用户空间的第一个进程。init 进程就是用来解析并执行 init.rc 配置文件的，这个文件位于系统根目录下，这个配置文件中会导入其他的配置文件如 init.zygote32.rc 或者 init.zygote64_32.rc，具体导入哪一个由 ro.zygote 的值决定。这两个配置文件的作用就是启动 Zygote 进程。在 init.zygote32.rc 中，启动 Zygote 的语句如下：

```
serivce zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	class main
	socket zygote stream 660 root system
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
	onrestart restart netd
```

我们主要看 service 块的第一行，这行命令的语义是：启动一个名为 zygote 的服务，这个服务的可执行文件路径是 /system/bin/app\_process。后面跟的都是 app\_process 这个可执行文件的参数。app\_process 对应的源码文件为你 app\_main.c，这个源码文件的 main() 方法中有对这些参数的解释：以 - 开头的都是虚拟机参数，这些参数会传入即将要启动的虚拟机中；接下来是运行目录，有点类似终端执行命令时的当前目录，在这里是 /system/bin；接下来是一些以 -- 开头的参数，这些参数表示启动参数，“--zygote” 表示启动的是 zygote 进程，“--application”表示启动的是普通 Java 进程，“--start-system-server” 表示要启动 system_server 进程，“--nice-name”用来指定进程的名字。接下来的参数指明的是要启动的 Java 类，如果前面使用了 “--zygote“参数，那么启动的就是 zygote 进程，这里就可以不指明，因为 zygote 进程默认就是执行 ZygoteInit 类，如果使用了 “--application”，那么启动的就是普通的 Java 进程，这里需要指明要执行的 Java 主类；接下来的参数是都是传入该主类的参数，这些参数只有启动的是普通 Java 进程时才有用。

app_main.c 的 main() 方法的主要逻辑都是在解析相应的参数，它对参数的的解析主要分为以下几个步骤：

- 创建 AppRuntime 并解析虚拟机参数：

```c
int main(int argc, char* const argv[])
{
    ......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i])); //将有效参数加入到虚拟机参数表中
    }
    ......
}
```

- 解析启动参数

```java
int main(int argc, char* const argv[]) 
{	
	......
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg); // 解析到了主类名，说明启动的是普通 Java 程序
            break;
        } else {
            --i;
            break;
        }
    }
    ......
}
```

- 准备要执行的主类的参数，这里的主类只有“ZygoteInit” 和 “RuntimeInit”两种

```java
int main(int argc, char* const argv[]) 
{	
	......
    Vector<String8> args;
    if (!className.isEmpty()) {
    	 // 非 zygote 模式，我们需要将后面的参数传入 AppRuntime 中保存，而不是传入 RuntimeInit 中
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // zygote 模式
        maybeCreateDalvikCache();
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);
        // zygote 模式中，将后面的参数全部传入 ZygoteInit 的 main() 方法中
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
    ......
}
```

- 修改进程名

```java
int main(int argc, char* const argv[]) 
{	
	......
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }
    ......
}
```

- 启动 “ZygoteInit” 或者 “RuntimeInit” 类

```java
int main(int argc, char* const argv[]) 
{	
	......
    if (zygote) { // 如果启动的是 zygote 进程
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) { // 如果启动的是普通 Java 进程
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else { // 否则报错
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
    ......
}
```
start() 方法最终会启动虚拟机并执行上面传入的类。这里可能有点奇怪，说好的启动普通 Java 进程时启动的是指定的 Java 主类，为什么这里却是执行 RuntimeInit 呢？其实逻辑是这样的：先执行 RuntimeInit 做一些必要的设置，再回到 AppRuntime 中执行指定的 Java 主类。这也就为什么在第三步的时候普通 Java 类的类名和参数要通过 AppRuntime#setClassNameAndArgs 的成员变量中，而 ZygoteInit 的参数保存在 args:Vector<String8> 中。因为这样从 RuntimeInit 回到 AppRuntime 中时就可以根据之前保存的类名和参数去执行指定的 Java 主类了。更详细的解析需要通过阅读 AppRuntime 的源码来说明。

### AndroidRuntime

AppRuntime 继承自 AndroidRuntime，它的大部分方法都直接继承自 AndroidRuntime，它仅仅是实现了 AndroidRuntime 定义的一些生命周期回调方法。所以我们重点来看 AndroidRuntime，首先看看它的构造方法：

```c++
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t BlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
    SkGraphics::Init();
    // mOptions 用来存储虚拟机参数
    mOptions.setCapacity(20);

    assert(gCurRuntime == NULL);        // 确保只有一个实例
    gCurRuntime = this; // 将该实例的指针保存在 gCurRuntime 这个全局指针中
}
```

在 app_main.c 的 main() 方法中我们看到过 AppRuntime 的实例化过程，它是作为一个局部变量创建的，在 C++ 中，局部变量是在栈上分配的，而栈上分配的变量有一个特点，就是在方法执行完后它对应的内存就会被自动释放掉，不管是否有指向该变量的引用，所以一般不让全局的指针指向局部变量，这一点和 Java 有所不同。因此这里会有一个问题：如果 main() 方法执行完了，那么 gCurRuntime 所指向的那块内存已经被释放掉了，那么当我们引用 gCurRuntime 时程序肯定会报错。的确，如果 main() 运行完毕了我们在通过引用 gCurRuntime 去调用 AppRuntime 实例的方法肯定会报错，但关键在于，main() 方法退出了程序也就退出了，程序根本没有再引用 gCurRuntime 的机会了，这样看来，是不用担心 gCurRuntime 指向已释放内存的，在程序中可以放心使用。

现在来看看它的一个重要方法，也是上面用到过的方法——start()，这个方法主要做了以下几件事：

- 启动虚拟机：

```c++
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
JNIEnv* env;
if (startVm(&mJavaVM, &env, zygote) != 0) {
    return;
}
```

这里创建了虚拟机并且导出了 JNIEnv 对象，这个对象可以用来和 Java 层交互。

- 回调 onVmCreated(env) 方法

```java
virtual void onVmCreated(JNIEnv* env)
{
    if (mClassName.isEmpty()) {
        return; // Zygote. Nothing to do here.
    }
    // 将类名中的 . 替换成 /
    char* slashClassName = toSlashClassName(mClassName.string());
    mClass = env->FindClass(slashClassName); // 查找指定的类
    if (mClass == NULL) { // 找不到这个类
        ALOGE("ERROR: could not find class '%s'\n", mClassName.string());
    }
    free(slashClassName);
    mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass));
}
```
之前说了，AppRuntime 实现了 AndroidRuntime 中定义的各种生命周期回调方法，onVmCreated() 方法就是其中之一，因此这个方法的实现在 AppRuntime 中。这里对启动的是 zygote 进程还是普通的 Java 进程进行了不同的处理，如果是  zygote 进程，那么什么也不干，如果是普通的 Java 进程，那么就根据之前保存的类名来加载这个类。

- 注册 JNI 函数

```java
if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
}
```

这一步会将很多 Java 层的 native 方法和 C++ 层的方法对应起来。

- 准备 Java 类的参数

```java
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
```
这里创建了一个 Java 字符串数组对象，然后把 C++ 字符串转换成 Java 层的字符串对象后放进该数组中。除了原本在 options 中的字符串外，这里还会把要启动的类的完整类名作为第一个元素放进字符串数组中。

- 启动 Java 类

```java
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        // 找到 main 方法
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            // 执行 main() 方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
			......
        }
    }
    free(slashClassName);
    // 销毁虚拟机
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
```
这里就是启动 Java 类的地方了，根据前面的分析我们知道，这里的 Java 类只有 ZygoteInit 和 RuntimeInit 两种情况。我们先考虑启动的是 zygote 进程的情况，因此我们接下来分析一下 ZygoteInit 这个类。

### ZygoteInit
先看它的 main() 方法：

```java
public static void main(String argv[]) {
    ......
    try {
        ......
        boolean startSystemServer = false;
        String socketName = "zygote";
        String abiList = null;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                socketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }
        ......
        registerZygoteSocket(socketName);
        ......
        preload(); // 预加载系统资源
        ......        
        // Zygote process unmounts root storage spaces.
        Zygote.nativeUnmountStorageOnInit();
        ZygoteHooks.stopZygoteNoThreadCreation();
        if (startSystemServer) {
            startSystemServer(abiList, socketName);
        }
        Log.i(TAG, "Accepting command socket connections");
        runSelectLoop(abiList);
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run(); // 通过抛出异常的方式清理方法调用栈，从此处执行 zygote 子进程中的逻辑
    } catch (Throwable ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```
解析参数，先是判断是否要启动系统进程，然后获取 abi 列表，最后获取 socket 名称。之后根据 socket 名称注册服务端 socket。接下来就会预加载系统资源，因为应用进程都需要用到系统资源，因此 zygote 提前将这些资源加载好，到时候 fork 出应用进程的时候就可以直接把预加载的系统资源复制过去了，避免每次 fork 后子进程都要重复加载一遍系统资源，提高了效率。之后会决定是否启动 system\_server 进程。启动完 system\_server 之后的 zygote 进程就会进入一个死循环中，它会一直监听 Socket，看有没有客户端进程向自己发起通信。

### Zygote Fork 子进程
我们通过 startActivity() 或者 startActivityForResult() 来启动另一个应用程序时，会通过 Binder 向 AMS 发动请求，AMS 接收到请求后会构造一个 ActivityRecord 对象，这个对象包含要启动的 Activity 的各种信息。之后 AMS 会判断时候 Activity 需要运行的进程是否存在，如果不存在，那么通过 socket 请求 zygote 进程 fork 出一个子进程，再将 ActivityRecord 通过 ApplicationThread 传给那个进程，让它根据这个 ActivityRecord 去启动对应的 Activity。在 AMS 中，请求 zygote fork 进程是通过 Process#start() 方法来完成的，这个方法又会调用 startViaZygote() 方法，这个方法准备好参数后会接着调用 zygoteSendArgsAndGetResult() 方法，这个的名称很好的解释了它的作用：向 zygote 进程发送参数然后获取结果。

zygote 进程接收到消息后就会为本次建立连接，接着会调用 zygote.forkAndSpecialize() fork子进，这个方法是通过 native 层的 ForkAndSpecializeCommon() 方法来进行 fork 的，这里不再深入，直接看看 fork 之后 Java 层的结果：

```java
try {
    if (pid == 0) {
        // in child
        IoUtils.closeQuietly(serverPipeFd);
        serverPipeFd = null;
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
        // should never get here, the child is expected to either
        // throw ZygoteInit.MethodAndArgsCaller or exec().
        return true;
    } else {
        // in parent...pid of < 0 means failure
        IoUtils.closeQuietly(childPipeFd);
        childPipeFd = null;
        return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
    }
} finally {
    IoUtils.closeQuietly(childPipeFd);
    IoUtils.closeQuietly(serverPipeFd);
}
```
这里根据不同的进程做了不同的处理，如果是子进程，就会调用 handleChildProc() 方法来进行子进程的初始化工作，这个方法如下：

```java
private void handleChildProc(Arguments parsedArgs,
        FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
        throws ZygoteInit.MethodAndArgsCaller {
    closeSocket();
    ZygoteInit.closeServerSocket();
    if (descriptors != null) {
        try {
            Os.dup2(descriptors[0], STDIN_FILENO);
            Os.dup2(descriptors[1], STDOUT_FILENO);
            Os.dup2(descriptors[2], STDERR_FILENO);
            for (FileDescriptor fd: descriptors) {
                IoUtils.closeQuietly(fd);
            }
            newStderr = System.err;
        } catch (ErrnoException ex) {
            Log.e(TAG, "Error reopening stdio", ex);
        }
    }
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }
    // End of the postFork event.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.remainingArgs);
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                parsedArgs.remainingArgs, null /* classLoader */);
    }
}
```
这里关闭了 socket，因为子进程不再需要这个 socket 了。之后会通过 parseArgs.invokeWith 来判断是启动应用进程还是启动普通 Java 进程，invokeWith 对应的是 AMS 传过来的 --invoke-with 参数，invokeWith 指定的是要执行的 Java 类。由此看来，Zygote 不仅可以启动 Android 应用进程，也可以启动普通 Java 进程。

如果启动的是 Android 应用进程，那么 RuntimeInit.zygoteInit() 方法会得到调用，这个方法会执行一些初始化工作，如设置缺省的异常处理，设置时区以及初始化 Binder 线程池。最后会调用 invokeStaticMain() 方法调用 ActivityThread 类的 main() 方法。invokeStaticMain() 方法不会直接调用 ActivityThread 的 main() 方法，而是抛出了一个 MethodAndArgsCaller 类型的异常，这个异常和普通的异常不同，它的作用是清理栈帧，我们知道，应用程序的入口是 ActivityThread，而到目前为止，程序依旧跑在 runSelectLoop() 的循环中，在 ActivityThread.main() 方法之前已经积累了很多方法栈，应用程序正常情况下是不会退出 ActivityThread.main() 方法的，所以那些积累的方法栈除了占内存没有其他用途，因此通过抛出异常的方式将它们清理掉。invokeStaticMain() 方法如下：

```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException("Missing class when invoking static main " + className, ex);
    }

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException("Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException("Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }
    
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```
这里将 main() 方法对应的 Method 对象和 argv 封装在了 MethodAndArgsCaller 对象中，然后将其抛出。异常抛出了，那么在哪里进行处理呢？在 ZygoteInit 的 main() 方法中：

```java
public static void main(String argv[]) {
    ......
    try {
        ......
        runSelectLoop(abiList);
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run(); // 通过抛出异常的方式清理方法调用栈，从此处执行 zygote 子进程中的逻辑
    } catch (Throwable ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```

我们的程序之前一直在 runSelectLoop() 方法内（为什么？因为 zygote 进程就是在这个方法中无限循环，因此 fork 出来的子进程也会处于这个循环中），抛出 MethodAndArgsCaller 异常后就会在这里被 catch 住，之后便会执行 MethodAndArgsCaller#run() 方法，这个方法如下：

```java
public void run() {
    try {
        mMethod.invoke(null, new Object[] { mArgs });
    } catch (IllegalAccessException ex) {
        throw new RuntimeException(ex);
    } catch (InvocationTargetException ex) {
        Throwable cause = ex.getCause();
        if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
        } else if (cause instanceof Error) {
            throw (Error) cause;
        }
        throw new RuntimeException(ex);
    }
}
```
这个方法仅仅是简单的调用了 ActivityThread#main() 方法，至此，子进程才真正意义上地开始了工作。以上便是 Zygote 进程的启动原理和 Android 应用进程的启动方式，接下来会分析一下普通的 Java 进程是如何启动的。

	
## 普通 Java 进程的启动
普通的 Java 进程的启动靠的也是 app_process 这个可执行文件，一般会这样启动一个普通的 Java 程序： 

```
app_process -Djava.class.path=classpath parentDir MainClass MainClassArgs...
```
其实在分析 Zygote 进程启动过程的时候已经把一部分普通 Java 进程的逻辑也分析了，因为这两种进程有很多相似的逻辑，因此他们的代码都写在一块了。分道扬镳的地方在 AppRuntime#start() 这个方法，如果启动的是普通 Java 进程，那么便会执行 RuntimeInit#main() 方法，这个方法初始化一些东西之后便会调用 AppRuntime#onStarted() 方法：

```java
virtual void onStarted()
{
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();

    AndroidRuntime* ar = AndroidRuntime::getRuntime();
    ar->callMain(mClassName, mClass, mArgs);

    IPCThreadState::self()->stopProcess();
}
```

这个方法作了两件事：初始化 Binder 线程池，调用我们指定 Java 类的 main() 方法。前面讲过，如果执行的是普通的 Java 类，通过 AppRuntime#setClassNameAndArgs() 方法要启动的类的类名和参数分别保存在 mClassName 和 mArgs 字段中，mClass 是在 AppRuntime#onVmCreated() 中创建的，它对应的是该类的 Class 对象。至此，一个 Java 进程就启动了。现在我们通过实践来体会一下让 Android 系统启动我们写好的普通 Java 程序。

### 在 Android 系统上运行普通的 Android 程序
因为只是做个试验而已，因此我们就编写个 HelloWord 类：

```java
public class HelloWorld {
	public static void main(String[] args) {
		System.out.println("hello world");
		System.out.println(args[0]);
	}
}
```
现在编译它，记住要用 JDK1.7 编译，因为 Android 目前还不支持 JDK1.8 编译出来的 class 类。编译好后就是一个 HelloWorld.class 文件了，我们知道 DVM 无法运行 class 文件，因此我们需要通过 dx 工具将该 class 文件转换成 dex 文件。dx 工具在 `sdk/build-tools/build-tool版本号` 目录下。我们将 Test.class 放置在该目录下，然后打开终端，切换到这个目录并执行如下命令：

```
dx --dex --output=test.dex HelloWorld.class
```
运行成功的话在这个目录下就能看到 test.dex 文件了，现在我们把这个文件放置在 /sdcard 下，现在通过 adb 执行以下命令：

```
adb shell
app_process -Djava.class.path=/sdcard/test.dex HelloWorld first-parameter
```
接下来不出意外的话终端会出现这两行：

```
hello world
first-parameter
```
当然，除了通过终端的方式执行 Java 程序，也可以在 Android 应用的代码里执行。前面的步骤不变，我们编写如下代码：

```java
public class MainActivity extends AppcompatActivity {
	
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.main);
		
		try {
		  Process proc = Runtime.getRuntime().exec("app_process -Djava.class.path=/sdcard/test.dex HelloWorld first-parameter");
          InputStream in = proc.getInputStream();
          InputStreamReader isr = new InputStreamReader(in);
          BufferedReader br = new BufferedReader(isr)
          Toast.makeText(this, br.readLine(), Toast.LENGTH_SHORT).show();
          Toast.makeText(this, br.readLine(), Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
          e.printStackTrace();
        }
    }
}
```
Android 应用一启动时就会执行启动 Java 进程的命令，接着获取该命令的输入流（可通过 Process 获取三种流，输入流、错误输入流和输出流，这分别对应着 Process 的输出流、错误输出流和输入流），接着通过 Taost 依次显示从输入流获取的两行字符串。不出意外的 Toast 会依次显示 “hello world” 和 “first-parameter”。

## 总结
以上是对 《深入理解 Android 5.0 系统》中第八章的总结和理解。

参考链接：

- [Android中执行java命令的方法及java代码执行并解析shell命令
](http://m.jb51.net/article/75147.htm)
- [execute shell command from android](https://stackoverflow.com/questions/20932102/execute-shell-command-from-android)
- [AndroidXref - Android Source Code Cross Reference](http://androidxref.com/7.1.1_r6/)

      
