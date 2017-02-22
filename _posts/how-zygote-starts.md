---
layout:     post
title:      "Zygote 的启动过程"
subtitle:   "how zygote starts"
date:       2017-02-22
author:     "lwenkun"
header-img: "img/post-bg-how-zygote-starts.jpg"
tags:
- android
- zygote
---

# Zygote 的启动过程 #
这篇文章是看 Gityuan 的博客文章 [Android系统启动-zygote篇](http://gityuan.com/2016/02/13/android-zygote/) 后的总结。Gityuan 的那篇博客已经很好的阐述了 `zygote` 的启动过程，所以这里我就只谈谈我自己的理解。

## C++ 层 ##

zygote 是由 init 进程 fork 来的，而 init 是 linux 系统中用户空间的第一个进程。它在运行过程中会读取一系列的后缀名为 `.rc` 的文件，然后根据这些文件的内容执行相应的动作。zygote 进程就是 init 进程
根据文件 `init.rc` 中的 `service zygote /system/bin/app_process -Xzygote /system/bin/ --zygote --start-system-server` 这行代码所启动的。这行代码同时还指定了 zygote 进程启动后需要执行二进制程序 `/system/bin/app_process`，这是 C++ 程序编译后的二进制文件，其源文件为 `App_main.cpp`，在这个 C++ 程序的入口方法 `main()` 中，主要做了这几件事：对传入的参数进行解析，然后调用 `AndroidRuntime.cpp` 中的 `AndroidRuntime::start()` 方法，该方法的前两个参数分别是要运行的 java 类的全名（这里就是 `"com.android.internal.os.ZygoteInit"`）和相关参数。由此可见该方法的作用大概就是“把指定的类放到在 java 层去执行”。但是问题来了，我们知道运行 java 程序的运行必须要有 java 环境，本地操作系统只能运行 native 语言编写的程序。在普通的 java 程序中，这个环境就是 JVM，但在安卓系统中就是 dalvik（或者 art) 虚拟机了。所以运行 `ZygoteInit` 这个类，首先得创建虚拟机。这时候就进入 `AndroidRuntime::startVm()` 方法中了。

说到虚拟机，我谈谈我自己的理解：从 java 层的角度来看，虚拟机像是一个操作系统，能够给 java 程序提供各种资源。但是从本地操作系统的角度来看，虚拟机就是一个运行在其上的普通应用，它和操作系统中的大多数应用一样，获取操作系统提供的资源，接受操作系统的管理。因此虚拟机执行 class 文件（JVM）或者执行 dex 文件（dalvik 或 art）简单点来看就是虚拟机根据文件的内容产生相应的动作的过程，这和普通程序通过读取配置文件（如 xml，json，txt）的内容执行相应的操作是等价的，只不过 class 文件可以包含更多的语义，表达的信息更丰富。

虚拟机创建好了之后并不能马上运行 java 程序，因为很多 java 程序中带有 native 方法，需要由 native 语言编写的程序去实现，但是虚拟机本身并不能提供这些方法的实现。所以就必须在虚拟机外部提供这些函数的实现，然后通过某种方式将 java 层的方法和 native 层的方法对应起来。在这里就是通过 `AndroidRuntime::startReg()` 实现的，这个方法将各种要用到的 native 层方法预先注册，使其和 java 层的方法对应起来，如创建线程的方法。（其实，JNI 技术也是通过类似方法实现的，虚拟机请求操作系统链接相应的 `.so` 动态库（这和普通 native 程序的行为是一样），然后当 class 文件执行 native 方法时，虚拟机就让本地操作系统转到 `.so` 文件中的相应方法。虚拟机本身并不能执行 `.so` 文件，它也只是普通的应用，虚拟机和动态库的代码都是在操作系统上运行的）

## java 层 ##
好了，一切准备就绪之后终于可以执行 `ZygoteInit` 类了，这个类的入口方法是 `main` 方法，这个方法做了这几件事：

- 注册 socket，开启 socket 服务端，达到进程间通信的目的
- 预加载类（通过 `Class.forName()` 加载）和资源
- 开启 system_server 进程，用于运行各种服务，供应用进程调用
- 开启循环，监听其他进程发来的消息，通过 socket 进行通信

除了 system_server 进程，所有的安卓应用也都是从 zygote fork 来的。而根据 linux 的 fork 机制，子进程相当于父进程的复制，所以 zygote 进程中创建的 dalvik 虚拟机以及预加载的各种类和资源就直接拷贝到了各应用进程中了，增加了执行效率。