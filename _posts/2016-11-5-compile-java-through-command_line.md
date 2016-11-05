---
layout:     post
title:      "命令行编译、打包并运行一个 java 程序"
subtitle:   "compile java through command line"
date:       2016-11-05
author:     "lwenkun"
header-img: "img/post-bg-compile-java-through-command-line.png"
tags:
- java
---

# 命令行编译、打包并运行一个 java 程序 #

很多同学一开始学 java 都是用 Eclipse、intellij 等 IDE 来写 java 程序的，这些 IDE 极大的简化了开发流程，很多工作都在不知不觉中帮我们做好了。出于好奇，在网上查阅各种资料后，决定自己动手用最原始的方式————命令行，编译并打包一个 hello world。

## 第一步：编写源代码 ##
因为是一个 hello world，用记事本来写也没任何问题。
首先我们在桌面创建项目文件夹 `HelloWorld`，在里面新建一个包名为 `xyz.lwenkun`，然后在该包下编写如下程序：

```java
package xyz.lwenkun;
//example.java
public class Example {
   public static void main(String[] args) {
      System.out.println("hello world");
   }
}
```

## 第二步：编译 ##
源码准备好后，就开始编译工作了，编译源码需要用到 `javac` 命令，使用方法是 

```
javac /Users/lwenkun/desktop/HelloWorld／xyz/lwenkun/Example.java
```
或者在源码文件所在目录执行 

```
javac Example.java
```

执行完后会在源文件所在位置生成 java 字节码文件 `Example.class`。
然后我们把源文件移除，只留下 `Example.class`。

## 第三步：打包 ##

在介绍打包前，我先来说说如何直接运行编译后的 class 文件，这里面有几个值得注意的问题。

直接运行 java 字节码文件的命令是

```
java -cp classpath main-class
```

`classpath` 是类路径的意思，java 虚拟机的系统类加载器加载类时就要用到这个变量来搜索目标类，从而将其加载到内存中，关于 `classpath` 的解释，可以看看这篇文章。`main-class` 代表的是程序的主类，它是 java 程序的入口，必须要是类的全限定名，比如我们的实例类的全限定名为 `xyz.lwenkun.Example`。

那么我们怎么运行我们的程序呢？

```
java -cp /Users/lwenkun/desktop/HelloWorld/xyz/lwenkun xyz.lwenkun.Example
```

嗯，看上去对了，但是如果你自己动手试了的话会发现会报错。我们明明就是按照命令格式来做的啊。原因就是 `classpath` 的理解问题，`classpath` 是类的搜索路径，其实严格来说就是类所在包的目录。它只需要指定类包所在文件夹，在我们的例子中就是项目根目录 `／User/lwenkun/desktop/HelloWorld`。

所以正确的写法是

```
java -cp /Users/lwenkun/desktop/HelloWorld xyz.lwenkun.Example
```

这样就会在终端输出：

```
hello world
```


现在我们来讲如何打包。

打包要完成的工作就是将我们的项目打包成一个可执行的 jar 包，常见的 jar 包有两种：一种作为其他程序的依赖库，没有主类；另一种是做为可执行程序，有主类，用鼠标点击就可以运行。我们的项目包含主类，因此我们把它打包成可执行的 jar 包。关于 jar 包更深入的分析可以看看[这篇文章](https://www.ibm.com/developerworks/cn/java/j-jar/)。jar 文件构大致如下：

 ![](/img/in-post/post_compile_java_through_command_line/structure_of_jar.png)

不管是哪种 jar 包，它们都有一个 `MEAT-INF` 目录，下面有一个 `MANIFEST.MF` 文件，这个文件一个清单，记录着 jar 包相关的一些属性，如主类、依赖库的路径，不过作为依赖库的 jar 包没有主类属性。如果打包时我们不指明清单文件那么打包工具会生成一个默认的清单：

```
Manifest-Version: 1.0
Created-By: 1.8.0_101 (Oracle Corporation)


```

默认是没有主类信息和依赖库属性，所以我们必须手动添加。我们在桌面新建一个 `MANIFEST.txt` 文件(文件名和后缀不重要)，用文本编辑器打开，在里面添加主类属性：

```
Main-Class: xyz.lwenkun.Example


```

这样我们的属性就能追加到默认的清单文件中了，如果有相同名称的属性，那么就会将默认的属性覆盖掉。

我们的项目没有依赖包，所以不需要声明依赖库路径这个属性。那假如在项目的根目录，也就是包所在目录 `/Users/lwenkun/desktop` 下有一个目录 `lib`，它下面有三个名称分别为 `lib1.jar`、`lib2.jar` 和 `lib3.jar` 的包作为我们的依赖库，那么我们的清单内容就应该是这样的：

```
Main-Class: xyz.lwenkun.Example
Class-Path: lib/lib1.jar lib/lib2.jar
 lib/lib3.jar


```


注意几点：

 - MANIFEST 清单的格式是 `key: value`，冒号后面还有一个空格
 - 依赖库之间通过空格来分隔
 - 每行一个属性，但是如果我们依赖的库太多了，可以转行，但是行首要加个空格（如上）
 - 最后，也是最容易忽视的一点，最后一个属性写完后要连续回车两次作为结束。否则，最后一行属性会被丢弃

如果我们没有额外添加 MANIFEST 属性，我们可以用 `jar cf output-file application-dir` 命令来打包；如果需要额外添加 MANIFEST 属性，打包命令就是 `jar cmf manifest-file  output-file pakage-dir1 package-dir2 ...`。当然，在这里我们额外添加了 MANIFEST 属性，所以我们打包所用的命令就是：

```
jar cmf MANIFEST.txt helloworld.jar /Users/lwenkun/desktop/HelloWorld/xyz
```

其中 `output-file` 是打包后的 .jar 文件，`package-dir` 就是我们的包的根目录，可以指定多个，在这里我们只有一个包（[更多 jar 相关命令](http://www.jianshu.com/p/61cfa1347894)）。

这样后我们就在当前目录下(假设是桌面)生成了一个 `helloworld.jar`，我们用命令行运行这个程序：

```
java -jar /Users/lwenkun/desktop/helloworld.jar
```

如果我们打包时忘记了添加 `Main-Class` 这个属性怎么办呢？当然最好的方式是重新打包。当然你说你就不想重新打包，那也行，那么执行方式就是这样的了：

```
java -cp /Users/lwenkun/desktop/helloworld.jar xyz.lwenkun.Example
```

这里把 jar 包的路径作为 `classpath`，然后在后面指明主类。前面说了 `classpath` 指明的是包所在目录，当然也可以是打包后的包所在的目录，这里可以把 `helloworld.jar` 看作是包所在目录，它的地位确实像个目录。

输出：

```
hello world
```

参考资料：

- [MANIFEST.MF中的格式问题](http://www.360doc.com/content/10/1006/17/61497_58863251.shtml)
- [Java环境变量Classpath](http://developer.51cto.com/art/201209/357217.htm)
- [命令行执行Java文件](http://www.cnblogs.com/lleid/archive/2013/03/21/java.html)
- [JAR 文件揭密](https://www.ibm.com/developerworks/cn/java/j-jar/)
