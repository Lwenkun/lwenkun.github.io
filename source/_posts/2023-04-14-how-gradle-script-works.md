---
title: Gradle 脚本运行原理
date: 2023-04-13 21:14:51
layout:     post
subtitle:  ""
author:     "李文坤"
catalog:  true
tags:
    - Gradle
---

# 前言
相信大家对 gradle 肯定已经是有一定的了解了，gradle 的强大之处不在于它本身提供了多少功能，而是它的扩展性强，几乎任务都可以通过插件的形式集成到构建流程中。我不知道大家是否和我一样，对 gradle 背后的原理很感兴趣，想要知道我们平常的那些傻瓜式的配置，背后到底是如何生效的。这篇文章将会介绍 gralde 脚本的运行原理。

# 一个简单的 gradle 脚本
build.gradle
```groovy
plugins {
    id 'org.jetbrains.kotlin.jvm' version '1.8.0'
    id 'application'
}

group = 'me.liwenkun'
version = '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.jetbrains.kotlin:kotlin-test'
    implementation gradleApi()
}

test {
    useJUnitPlatform()
}

kotlin {
    jvmToolchain(11)
}

application {
    mainClassName = 'MainKt'
}
```

这个脚本相信大家很熟悉，groovy 的语法我在这里不再介绍，只是针对这个脚本里出现的语法简单说一下：groovy 的方法调用和 java 其实差不多，方法名+()，只不过可以省略括号。大括号其实是闭包的语法，所以，上面的 plugins {}、repositories {} 之类的配置块其实就是一个方法调用，方法名是大括号前面的标识符，方法参数就是大括号表示的闭包。但是，虽然这些配置块看上去都一样，但其实是有本质区别的，后面我们会讲到。

# groovy 
在了解脚本的运行原理之前，我们首先需要了解 groovy，因为 gradle 脚本其实就是 groovy 脚本，只是后缀名改了而已。能成为脚本语言的语言，它一定能够解释执行，至少看上去是解释执行的。那么 groovy 解释执行的方法有哪些呢？
1. 命令行运行脚本文件
```shell
grovvy 
```
2. 命令行交互式运行
```shell
```
3. 将 groovy 作为项目依赖，在项目中使用其提供的接口来运行
```java
```
前两种用法只是单纯的使用，面向的普通用户，使用的是命令行接口；而第三种方式则是面向开发者，将 groovy 集成到第三方应用中，将其作为应用的一部分来使用，使用的是它的编程接口。
> 其实不仅仅是 groovy，很多的 C 语言编写的工具包都提供这两种使用方式，以 linux 下的软件包为例：bin 目录下的文件是可执行文件，而 lib 目录下则是库。bin 依赖于 lib，是 lib 的包装，bin 用来在命令行使用，如果是工具面向开发者，除了提供 bin 和 lib 还会提供 include 作为编程接口，这样开发时就可以通过引入 include 中的头文件，来调用 lib 下的各种库

我们知道，虽然是脚本语言，但它最终还是运行在 JVM 之上的，JVM 可不会直接执行源码，它只认识字节码。所以，groovy 必然要先将源代码编译成字节码文件，才能运行。然而，当 gradle 执行我们的脚本文件的时候，我们并不知道它到底把字节码文件生成在了哪里，幸运的是，我们可以通过断点调试来获取。idea 中在任务旁边会有三角形，我们点击这个三角形，选择调试任务，这样 gradle 中的断点就会断住。然后我们通过调用栈可以找到这个 gradle 文件对应的字节码文件的位置。我们将这个字节码文件拖入到 idea 窗口中，就会得到反编译后的源码文件。

这个源码文件对应的类叫做 `_BuildScript_`，这个类只有一个方法 `run()`，这个类继承自 `ProjectScript`。这是什么情况，为什么会得到这个类。我们有必要了解下 Groovy 编译机制，默认情况下，它会将所有的脚本通通放在一个继承自 `Script` 类的子类的 `run()` 方法中运行，`Script` 类是 Groovy 提供的类，表示一个脚本对象。这个继承自 `Script` 的类是动态生成的。可以这么认为，`Script` 只是一个模板，需要子类来对这个模板进行填充，填充的过程就是将脚本文件的代码放到 `run()` 中运行。这个子类的名字是可以定义的，除此之外，模板也可以自定义，但必须是 `Script` 的子类。就是这一点，给了 gradle 发挥的空间。下面我们来看看 Gradle 是如何利用这一点来发挥的。

前面讲过了使用 groovy 的第三种方式，前面两种面向普通用户，可定制性差，而第三种面向开发者，可定制性就很强了，因为编程接口远比命令行接口丰富。对于 gradle 开发团队来说，他们就是通过第三种方式来使用 groovy 的。于是我们可以推测一下，gradle 应该是将脚本的模板类指定为了自己的 `ProjectScript` 类，并且将生成的类叫做 `_BuildScript_` ，这些工作在代码中如何实现呢?

实际的实现过程比较复杂，逻辑弯弯绕绕藏得比较深，我将其简化一下，用一下代码来表示：


```java

```

可见，Gradle 将脚本的模板类指定为 ProjectScript，而且调用了脚本的 init() 方法来对这个脚本初始化。init() 方法接受两个参数，我们只关注第一个参数，第一个参数是 DefaultProject 对象。如果把这两个类搞清楚了，那么 Gradle 的脚本运行机制可以说基本就掌握了。

我们一个个来。先看 ProjectScript，这个对象中有很多我们熟知的方法：plugins() apply();
ProjectScript 的继承关系是：ProjectScript -> PluginsAwareScript -> DefaultScript -> BasicScript -> org.gradle.groovy.scripts.Script -> groovy.lang.Script。

先看看 groovy.lang.Script 的源码：
```java
class abstract Script extends GroovyObjectSupport {
    public void setBinding(Binding binding) {
        this.binding = binding;
    }

    public Object getProperty(String property) {
        try {
            return binding.getVariable(property);
        } catch (MissingPropertyException e) {
            return super.getProperty(property);
        }
    }

    public void setProperty(String property, Object newValue) {
        if ("binding".equals(property)) {
            setBinding((Binding) newValue);
        } else if ("metaClass".equals(property)) {
            setMetaClass((MetaClass) newValue);
        } else if (!binding.hasVariable(property)
                // GROOVY-9554: @Field adds setter
                && hasSetterMethodFor(property)) {
            super.setProperty(property, newValue);
        } else {
            binding.setVariable(property, newValue);
        }
    }

    public Object invokeMethod(String name, Object args) {
        try {
            return super.invokeMethod(name, args);
        }
        //...
    }

    public void println() {}
    public void print(Object value) {}
    public void println(Object value) {}
    public Object evaluate(String expression) throws CompilationFailedException {}
    public abstract Object run();

    //...
}
```

这就是我们说的 groovy 编译脚本时提供的模板类，这个类其实上面还有两层继承关系，但是由于比较简单，我这里就没有表示出来。我们的脚本代码最终都会编译到 run 运行。我们看到这里定义了一些熟悉的方法。

而 ProjectScript -> PluginsAwareScript -> DefaultScript -> BasicScript 这几个类定义了 gralde 脚本中我们常见的方法：

这些方法都是脚本可以直接调用的，那么具体脚本是如何利用，我们知道，我们的脚本逻辑最终都是运行在 run() 里面的，脚本是如何调到模板类中的对象的呢？

我们就必须看一下 BuildScript 的 run() 方法了：
 ```java
 ```
可以看到，这里面并没有出现 println，取而代之的是 Callsite，这个是什么呢？我们可以姑且认为这是对 println 的反射调用。

好了