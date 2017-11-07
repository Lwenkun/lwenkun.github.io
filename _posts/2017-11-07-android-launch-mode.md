---
layout:     post
title:      "Android LaunchMode 总结"
subtitle:   "Android LaunchMode 总结"
catalog:  true
date:       2017-11-07
author:     "lwenkun"
header-img: "img/post-bg-android-event-dispatch.png" 
tags:
    - Android
    - LaunchMode
---

# Android LaunchMode 总结
Android 中的 LaunchMode 是一个比较基础的知识点，关于这块之前每次都是先用现学，然后学了之后又忘了，现在把 LaunchMode 的规律记录下来以备后用。

## 需要了解的知识点
在讲 LaunchMode 之前，需要了解一下几点知识：

1. task 有属性 affinity，Activity 有属性 taskAffinity
2. 可以存在两个 affinity 一样的 task；
3. 一个 Activity 的 taskAffnity 默认值为 package name，如果有指定值就会设为指定值

## LuanchMode 规律总结

Manifest 中注册了两个 Activity，它们的类型分别为 AActivity 和 BActivity。假设 AActivity 的 taskAffinity 为 a，现在通过 BActivity 的一个实例 B 启动 AActivity：

#### 如果此时 AActivity 的实例不存在于任何 task 中，那么：

- 如果  AActivity 的启动模式为 singleTask， 那么先会找是否存在 affinity 为 a 的 task，如果存在并且这个 task 中的 Activity 不是 singleInstance，那么就在那个 task 中创建并启动一个 AActivity 的实例 ；如果不存在，那么新建一个 affinity 为 a  的 task，然后在新建的 task 中创建并启动一个 AActivity 的实例；
- 如果 AActivity 的启动模式为 singleInstance，那么无论是否存在 affinity 为 a 的 task ，总会新建一个 affinity 也为 a 的 task，并在这个新建的 task 中创建并启动一个 AActivity 的实例；
- 如果 Activity 的启动模式为 standard （默认值) 或者 singleTop，那么就要看 BActivity 的启动模式了：
   - 如果 BActivity 的启动模式不为 singleInstance 的话，那么  AActivity  的 taskAffinity 属性会被忽略，然后就在 B 所在的 task 中创建并启动一个 AActivity 的实例；
   - 如果  BActivity 的启动模式为 singleInstance 的话，那么 AActivity 的 taskAffinity 属性不会被忽略，此时系统就会去找一个 affinity 为 a 的 task：如果能找到并且这个 task 中的 Activity 不为 singleInstance，那么就会在那个 task 中创建并启动一个 AActivity 的实例；否则新建一个 affinity 为 a 的 task，然后在这个新建的 task 中创建并启动启动一个 AActivity 的实例。

#### 如果 AActivity 的实例已经存在于某个任务栈中，那么：

- 如果 AActivity 的启动模式为 standard ，那么就要看 BActivity 的启动模式了：
   - 如果 BActivity 的启动模式不为 singleInstance 的话，那么  AActivity  的 taskAffinity 属性会被忽略，然后就会在 B 所在的 task 中启动；
   - 如果  BActivity 的启动模式为 singleInstance 的话，那么 AActivity 的 taskAffinity 属性不会被忽略，系统会去找一个 affinity 为 a 的 task：如果能找到并且这个 task 中的 Activity 不为 singleInstance，那么就会在那个 task 中创建并启动一个 AActivity 的实例；否则新建一个 affinity 为 a 的 task，然后在这个新建的 task 中创建并启动一个  AActivity 的实例。
- 如果 AActivity 的启动模式为 singleTop 的话 ，同样要看 BActivity 的启动模式：
   - 如果 BActivity 的启动模式不为 singleInstance 的话，那么  AActivity  的 taskAffinity 属性会被忽略，然后就会在  B 所在的 task 中启动；但是，如果 BActivity = AActivity，那么就不会创建并启动一个 AActivity 的实例了，此时只会调用 B 的 onNewIntent 方法；
   - 如果 BActivity 的启动模式为 singleInstance 的话，那么 AActivity 的 taskAffinity 属性不会被忽略，此时系统就会去找一个 affinity 为 a 的 task：如果能找到并且这个 task 中的 Activity 不为 singleInstance，那么就会在那个 task 中启动；但是如果这个 task 的栈顶已经有一个 AActivity 的实例 A，就不会再次创建并启动另一个 AActivity 的实例了，此时只会调用栈顶的这个 A 的 onNewIntent() 方法；
- 如果 AActivity 的启动模式为 singleInstance，那么一定会找到一个 affinity 为 a 的 task（为什么？因为我们假设 AActivity 的实例已经存在于某个 task 中，而根据 singleInstance 的特点，这个 task 的 affinity 一定为 a），并且该 task 中只有一个 AActivity 的实例 A，此时不会再次创建并启动另一个 AActivity 的实例，此时只会调用  A 的 onNewIntent() 方法；
- 如果  AActivity 的启动模式为 singleTask， 那么先一定会找到 affinity 为 a 的 task（为什么？因为我们已经假设 AActivity 的实例已经存在于某个 task 中，而根据 singleTask 的特点，这个 task 的 affinity 一定为 a），并且该栈中已经存在一个 AActivity 的实例 A，此时会把该 task 中位于 A 上方的所有 Activity 清理出栈，然后调用 A  的 onNewIntent() 方法。

## 总结
关于 LaunchMode，光靠阅读文章是不能深刻理解其中规律的。大家可以尝试自己写一些各种启动模式的 Activity，让他们相互启动，再通过 adb shell dumpsys 这个命令查看任务栈以及其中的 Activity，这样应该能很快理解 LaunchMode 的规律的，也能记得更牢。
