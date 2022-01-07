---
layout:     post
title:      "安卓动态加载入门"
subtitle:   "安卓动态加载"
date:       2016-11-11
catalog:  true
author:     "lwenkun"
header-img: "img/post-bg-android-load-class-dynamically.jpg"
tags:
    - Android
    - 动态加载
---

# 安卓动态加载入门 #

这几周为了理解安卓动态加载技术算是花了不少时间，遇到很多坑，当然也学到了不少。一开始是学习 java 虚拟机，了解类文件格式，然后又在各种博客网站上看 dalvik 虚拟机和 dex 文件格式，了解安卓的类加载机制，到后来又去了解 art 虚拟机和 oat 文件格式。虽然有些地方没搞太清楚，学习的不够深入，但总算把动态加载的大概原理弄清了，也算是为之后更深入学习安卓动态加载以及热修复、热更新等技术打下基础吧。

<!-- more -->

## 什么是动态加载技术 ##

这个在网上没有看到严格的定义，不过就我个人的理解，动态加载代码就是通过在运行时加载外部代码（磁盘，网络等）改变程序行为的技术。关于安卓动态加载技术的文章网上有很多，但很多都是基于较低安卓版本的，对于较高版本有些地方不一定适用。我这里准备基于 andriod M 来和大家分享一下安卓的动态加载技术，让大家对这项技术有一个初步的了解。

## 动态加载技术详解 ##

不管是 java 应用还是安卓应用，动态加载技术的核心都是类加载机制，所以我们有必要先了解下安卓的类加载机制，而安卓的类加载机制沿袭了普通的 java 应用的类加载机制，因此我们先看看 java 虚拟机（JVM）是怎么加载类的。

### JVM 类加载机制 ###
JVM 的类加载机制是双亲委派模型，但是这个“双亲”感觉有点误导，因此我更喜欢叫它委派式模型。这里不对 JVM 委派式的类加载机制做过多分析，贴上一张图供大家去理解：

![](/img/in-post/post_android_load_class_dynamically/android-load-class-dynamically-1.png)

结合这张图说明几点：

- `BootStrapClassLoader` 是顶级的类加载器，它是唯一一个不继承自 `ClassLoader` 的类加载器，它高度集成于 JVM，是 `ExtensionClassLoader` 的父加载器，它的类加载路径是 `JDK\jre\lib` 和 用户指定的虚拟机参数 `-Xbootclasspath` 的值。
- `ExtensionClassLoader` 是 `BootStrapClassLoader` 的子加载器，同时是 `SystemClassLoader` （有的地方称 `AppClassLoader`）的父加载器，它的类加载路径是 `JDK\jre\lib\ext` 和系统属性 `java.ext.dirs` 的值。
- `SystemClassLoader` 是 `ExtensionClassLoader` 的子加载器，同时是我们的应用程序的类加载器，我们在应用程序中编写的类一般情况下（如果没有用到动态加载技术的话）都是通过这个类加载加载的。它的类加载路径是环境变量 `CLASSPATH` 的值或者用户通过命令行可选项 `-cp (-classpath)` 指定的值。
- 类加载器由于父子关系形成树形结构，开发人员可以开发自己的类加载器从而实现动态加载功能，但必须给这个类加载器指定树上的一个节点作为它的父加载器。
- 因为类加载器是通过包名和类名（或者说类的全限定名），所以由于委派式加载机制的存在，全限定名相同的类不会在有 **祖先—子孙** 关系的类加载器上分别加载一次，不管这两个类的实现是否一样。
- 不同的类加载器加载的类一定是不同的类，即使它们的全限定名一样。如果全限定名一样，那么根据上一条，这两个类加载器一定没有 **祖先-子孙** 的关系。这样来看，可以通过自定义类加载器使得相同全限定名但实现不同的类存在于同一 JVM 中，也就是说，类加载器相当于给类在包名之上又加了个命名空间。
- 如果两个相同全限定名的类由两个非 **祖先-子孙** 关系的类加载器加载，这两个类之间通过 `instanceof` 和 `equals()` 等进行比较时总是返回 `false`。

我们知道，安卓应用和普通的 java 应用不同，它们运行于 Dalvik 虚拟机。JVM 是基于栈的虚拟机，而 Dalvik 是基于寄存器的虚拟机。因此，java 虚拟机具有更大的指令集，而 Dalvik 虚拟机的指令更长。除此之外，考虑到 Dalvik 虚拟机运行于移动设备，内存空间和 CPU 执行效率有限，因此采用 dex 作为储存类字节码信息的文件。当 java 程序编译成 class 后，编译器会使用 dx 工具将所有的class 文件整合到一个 dex 文件，目的是使其中各个类能够共享数据，在一定程度上降低了冗余，同时也是文件结构更加紧凑。虽然这两种虚拟机有诸多不同，但是 Dalvik 继承了 JVM 的委派式的类加载机制，因此上面的**部分**（主要是后面四条）结论对于安卓来说也是同样适用的。

因为安卓的类加载机制也是委派式的，所以如果你知道 JVM 的类加载机制，那么通过类比学习安卓的类加载机制就很容易了。本来准备放张图来对比说明安卓的类加载模型的，但是想想我们还是有必要先了解安卓中两个重要的类加载器以及内部的细节：`DexClassLoader` 和 `PathClassLoader`。

### DexClassLoader & PathClassLoader ###

先看看这两个类加载器的定义（点击超链接可查看注释）：

- [DexCloassLoader](http://androidxref.com/7.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java)

```java
package dalvik.system;
import java.io.File;

public class DexClassLoader extends BaseDexClassLoader {
   
    public DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

- [PathClassLoader](http://androidxref.com/7.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java)

```java
package dalvik.system;
public class PathClassLoader extends BaseDexClassLoader {
   
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String libraryPath, ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

可以看到，这两个类加载器都是继承自 [BaseDexClassLoader](http://androidxref.com/7.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)，只是分别实现了自己的构造方法。那么我们自然对这个 BaseDexClassLoader 很感兴趣，看看它的构造方法：

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);
}
```

说下这个构造方法的几个参数：

- 第一个参数指的是我们要加载的 dex 文件的路径，它有可能是多个 dex 路径，取决于我们要加载的 dex 文件的个数，多个路径之间用 `:` 隔开。
- 第二个参数指的是优化后的 dex 存放目录。实际上，dex 其实还并不能被虚拟机直接加载，它需要系统的优化工具优化后才能真正被利用。优化之后的 dex 文件我们把它叫做 odex （optimized dex，说明这是被优化后的 dex）文件。其实从 class 到 dex 也算是经历了一次优化，这种优化的是机器无关的优化，也就是说不管将来运行在什么机器上，这种优化都是遵循固定模式的，因此这种优化发生在 apk 编译。而从 dex 文件到 odex 文件，是机器相关的优化，它使得 odex 适配于特定的硬件环境，不同机器这一步的优化可能有所不同，所以这一步需要在应用安装等运行时期由机器来完成。需要注意的是，在较早版本的系统中，这个目录可以指定为外部存储中的目录，较新版本的系统为了安全只允许其为应用程序私有存储空间（`/data/data/apk-package-name/`）下的目录，一般我们可以通过 `Context#getDir(String dirName)` 得到这个目录。
- 第三个参数的意义是库文件的的搜索路径，一般来说是 `.so` 库文件的路径，也可以指明多个路径。
- 第四个参数就是要传入的父加载器，一般情况我们可以通过 `Context#getClassLoader()` 得到应用程序的类加载器然后把它传进去。

这个构造函数的意义很简单，它做了两件事：连接了父加载器；构造了一个 [DexPathList](http://androidxref.com/7.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java) 实例保存在 `pathList` 中。这个 `pathList` 现在我们还不知道它是何方神圣，但是我们通过类名隐约的感觉到它保存了 Dalvik 虚拟机要加载的 dex 文件的路径，实际情况如何呢？我们看看这个类：

```java
public DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory) {

        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
       }

       if (optimizedDirectory != null) {
           if (!optimizedDirectory.exists())  {
               throw new IllegalArgumentException(
                       "optimizedDirectory doesn't exist: "
                       + optimizedDirectory);
           }

           if (!(optimizedDirectory.canRead()
                           && optimizedDirectory.canWrite())) {
               throw new IllegalArgumentException(
                       "optimizedDirectory not readable/writable: "
                       + optimizedDirectory);
           }
       }

       this.definingContext = definingContext;

       ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
       // save dexPath for BaseDexClassLoader
       this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                              suppressedExceptions, definingContext);

       // Native libraries may exist in both the system and
       // application library paths, and we use this search order:
       //
       //   1. This class loader's library path for application libraries (librarySearchPath):
       //   1.1. Native library directories
       //   1.2. Path to libraries in apk-files
       //   2. The VM's library path from the system property for system libraries
       //      also known as java.library.path
       //
       // This order was reversed prior to Gingerbread; see http://b/2933456.
       this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
       this.systemNativeLibraryDirectories =
               splitPaths(System.getProperty("java.library.path"), true);
       List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
       allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

       this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories,
                                                         suppressedExceptions,
                                                         definingContext);

       if (suppressedExceptions.size() > 0) {
           this.dexElementsSuppressedExceptions =
               suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
       } else {
           dexElementsSuppressedExceptions = null;
       }
}
```

这个构造方法也很简单，这里我们主要看这几行代码：

```java
this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                              suppressedExceptions, definingContext);
...
this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
this.systemNativeLibraryDirectories = 
                splitPaths(System.getProperty("java.library.path"), true);
List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories,
                                          suppressedExceptions,
                                          definingContext);
```

这几行代码做的事也很清晰明了，就是给两个字段赋值。一个是 `dexElements`，另一个是 `nativeLibraryPathElements`。我们来看看这两个字段是怎么得到的：

- `dexElements` 是通过 `makeDexElements()` 方法得到的，我们主要关注这个方法的前两个参数。第二个参数前面已经说了，是 dex 文件优化后的存放目录。第一个参数是通过 `splitDexPath()` 得到的，这个方法方法最终会调用 `splitPaths()`，所以我们看看 `splitPaths()` 是怎样的：

```java
private static List<File> splitPaths(String searchPath, boolean directoriesOnly) {
       List<File> result = new ArrayList<>();

       if (searchPath != null) {
           for (String path : searchPath.split(File.pathSeparator)) {
               if (directoriesOnly) {
                   try {
                       StructStat sb = Libcore.os.stat(path);
                       if (!S_ISDIR(sb.st_mode)) {
                           continue;
                       }
                   } catch (ErrnoException ignored) {
                       continue;
                   }
               }
               result.add(new File(path));
           }
       }

       return result;
}
```

这个方法做的事正如其名字所表达的，就是把用 `:` 分隔的路径分割后保存为 File 类型的列表返回。现在看看 `makeDexElements()` 这个方法：

```java
private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
                                            List<IOException> suppressedExceptions,
                                            ClassLoader loader) {
     return makeElements(files, optimizedDirectory, suppressedExceptions, false, loader);
}
```
就是利用已有参数简单调用了 `makeElements()`，其中，`ignoreDexFiles` 传入的是 `false`，`makeElements()` 的实现： 

```java
private static Element[] makeElements(List<File> files, File optimizedDirectory,
                                          List<IOException> suppressedExceptions,
                                          boolean ignoreDexFiles,
                                         ClassLoader loader) {
       Element[] elements = new Element[files.size()];
       int elementsPos = 0;
       /*
        * Open all files and load the (direct or contained) dex files
        * up front.
        */
       for (File file : files) {
           File zip = null;
           File dir = new File("");
           DexFile dex = null;
           String path = file.getPath();
           String name = file.getName();

           if (path.contains(zipSeparator)) {
               String split[] = path.split(zipSeparator, 2);
               zip = new File(split[0]);
               dir = new File(split[1]);
           } else if (file.isDirectory()) {
               // We support directories for looking up resources and native libraries.
               // Looking up resources in directories is useful for running libcore tests.
               elements[elementsPos++] = new Element(file, true, null, null);
           } else if (file.isFile()) {
               if (!ignoreDexFiles && name.endsWith(DEX_SUFFIX)) {
                   // Raw dex file (not inside a zip/jar).
                   try {
                       dex = loadDexFile(file, optimizedDirectory, loader, elements);
                   } catch (IOException suppressed) {
                       System.logE("Unable to load dex file: " + file, suppressed);
                       suppressedExceptions.add(suppressed);
                   }
               } else {
                   zip = file;

                   if (!ignoreDexFiles) {
                       try {
                           dex = loadDexFile(file, optimizedDirectory, loader, elements);
                       } catch (IOException suppressed) {
                           /*
                            * IOException might get thrown "legitimately" by the DexFile constructor if
                            * the zip file turns out to be resource-only (that is, no classes.dex file
                            * in it).
                            * Let dex == null and hang on to the exception to add to the tea-leaves for
                            * when findClass returns null.
                            */
                           suppressedExceptions.add(suppressed);
                       }
                   }
               }
           } else {
               System.logW("ClassLoader referenced unknown path: " + file);
           }

           if ((zip != null) || (dex != null)) {
               elements[elementsPos++] = new Element(dir, false, zip, dex);
           }
        }
       if (elementsPos != elements.length) {
           elements = Arrays.copyOf(elements, elementsPos);
       }
       return elements;
}
```

这个方法的名字也很好的说明了它要做的事，就是装配 `Element` 数组。装配 `Element` 数组的工作主要在 `for` 循环中，除了异常情况，它的每一次循环都构造了一个 `Element`。`Element` 是什么东西？你可以大概的把它理解为一个实体类。忽略异常情况，我们现在来分析这些 `Element` 是如何构造的，首先循环的开始部分定义了构造 `Element` 要用到的参数，然后对传入的每个 `File` 判断其类型：

- 第一个判断我也没看太懂，不知道为什么这么做，好在这不是重点，我们往后看。

- 第二个判断是，如果文件是一个目录，那么直接把这个目录传入 `Element` 的构造方法构造一个 `Element`；如果不是就进行下一个判断。

- 第三个判断中又有两个判断：

  - 根据后缀看它是不是 dex 文件，如果是，那么就通过 `loadDexFile()` 来加载一个 [DexFile](http://androidxref.com/7.0.0_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexFile.java) 对象（这个 `DexFile` 是什么我们等下再讲，你可以把它理解为一个对应着一个 dex 文件的对象）。如果成功加载了，那么就把它传入 `Element` 构造方法构造一个 `Element`。
  - 如果不是 dex 文件，那么不管它什么后缀名，都把它看作是一个 zip，前提是它必须是一个 zip 格式的文件（如 zip，jar，apk），并且这个 zip 格式的文件必须要包含一个 dex 文件，同时这个文件须位于 zip 内部的根目录下。然后又会利用这个 zip 文件加载一个 `DexFile` 对象。最后将这个 zip 和连同加载出来的 `DexFile` 对象一起传入 `Element` 的构造方法构造一个 `Element` 对象。

`Element` 数组的构造我们大概理解清楚了。现在看下 `loadDexFile()` 怎样加载 `DexFile` 的：

```java
private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
                                      Element[] elements) throws IOException {
       if (optimizedDirectory == null) {
           return new DexFile(file, loader, elements);
       } else {
           String optimizedPath = optimizedPathFor(file, optimizedDirectory);
           return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
       }
}
```

先说明下，无论是 `DexFile(File file, Classloader loader, Elements[] elements)` 还是
`DexFile.loadDex()` 最终都会调用 `DexFile(String sourceName, String outputName, int flags, ClassLoader loader, DexPathList.Element[] elements)` 这个构造方法。所以 `loadDexFile()` 这个方法的逻辑就是：如果 `optimizedDirectory` 为 null，那么就直接利用 dex 文件对应的 `file` 构造一个 `DexFile`；否则就根据要加载的 dex（或者包含了 dex 的 zip） 的文件名和优化后的 dex 存放的目录组合成优化后的 dex（也就是 odex）文件的输出路径，然后利用原始路径和优化后的输出路径构造出一个 `DexFile`。关于 `DexFile` 内部的细节到时候分析类加载过程的时候会讲，这里就不细说了。

通过前面的分析我们知道，我们可以知道 `dexElements` 主要作用就是用来保存和 dex 文件对应的 `DexFile` 对象的。 

- `nativeLibraryPathElements` 产生的方法和 `pathList` 差不多，它保存的主要是本地方法库（本地方法库的存在形式一般是 `.so` 文件）对应的对象，包括应用程序的本地方法库和系统的本地方法库。这里就不对它过多讲解了。

分析完这两字段，现在我们回过头来看看 `DexPathList` 这个对象，这个对象持有 `dexElements` 和 `nativeLibraryPathElements` 这两个属性，也就是说它保存了 dex 和 本地方法库。而 dex 保存着类的字节码信息，这样的话如果我们的类加载器要加载某个类的话，是不是只要操作这个对象就可以了呢？事实上的确如此，我们看看 `DexPathList` 的文档说明：

>A pair of lists of entries, associated with a {@code ClassLoader}.
One of the lists is a dex/resource path &mdash; typically referred
to as a "class path" &mdash; list, and the other names directories
containing native code libraries. Class path entries may be any of:
a {@code .jar} or {@code .zip} file containing an optional
top-level {@code classes.dex} file as well as arbitrary resources,
or a plain {@code .dex} file (with no possibility of associated
resources).</br>This class also contains methods to use these lists to look up
classes and resources.

大概的意思就是 `DexPathList` 的作用和 JVM 中的 `classpath` 的作用类似，JVM 根据 `classpath` 来查找类，而 Dalvik 利用 `DexPathList` 来查找并加载类。`DexPathList` 包含的路径可以是 `.dex `文件的路径，也可以是包含了 dex 的 `.jar` 和 `.zip` 文件的路径。

对于类加载器的分析先到这里，现在我们看看 `BaseDexClassLoader` 是如何加载类的。

### BaseClassLoader 加载类的过程 ###

我们知道，一个类加载器的入口方法是 `loadClass()`：

```java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }

            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }

        return clazz;
    }
```
这个方法封装了委派式加载机制，所以一般不重写。`CLassLoader` 的子类通常重写 `findClass()` 来定义自己的类加载策略。`BaseDexClassLoader` 也继承自 `ClassLoader`，因此我们就从 `findClass()` 方法来分析下 `BaseClassLoader` 加载类的过程。

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
       List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
       Class c = pathList.findClass(name, suppressedExceptions);
       if (c == null) {
           ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
           for (Throwable t : suppressedExceptions) {
               cnfe.addSuppressed(t);
           }
           throw cnfe;
       }
       return c;
}
```

这个方法的重点就是 `Class c = pathList.findClass(name, suppressedException)`，`pathList` 很熟悉对不对？它就是前面分析的 `BaseDexClassLoader` 中的 `DexPathList` 对象。这里 `BaseClassLoader` 把查找类的任务委托给了 `pathList`。

我们看看 `DexPathList` 的 `findClass()` 对象做了哪些事：

```java
public Class findClass(String name, List<Throwable> suppressed) {
       for (Element element : dexElements) {
           DexFile dex = element.dexFile;

           if (dex != null) {
               Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
               if (clazz != null) {
                   return clazz;
               }
           }
       }
       if (dexElementsSuppressedExceptions != null) {
           suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
       }
       return null;
}
```

方法的逻辑很清晰，它遍历了 `dexElements` 中的所有 `DexFile`，通过 `DexFile` 的 `loadClassBinaryName()` 方法加载目标类。可见，`dexElements` 又把查找类的任务委托给了 `DexFile`，看来 `DexFile` 这个对象的地位最低，大佬们都假装把活干完了，暗地里却把活丢给了它。前面说了，`DexFile` 对应着一个 dex 文件(或者包含 dex 文件的 zip 格式文件)，那么我们看看他是怎样在对应的 dex 文件中查找类的。

首先分析它的构造方法：

```java
private DexFile(String sourceName, String outputName, int flags, ClassLoader loader,
           DexPathList.Element[] elements) throws IOException {
       if (outputName != null) {
           try {
               String parent = new File(outputName).getParent();
               if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                   throw new IllegalArgumentException("Optimized data directory " + parent
                           + " is not owned by the current user. Shared storage cannot protect"
                           + " your application from code injection attacks.");
               }
           } catch (ErrnoException ignored) {
               // assume we'll fail with a more contextual error later
           }
       }

       mCookie = openDexFile(sourceName, outputName, flags, loader, elements);
       mFileName = sourceName;
       //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
}
```

估计你已经找到这个方法的重点了，没错，就是 `openDexFile()`，它最终会调用 `openDexFileNative()`，这家伙是个本地方法，我们就不[深究](http://blog.csdn.net/jltxgcy/article/details/50552674)了。它做的事就是把对应的 dex 文件加载到内存中，然后返回给 Java 层一个 `Object:mCookie` 用来标识本次和 Java 层的交互，后续的操作包括从 dex 文件中加载目标类和关闭 `DexFile` 对象释放资源都用到了这个 `mCookie`。此外，这个本地方法还做了一件重要的事，那就是优化 dex 并将其输出到指定文件夹。

在构造方法中 `DexFile` 就完成了 dex 文件的加载过程。现在我们回到 `DexFile` 对象的 `loadClassBinaryName()`：

```java
public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
       return defineClass(name, loader, mCookie, this, suppressed);
}

private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                    DexFile dexFile, List<Throwable> suppressed) {
       Class result = null;
       try {
           result = defineClassNative(name, loader, cookie, dexFile);
       } catch (NoClassDefFoundError e) {
           if (suppressed != null) {
               suppressed.add(e);
           }
       } catch (ClassNotFoundException e) {
           if (suppressed != null) {
               suppressed.add(e);
           }
       }
       return result;
}
```

终于看到了尽头，没错，class 对象在 java 层加载过程的尽头就是这个 `defineClass()` 方法。这个方法调用本地方法 `defineClassNative()` 从 dex 中查找目标类，如果找到了，就把这个代表这个类的 `Class` 对象返回。至此，Dalvik 虚拟机加载类的整个过程就结束了。现在我们回过头看看 `DexClassLoader()` 和 `PathClassLoader()`，这两个类加载器的唯一区别就是前者指定了优化后的 dex 文件的输出路径，后者没有指定。也就这一点差异造成了它们不同的使用场景：`DexClassLoader` 用来加载 .dex 文件以及包含 dex 文件的 .jar、.zip 和未安装的 .apk 文件，因此需要指定优化后的 dex 文件的输出路径；`PathClassLoader` 一般用来加载已经安装到设备上的 `.apk`，因为应用在安装的时候已经对 apk 文件中的 dex 进行了优化，并且会输出到 `/data/dalvik-cache` 目录下（android M 在这目录下找不到，应该是改成了 `/data/app/com.example.app-x/oat` 目录下），所以它不需要指定优化后 dex 的输出路径。下面用一张图来总结下安卓的类加载机制：

![](/img/in-post/post_android_load_class_dynamically/android-load-class-dynamically-2.png)

对这个模型作一下说明：

- `BootClassLoader` 是顶级的类加载器，这个类加载器在系统启动时就已经建立了，整个系统只有一个实例，它用来加载安卓核心类库。

- `PathClassLoader` 是每个应用进程的 Dalvik 虚拟机私有的类加载器，在应用启动时创建。它的 `DexPathList` 的 dex 加载路径是 `/data/app/apk-package-name-x/base.apk`（android M），用来加载我们已安装应用的 apk 中的 dex 文件。我们在应用中编写的的类默认是委托此类加载。

- Custom ClassLoader，这是开发人员自己实现的类加载器，通常是 `PathClassLoader` 或者 `DexClassLoader`。如果使用前者通常用来加载已经安装过的插件 apk 中的 dex 文件，如果使用后者通常用来加载 `.dex` 文件以及包含 dex 的 `.jar`、`.zip` 和 未安装的 `.apk` 文件。

- 我们可以做如下类比：

  - 把 **Dalvik** 类比于 **JVM**
  - 把 **dex** 文件 类比于 **class** 文件
  - 把 **dex 文件的路径（`DexPathList`）** 类比于 **类加载路径（`classpath`）** 。

相信现在大家对安卓的类加载机制有了大概的了解，为了避免文章篇幅过长，我打算把动态加载在安卓中的应用放在下一篇博客当中，感谢大家的阅读。

