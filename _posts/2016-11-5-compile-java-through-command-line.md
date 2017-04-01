---
layout:     post
title:      "命令行编译、打包并运行一个 java 程序"
subtitle:   "compile java through command line"
date:       2016-11-05
catalog:  true
author:     "lwenkun"
header-img: "img/post-bg-compile-java-through-command-line.png"
tags:
    - Java
---

# 命令行编译制作一个 Java 程序 #

和很多同学一样，一开始学 java 都是用 Eclipse、intellij 等 IDE 来写 java 程序的，这些 IDE 极大的简化了开发流程，很多工作都在不知不觉中帮我们做好了。出于好奇，在网上查阅各种资料后，决定自己动手用最原始的方式————命令行，编译并打包一个 hello world。

## 第一步：编写源代码 ##
因为是一个 hello world，用记事本来写也没任何问题。
首先我们在桌面创建项目文件夹 `HelloWorld`，在里面新建一个包名为 `xyz.lwenkun`，然后在该包下编写如下程序：

```java
package xyz.lwenkun;

import com.lib;
//Example.java
public class Example {
   public static void main(String[] args) {
      Lib lib = new Lib();
      lib.print();
   }
}
```

同时在桌面新建一个包 ”com.lib“，在里面编写 `Lib.java` 作为我们 HelloWorld 要依赖的类： 

```java
package com.lib
//Lib.java
public class Lib {
   public void print() {
      System.out.println("hello world");
   }
}
```

## 第二步：编译 ##
源码准备好后，就开始编译工作了，编译源码需要用到 `javac` 命令，使用方法是 

```
javac <options> <source-files>
```

因为我们的项目依赖于 `Lib` 这个类，所以我们首先把这个类编译好：

```
javac /Users/lwenkun/desktop/com/lib/Lib.java
```

然后再编译主类：

```java
javac -cp /Users/lwenkun/desktop /Users/lwenkun/desktop/HelloWorld/xyz/lwenkun/Example.java
```

不同于编译 `Lib` 类，这里我们用到了 `-cp` 选项。其中 `-cp` 是 `-classpath` 的简写，`-classpath` 后面指定的一般是被引用的类所属类包所在的目录或者所在 jar 包的路径（我们称其为 `classpath`），编译时或者运行时 JVM 的**系统类加载器**就要用到 `classpath` 变量来搜索目标类。注意这个变量指明的是类所属**类包所在的目录**或者**所在 jar 包的路径**而不是**具体类的路径**。比如在编译某个类时要引用另一个类 `Lib1`，这个类在类包 `com.example1` 中，而这个类包又在 `/Users/lwenkun/desktop/package-dir1` 目录下 ，那么我只需指定 `classpath` 为 `/Users/lwenkun/desktop/package-dir1` 就可以了。当然在实际情况中一个类引用到的类有很多，如果这些被引用的类(1)在同一个包下(2)或者它们所属类包在同一目录下(3)或者在同一 jar 包内，`classpath` 自然就为同一个值，那就不需要重复指定了。但是如果它们(1)在不同的包里而且这些类包位于不同的目录下(2)或者在不同的 jar 包内，比如我还要引用一个类 `Lib2`，它在一个名为 `com.example2` 的类包下，这个类包又位于 `/Users/lwenkun/desktop/package-dir2` 目录下，那么就要指定多个 `classpath` 了，这些 `classpath` 之间用 `:` （macOSx、Linux、Unix）或者 `;` （Windows）隔开，如：

```java
javac -cp /Users/lwenkun/desktop/package-dir1:/Users/lwenkun/desktop/package-dir2 ClassToBeCompiled.java
```

关于 `classpath` 的更多解释，可以看看[这篇文章](http://developer.51cto.com/art/201209/357217.htm)。要注意的是：如果不指定的话，classpath 的默认值是 `.`，代表的就是当前的用户目录；如果用户指定了那这个默认值就会被清除。

在我们的 HelloWorld 项目中，我们用到了位于桌面的 `com.lib` 包中的 `Lib` 类，所以我们需要指定的 `classpath` 当然是 `／Users/lwenkun/desktop` 了。

执行完后这两个编译命令后会分别在各自源文件所在目录生成 java 字节码文件 `Example.class` 和 `Lib.class`。源文件现在已经没用了，我们把源文件移除，只留下 `Example.class` 和 `Lib.class`。

实际上编译完之后就可以运行了，运行 java 字节码的命令是

```
java <options>  <main-class> [args...]
```

这里我们运行的命令是：

```java
java -cp /Users/lwenkun/desktop:/Users/lwenkun/desktop/HelloWorld xyz.lwenkun.Example
```

发现和编译时格式差不多，`-cp` 后面指定的是运行时的 `classpath`, 虚拟机此时会根据这个值查找目标类。要注意的就是最后面的类名是主类的全限定名，比如我们的主类的全限定名就为 `xyz.lwenkun.Example`。系统查找类会根据类包所在目录结合类的全限定名来定位具体的类。来说下这条命令做了些什么：首先当然是启动 java 虚拟机，然后查找指定的主类，依据什么来查找呢？当然就是前面指定的 `classpath`，先在第一个目录下找，根据全限定名，类的位置应该是 `/Users/lwenkun/desktop/xyz/lwenkun/Example`，发现找不到，再用同样的方法在第二个目录中找，这时候类的位置应该是 `/Users/lwenkun/desktop/HelloWorld/xyz/lwenkun/Example`，发现找到了，那么就加载这个类并执行这个类的 `mian()` 方法。`main()` 方法中又用到了 `Lib` 这个类，它在 `Example` 中的声明是 `com.lib.Lib`，类加载器又用类似方法开始查找 `Lib` 类，发现在第一个目录中找到了该类，那么就把它加载到内存当中。这里省略了很多细节，关于系统查找类的详细说明，可以看看[这篇文章](http://docs.oracle.com/javase/6/docs/technotes/tools/findingclasses.html)。

命令的运行结果是：

```
hello world
```

## 第三步：打包 ##

打包就是我们的项目打包成 jar 包，jar 包是一种 zip 格式的文件，从结构上来看我们可以简单的理解为 jar 包就是把几个类包压缩在一起。常见的 jar 包有两种：一种作为其他程序的依赖库，没有主类；另一种是作为可执行的程序，有主类，用鼠标点击就可以运行。我们的项目包含主类，因此我们把它打包成可执行的 jar 包。关于 jar 包更深入的分析可以看看[这篇文章](https://www.ibm.com/developerworks/cn/java/j-jar/)。jar 文件构大致如下：

 ![](/img/in-post/post_compile_java_through_command_line/structure_of_jar.png)
 
 不管是哪种 jar 包，它们都有一个 `META-INF` 目录，下面有一个 `MANIFEST.MF` 文件，这个文件是个清单，记录着 jar 包相关的一些属性，常用的如 `Main-Class` 和 `Class-Path`，前者指定主类，也就是程序的入口点，后者的作用和命令行中的 `-cp` 是同样的含义，用来指定引用到的类所属类包所在的目录或者所在的 jar 包路径。如果打包时我们不指明清单文件那么打包工具会生成一个默认的清单添加到 jar 包中：

```
Manifest-Version: 1.0
Created-By: 1.8.0_101 (Oracle Corporation)


```

默认的清单中没有 `Main-Class` 和 `Class-Path` 等属性，如果需要这些属性我们必须自己添加。添加方法是新建一个 `MANIFEST.txt` 文件(文件名和后缀不重要)，用文本编辑器打开，在里面添加某些属性：

```
属性名1: 属性值1
属性名2: 属性值2


```

到时候我们就能通过打包命令把这个文件中的属性追加到默认的清单文件中了。如果有和默认属性名称相同的属性，会将默认属性覆盖掉。

打包用到的命令是：

```java
jar {ctxui}[vfmn0PMe] [jar-file] [manifest-file] [entry-point] [-C dir] files ...
```

其中 `jar-file` 是输出的 jar 包的文件名，`manifest-file` 是清单文件的名称（如果有的话），`files...` 是指若干要打包的 Class 文件。

介绍几种比较简单的打包方式（[更多 jar 相关命令](http://www.jianshu.com/p/61cfa1347894)）：

- 使用默认清单文件和几个类包创建一个 jar 文件

```
jar cf class.jar package-root-dirs...
```

- 使用现有清单文件和几个类包创建一个 jar 文件

```
jar cfm class.jar mainfest-file package-root-dirs...
```

- 使用现有的清单文件并用 `foo` 目录下的所有包创建一个 jar 文件

```
jar cvfm classes.jar manifest-file -C foo/ .
```

其中，`package-root-dir` 指的是包的根目录，例如，在我们的项目中就是 `/Users/lwenkun/desktop/HelloWorld/xyz`。`foo` 就是包所在的目录，在我们的项目中就是 `/Users/lwenkun/desktop/HelloWorld`。

我们先将依赖类所在包的打成 `lib.jar` ：

```
jar cf lib.jar /Users/lwenkun/desktop/com
```

然后把生成的 `lib.jar` 放在 `HelloWorld` 文件夹的 `lib` 文件夹中。

对于主项目，因为我们的项目有主类，所以要声明 `Main-Class` 属性；又因为我们的项目有要依赖的类，并且我们要依赖的类的类包没有和我们的主项目的类包在同一目录下，所以要声明 `Class-Path` 属性。因此我们要创建包含如下内容的清单文件 `MANIFEST.txt`（文件名和后缀不重要，只要是文本类型的）：

```
Main-Class: xyz.lwenkun.Example
Class-Path: lib/lib.jar


```

这样打包时文件中的这些属性就会追加到默认清单文件中了。根据前面我们对 `classpath` 的解释，如果我们的项目依赖的类分别处于三个 jar 包和一个类包中，其中 jar 包位于 `lib` 目录下，名称分别为 `lib1.jar`、`lib2.jar` 和 `lib3.jar`，类包位于桌面（`desktop`），那么我们的清单内容就应该是这样的：

```
Main-Class: xyz.lwenkun.Example
Class-Path: lib/lib1.jar lib/lib2.jar
 lib/lib3.jar /Users/lwenkun/desktop


```

`Class-path` 前三项指定的是 jar 包的路径（相对），最后一项指定的是类包所在目录（绝对）。

注意几点：

 - `MANIFEST` 清单的格式是 `key: value`，冒号后面还有一个空格（如上）
 - 依赖库之间通过空格来分隔
 - 每行一个属性，但是如果我们依赖的库太多了，可以转行，但是行首要加个空格（如上）
 - 最后，也是最容易忽视的一点，最后一个属性写完后要连续回车两次作为结束。否则，最后一行属性会被丢弃

现在我们来打包主项目中的类包：

```java
jar cfm helloworld.jar MANIFEST.txt /Users/lwenkun/desktop/HelloWorld/xyz
```

这样我们就在当前目录下(项目根目录 `HelloWorld`)生成了一个 `helloworld.jar`。把类包打包后，类包对于我们来说已经没用了，可以都移除掉。现在我们用命令行运行这个程序：

```java
java -jar /Users/lwenkun/desktop/HelloWorld/helloworld.jar
```

输出：

```
hello world
```

这里是通过 jar 包来执行我们的程序，因为 jar 包中指定了 `Main-Class`，所以 JVM 就能找到相应的主类并执行；同时在 `MANIFEST.MF` 中我们指定了 `Class-Path`，把我们依赖的 jar 包的相对路径添加进去了，所以要用到依赖类的时候，JVM 也能根据这个相对路径和依赖类的全限定名定位到依赖类。

如果我们打包时忘记了添加 `Main-Class` 这个属性怎么办呢？当然最好的方式是重新打包。当然你说你就不想重新打包，那也行，那么执行方式就是这样的了：

```java
java -cp /Users/lwenkun/desktop/HelloWorld/helloworld.jar xyz.lwenkun.Example
```

输出：

```
hello world
```

这里把主项目的 jar 包的路径添加进 `classpath`，然后在后面指明主类。前面说了 `classpath` 指明的是类包所在目录，也可以是类所在 jar 包的路径。其实你可能已经明白，类包目录和 jar 包路径其实是同一回事，它们都是类包的容器。

我们的程序有依赖库(`/Users/lwenkun/desktop/HelloWorld/lib/lib.jar`)，如果你又忘记添加 `Class-Path` 属性又该怎么办？可能你已经知道方法了：

```java
java -cp /Users/lwenkun/desktop/HelloWorld/helloworld.jar:/Users/lwenkun/desktop/HelloWorld/lib/lib.jar xyz.lwenkun.Example
```

我们把依赖类所在 jar 包的路径添加到 `classpath` 中，这样的话，虽然 `MANIFEST.MF` 中没有声明我们程序依赖的 jar 包路径，JVM 照样能够根据 `-cp` 指定的路径找到我们的依赖类。

参考资料：

- [MANIFEST.MF中的格式问题](http://www.360doc.com/content/10/1006/17/61497_58863251.shtml)
- [Java环境变量Classpath](http://developer.51cto.com/art/201209/357217.htm)
- [命令行执行Java文件](http://www.cnblogs.com/lleid/archive/2013/03/21/java.html)
- [JAR 文件揭密](https://www.ibm.com/developerworks/cn/java/j-jar/)
