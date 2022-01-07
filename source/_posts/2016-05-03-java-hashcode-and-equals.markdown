---
layout:     post
title:      "浅析 hashcode() 和 equals()"
subtitle:   "hashcode() & equals()"
date:       2016-05-03
catalog:  true
author:     "lwenkun"
header-img: "img/post-bg-java-hashcode-and-equals.png"
tags:
    - Java
---

# 浅析 hashcode() 和 equals()

在 Java 中，`equals()` 的默认实现的是判断两个引用变量是否指向同一内存空间，即这两个引用变量是否是对同一实例的引用。而 `hashcode()` 默认实现与 `native` 方法相关，我的猜测是和这两个引用所指对象的内存地址有关，事实上的确是这样。为什么我的猜测会是正确的呢？我们来分析原因。

<!-- more -->

Java 文档对子类重写这两个方法的要求是：

1. 要么两个方法都重写，要么都不重写；
2. 如果两个对象的通过 `equals` 方法比较返回 `true`，那么这两个方法的 `hashcode` 必须相等；  
3. 如果这两个对象的 `hashcode` 相等，这两个对象不一定 `equals` 比较后返回 `true`。

看看 `String` 的 `hashcode()` 方法和 `equals()` 方法：

- hashcode() :

```java
@Override public int hashCode() {
        int hash = hashCode;
        if (hash == 0) {
            if (count == 0) {
                return 0;
            }
            for (int i = 0; i < count; ++i) {
                hash = 31 * hash + charAt(i);
            }
            hashCode = hash;
        }
        return hash;
}
```

- equals()：

```java
@Override public boolean equals(Object other) {
        if (other == this) {
          return true;
        }
        if (other instanceof String) {
            String s = (String)other;
            int count = this.count;
            if (s.count != count) {
                return false;
            }
            
            if (hashCode() != s.hashCode()) {
                return false;
            }
            for (int i = 0; i < count; ++i) {
                if (charAt(i) != s.charAt(i)) {
                    return false;
                }
            }
            return true;
        } else {
            return false;
        }
}
```

对于 `String`， 我们可以看到：

- `equals()` 方法的 “关键变量”（比较的依据）是字符串中的每个字符。但是为了程序效率，这个方法一开始并不会逐个比较两个字符串的字符，而是先比较它们长度是否相等，再看它们的 `hashcode()` 是否相等（按照规范，如果两个对象相等，它们的 `hashcode()` 一定相等），最后才出 “杀手锏”，逐个地比较它们的字符。
- `hashcode()` 方法返回的是对每个字符的 ascii 码进行加权求和。

从文档的规范和 `String` 中两个方法的实现我们可以暂时可以推导出这样的结论：`hashcode()` 方法的返回值一定是通过对 `equals()` 中的关键变量进行某种函数变换得到的（简单点说就是和关键变量有关），只有这样才能保证 **equals() 返回 ture** => **关键变量相等** => **hashcode() 返回值相等**。

对于我们自定义的类，如果要重写这两个方法，应该首先重写 `equals()`。因为根据第二点要求，`hashcode()` 方法的实现应该是建立在 `equals()` 方法之上的：在重写 `equals()` 方法之后，我们再去保证对于 `euqals()` 比较返回 `true` 的两个对象，如何让它们的 `hashcode()` 返回值相等。

例如，定义一个 `People` 类：

```java
public class People {
    private int age;
    private String name;
    private String hometown;
    ......
}
```
如果有这样的定义：对于 `People` 的两个对象 `a` 和 `b`，如果它们的 `name` 相等我们就认为这两个对象相等。那么 `People` 的 `equals()` 方法就应该是这样实现的：

```java
public boolean equals(Object other) {
    if (！other instanceof People)
        return false;

    if(hashcode() != other.hashcode())
        return false;
    
    return name == null ? other.name == null : name.equals(other.name);
}
```

`equals()` 的 “关键变量” 是 `name`，所以 `hashcode()` 的返回值必须是通过对 `name` 的某种函数变换得到的。

```java
public int hashcode() {
     // 在 String 中，hashcode() 返回值就是通过对字符串的函数变换得到的，
     // 因此这里直接返回 name 的 hashcode 也能保证 People 的 hashcode() 返回值是
     // 通过对 name 的函数变换得到的
     return name.hashcode();
 }
```

按照上面的实现，我们可以保证，如果 `a.equals(b) == true`，一定有 `a.hashcode() == b.hashcode()`。

如果我们按照下面这样实现 `hashcode()` 会怎样呢？

```java
public int hashcode() {
   return hometown.hashcode();
} 
```

在上面的实现中，`hashcode()` 的返回值并不和 “关键变量” 有关，我们看看这会导致什么问题：假如有两个人 `name` 相等，`hometown` 不相等，那么它们通过 `equals()` 比较会返回 `true`，但是由于 `hometown` 的不相等会直接导致它们 `hometown.hashcode()` 的返回值不相等，进而导致它们自己的 `hashcode()` 返回值不相等，这显然违背了 java 的规范。

现在考虑判断两个人相等另一种定义：如果两个 `People` 对象的 `name` 和 `hometown` 都相同，那么它们相等。

那么对于 `equals()` 方法来说，中规中矩的实现是这样的：

```java
public boolean equals(Object other) {

    if (! other instanceof People)
        return false;
        
    return (name == null ? 
               other.name == null : name.equals(other.name))
        && (hometown == null ?
               other.hometown == null : hometown.equals(other.hometown));
}
```
而对于 `hashcode()` 方法来说，就可以这样实现（不是最好的做法，但是符合 java 规范）：

```java
public int hashcode() {
     return name.hashcode() + hometown.hashcode();
}
```
下面的实现也是可以的（同样不建议这样做）：

```java
public int hashcode()  {
   // 返回值只与其中的一个关键变量有关，
   // 虽然这样容易导致 hashcode 的聚集，但是理论上也符合 java 的规范
   return name.hashcode() ;
}
```
或者（推荐的做法）:

```java
public int hashcode() {
    int k = 17; // 任意的起始值
    k = k * 31 + name.hashcode();
    k = k * 31 + hometown.hashcode();
    return k;
}
```

但是这样就绝对是错误的（与无关变量 `age` 有关）：

```java
public int hashcode() {
     return homtown.hashcode() + age;
}
```

对于上面这种实现，虽然保证了和关键变量 `hometown` 有关，但是却引入 `age` 这个非关键变量（无关变量），我们看看这样做有什么问题：有两个 `People` 对象，它们的 `name` 和 `hometown` 相同，但 `age` 不相等，那么它们通过 `equals()` 比较会返回 `true`， 但它们 `hashcode()` 的返回值却不相等，因为 `age` 的不相等导致 `hometown.hashcode() + age` 不相等，进而导致了这两个对象 `hashcode()` 的返回值不相等。

因此对于开头的结论，我们还需要进行完善：`hashcode()` 方法的返回值一定是通过对 `equals()` 中的关键变量进行某种函数变换得到的（简单点说就是和关键变量有关），并且不能和非关键变量（无关变量，即 `equals()` 方法中用不到的变量）有关。

现在回到开头的问题：为什么 `hashcode()` 方法的默认实现与地址相关？因为在 `equals()` 的默认实现中，关键变量是两个引用变量所指向的内存地址，因此 `hashcode()` 方法返回值必须和对象的内存地址有关。

## 感谢阅读 ##
由于只是说明 `hashcode()` 和 `equals()` 的内在联系，因此文章中有的例子并是 `hashcode()` 和 `equals()` 的最佳实现（但还是满足 java 规范，对于最佳实现可以参考 《Effective Java》 这本书）。另外如果有什么不对的话，还望大家不吝赐教。

参考书籍：《Effective Java》
