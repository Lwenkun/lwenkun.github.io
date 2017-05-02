---
layout:     post
title:      "Java 8 方法引用"
subtitle:   "Method References"
date:       2017-03-23
author:     "lwenkun"
catalog:  true
header-img: "img/post-bg-java-8-method-references.png"
tags:
    - Java
    - Method Reference
---

# Java 8 方法引用

形如 `ClassName::methodName` 或者 `objectName::methodName` 的表达式，我们把它叫做方法引用（Method Reference）。可能你现在还理解不了，但是编译器足够聪明，或许你可以看看编译器是如何根据 “晦涩难懂” 的 Method Reference 来推断你的意图的。

## 方法引用的种类
方法引用有四种，分别是：

- 指向静态方法的引用
- 指向某个对象的实例方法的引用
- 指向某个类型的实例方法的引用
- 指向构造方法的引用

下面通过一些代码示例来和大家分享一下这四种引用使用场景，先定义一个 `Person` 类，后面我们会用到它：

```java
public class Person {
    private String name;
    
    private String getName() {
        return name;
    }
    
    public static int compareName(Person a, Person b) {
        return a.getName().compareTo(b.name);
    }
}
```

### 指向静态方法的引用

形式：`ContainingClass:staticMethodName`
 
如果要想实现一群人按名字排序，一般你会这么做：

```java
Arrays.sort(personArray, new Comparator() {
    @Override
    public int compare(Person a, Person b) {
        return a.getName().compareTo(b.getName());
    }
});
```

第二个参数很复杂，但是目的却很简单：实现 `Comparator` 的一个比较方法。我们自然会想——相信很多程序员都会这么想：有没有一种方法能够既简单明了又能无歧义的表达我们的意图？当然有，Java 8 引入的 lambda 表达式便很好的简化了我们的代码，瞧：

```java
Array.sort(personArray, (a, b) -> {
    return a.getName().compareTo(b.getName());
});
```

`(a, b)` 是 lambda 表达式的参数列表，箭头后的是方法体。编译器已经被训练得足够聪明：它知道 `(a, b)` 就是代表着要比较的两个 `Person` 对象。它在脑海里构想出了这个 lambda 表达式的 “原型”：

```java
new Comparator() {
    @Override
    public int compare(Person a, Person b) {
        return a.getName().compareTo(b.getName());
    }
}
```

机智的编译器从简化的代码中推断出了 `Comparator` 对象的 “原型”。但是程序员并不满足：还能不能进一步简化？当然可以，在不造成歧义的前提下可以无限地简化一个语法。我们注意到，`Person` 类有一个静态方法可以用来实现两个对象的比较，于是引入 **指向静态方法的引用**：

```java
Arrays.sort(personArray, Person::compareName);
```

`Person.compareName()` 方法有两个 `Person` 类型的参数，一个 `int` 类型的返回值，所以
`Person::compareName` 这个方法引用拥有两个 `Person` 类型的不确定量（在这里指的就是 `compareName()` 的两个参数；文章末尾有关于不确定量的具体解释）和一个 `int` 类型的返回值。而 `Comparator` 中待实现的方法 `compare()` 也有两个 `Person` 类型的参数和一个 `int` 的返回值。方法引用和 `Comparator` 中要实现的方法有足够的相似度，因此编译器将 `Person::compareName` 的不确定量和返回值与 `compare()` 的参数和返回值对应了起来，由此推断出我们的意图是：

```java
Arrays.sort(personArray, new Comparator() {
    @Override
    public int compare(Person a, Person b) {
        return Person.compareName(a, b);
    }
})

```

编译器成功地识别了程序员的意图。但是编译器的聪明程度也是有限的，假如方法引用所对应的静态方法返回值不是 `int` 而是 `boolean`，那么编译器无法将该方法引用还原为一个 `Comparator` 对象，因为它并不知道该返回什么 `int` 值。幸亏我们提供的静态方法引用有 `int` 类型的返回值，编译器还原时就可以把它返回。

### 指向某个对象的实例方法的引用
形式：`containingObject::instanceMethodName`

有一个比较器，它可以比较多种对象：

```java
public class Comparator() {
    public int comarePerson(Person a, Person b) {
        return a.getName().compareTo(b.getName());
    }
    
    public int compareDog(Dog a, Dog b) {
        return a.getName().compareTo(b.getName());
    }
    
    ......
}
```

上面的 `Dog` 类和 `Person` 类结构相似，所以代码就省略了。现在我们要根据一群狗的名字给它们排序，我们同样使用方法引用（当然，你依旧可以用 lambda 表达式），但是这次是 **指向某个对象的实例方法的引用**：

```java
Comparator aComparator = new Comparator(); // 实例化一个比较器
// aComparator::compareDog 是指向实例对象 aComparetor 的实例方法
// compareDog() 的方法引用
Arrays.sort(dogs, aComparator::compareDog); 
```

`compareDog()` 方法的两个参数是 `Dog` 类型，返回值为 `int`，也就是说 `aComparator::compareDog` 有两个 `Dog` 类型的不确定量和一个 int 类型的返回值。而 `Comparator` 中的 `compare()` 方法也是接受两个 `Dog` 对象，返回一个 `int` 值。根据这些线索编译器足够推断出我们的意图：

```java
Arrays.sort(dogs, new Comparator() {
    @Override
    public int compare(Dog a, Dog b) {
        return aComparator.compareDog(a, b);
    }
});
```

### 指向某个类型的实例方法的引用

形式：`ContainingClass::instanceMethodName`

```java
Stream.of("A", "is", "a", "dog").reduce("", new BinaryOperator() {
    @Override
    public String apply(String a, String b) {
        return a.toUpperCase() + b.toUpperCase();
    }
});
```

`reduce()` 方法是把某类型对象的集合降解为一个对象，过程是两两合并。它的第一个参数为初始值，第二个参数是一个接口，用来定义两两合并的具体策略。如果你暂时不懂 `Stream` 以及它的 `reduce()` 方法也没关系，我们只关注它的第二个参数。我们实现 `BinaryOperator` 接口的方法是把两个字符串变成大写后连接起来。代码看起来很臃肿，但是没有办法，我们没有合适的方法引用来将它简化（现有方法中没有能够将两个字符串先转换成大写然后连接的，当然你可以自己实现一个这样的方法，但没必要，因为这样会使得代码更加复杂），顶多可以转化成 lambda 表达式：

```java
Stream.of("A", "is", "a", "dog").reduce("", (a, b) -> {
        return a.toUpperCase() + b.toUpperCase();
    }
});
```

但是如果两两合并的方式是直接将两个字符串连接的话，情况就不同了。我们可以用 `String::concat` 这个 **指向某个类型的实例方法的引用** 来使代码更简洁：

```java
Stream.of("A", "is", "a", "dog").reduce("", String::concat);
```

编译器看看这行代码，愣了一下，不知道如何转化。还好，我们的编译器足够聪明，眼珠一转分析了下：我们要实现的方法有两个 `String` 的参数，一个 `String` 类型的返回值，而 `String::concat` 也有两个 `String` 类型的不确定量和一个 `String` 类型的返回值，其中一个不确定量是由于 `concat()` 这个方法是实例方法导致的，因为它必须通过一个 `String` 实例来调用；另一个不确定量为 `concat()` 方法的参数，它接受一个 `String` 类型的参数。以上信息足够编译器来推断我们的意图：程序员一定是想把两个参数和 `String::concat` 的两个不确定量对应起来，一个作为 `concat()` 方法调用的接收者，一个作为 `concat()` 方法的参数。这样一来上面的代码就转化成这样了：

```java
Stream.of("A", "is", "a", "dog").reduce("", new BinaryOperator() {
    @Override
    public String apply(String a, String b) {
        return a.concat(b);
    }
});
```

Wonderful ！程序员正是这么想的。有心的程序员可能会发出疑问：根据编译器的推断思路，对于任意一个方法引用，是不是只要它有两个 `String` 类型的不确定量和一个 `String` 类型返回值（当然类型要为 `String`）就可以作为参数传入 `reduce()` 了呢？于是程序员想了个方法来测试：

```java
Stream.of("A", "is", "a", "dog").reduce("", Test::test);

public class Test {
    public static String test(String a, String b) {
        return a.concat(b);
    }
}
```

程序员在 IDE 中写下上面的代码后，编译器没有错误提示 => 编译后通过 => 猜想正确。原因是 `Test` 的 `test()` 是一个静态方法，它有两个 `String` 类型的参数，一个 `String` 类型的返回值，也就是说 `Test::test` 有两个 `String` 类型的不确定量和一个 `String` 类型的返回值，因此编译器推断出程序员的意图是：

```java
Stream.of("A", "is", "a", "dog").reduce("", new BinaryOperator() {
    @Override
    public String apply(String a, String b) {
        return Test.test(a, b);
    }
});
```

### 指向构造方法的引用
形式：`ClassName::new`

```java
Stream.of("A", "is", "a", "dog").toArray(new IntFunction<String[]>() {
     @Override
     public String[] apply(int value) {
         return null; 
     }
});
```

`toArray()` 方法的作用是把数据流转换成一个数组，它接收的参数是一个 `IntFunction` 的接口实现对象。该接口有一个待实现的方法 `apply()`，它有一个 `int` 型的参数和一个 `String[]` 类型的返回值。我们依旧可以用方法引用来代替它，但这次我们用的是 **指向构造方法的引用**：

```java
Stream.of("A", "is", "a", "dog").toArray(String[]::new);
```

指向构造方法的引用可能较其他方法引用难理解一点，但是编译器却很机智的猜测到了程序员的意图：`String[]::new` 这个 指向构造方法的引用 有一个 `int` 类型的不确定量，即数组的长度；有一个 String[] 类型的返回值（构造方法不会有返回值，但是指向构造方法的引用却返回了一个该类型的实例）。因此编译器这样还原此方法引用：

```java
Stream.of("A", "is", "a", "dog").toArray(new IntFunction<String[]>() {
    @Override
    public String[] apply(int value) {
        return new String[value];
    }
})
```

## 总结
其实，JVM 本身并不支持指向方法引用，过去不支持，现在也不支持。Java 8 对方法引用的支持知识编译器层面的支持，虚拟机执行引擎并不了解方法引用。编译器遇到方法引用的时候，会像上面那样自动推断出程序员的意图，将方法引用还原成 **接口实现对象**，或者更形象地说，就是把方法引用设法包装成一个接口实现对象，这样虚拟机就可以无差别地执行字节码文件而不需要管什么是方法引用了。

需要注意的是，方法引用是用来简化接口实现代码的，并且凡是能够用方法引用来简化的接口，都有这样的特征：有且只有一个待实现的方法。这种接口在 Java 中有个专门的名称： **函数式接口**。当你用试图用方法引用替代一个非函数式接口时，会有这样的错误提示： <font color="red">xxx is not a functional interface</font>。

文章中很多地方提到过 **不确定量** 这个名词，这个名词并非官方的，而是我为了更好地说明问题而引入的，它的意思是： **方法引用在转换成接口实现对象的过程中需要确定的变量**。如 **方法引用所对应方法的接受者**（ **指向静态方法的引用** 和 **指向某个对象的实例方法的引用** 没有该不确定量，因为前者不需要接受者，而后者接受者已经确定），以及 **方法引用所对应方法的参数** 都属于 **不确定量**。方法引用的 **不确定量** 以及返回值需要和接口中的待实现方法的参数和返回值对应起来，这样编译器才能成功将其还原。

## 感谢阅读
写这篇文章的目的是让大家对方法引用有一个更好的理解，同时也作为个人的学习记录，但由于理解和水平有限，差错在所难免，还请大家不吝赐教。最后，感谢大家的阅读。

**参考文章:**  
[Method References](http://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html)  
[:: (double colon) operator in Java 8](http://stackoverflow.com/questions/20001427/double-colon-operator-in-java-8)  