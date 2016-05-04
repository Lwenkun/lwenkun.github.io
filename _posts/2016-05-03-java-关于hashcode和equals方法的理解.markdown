---
layout:     post
title:      "hashcode() 和 equals() 的理解"
subtitle:   "hashcode() & equals()"
date:       2016-05-03
author:     "lwenkun"
header-img: "img/in-post/post_java_equals_and_hashcode/bg.jpg"
tags:
    - java
---
# hashcode() 和 equals() 的理解 #

`equals()` 的默认实现的返回值根据的是两个对象是否占用同一内存空间，即这两个对象是否其实是同一实例的两个引用。

`hashcode()` 默认实现是一个和 `native` 方法相关的，我们可以不管他们，我的猜测是和这两个引用的所指对象的地址相关，事实上的确是这样，为什么我的猜测会是正确的呢，后面我会说原因。

java 文档对子类重写这两个方法的要求是：

1.要么两个方法都重写，要么都不重写；
2.如果两个对象的通过 `equals` 方法比较返回 `true`，那么这两个方法的 `hashcode` 必须相等；
3.如果这两个对象的 `hashcode` 相等，这两个对象不一定 `equals` 比较后返回 `true`.

看看 `String` 的 `hashcode()` 方法和 `equals()` 方法：

`hashcode()` :

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

`equals()` 方法：

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
            // TODO: we want to avoid many boundchecks in the loop below
            // for long Strings until we have array equality intrinsic.
            // Bad benchmarks just push .equals without first getting a
            // hashCode hit (unlike real world use in a Hashtable). Filter
            // out these long strings here. When we get the array equality
            // intrinsic then remove this use of hashCode.
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

我们可以看到：

`equals()` 方法的“核心条件”（两个对象的引用相等的条件）是比较两个字符串内容是否一样。但是为了程序效率，这个方法一开始并不会直接比较两个字符串的内容，而是先比较他们长度是否相等，再看他们的 `hashcode()` 是否相等（因为按照规范，如果两个对象相等，他们的 `hashcode()` 一定相等），最后才出“杀手锏”，逐个地比较他们的内容是否相等。

`hashcode()` 方法的实现是对每个字符的 ascii 码进行加权求和。

如果我们要重写这两个方法，应该首先重写前者。因为为了满足上面说的第二点要求，`hashcode()` 方法的实现应该与 `equals()` 方法的“核心条件”涉及的变量有关且只能与这些变量有关（注意：应该至少与其中的一个变量有关），用数学语言解释就是 hashcode 应该是 `equals()` 核心条件涉及变量中的一个或多个的函数，并且不能是其他无关变量（核心条件不涉及的变量）的函数。为什么可以用数学中的函数概念来解释 `hashcode()` 应该满足的要求？因为根据函数的性质，相同自变量有相同的因变量，而 `hashcode()` 正好需要满足这种性质。

为什么能推导出这种关系?我们用例子来说明。

例如， 对于 `String` 来说，两个字符串是否相等（`equals()` 是否返回 `true`）的核心条件是这两个字符串的内容是否相等，所以字符串的 hashcode 应该是关于这个字符串内容的函数。`String` 的 `hashcode()` 方法的确满足这个条件，因为如果两个 `String` 对象的内容相同，那么他们的所有字符的加权和一定相等，hashcode 也就相等。

又例如，定义一个类 `People`，该类有两个对象 `a` 和 `b`，这个类的 `equals()` 方法的“核心条件”是他们的名字相等，那么 `equals()` 方法的代码就可以这样实现：

```java
public boolean equals(Object other) {
  if (! other instanceof People) {

         if(a.hashcode() != b.hashcode()) {
               return false;
        }
    
        if(a.name.euqals( b.name)) {
            return true;
        }  
       
        return false;
  } else {
     return false
  }
}
```

我们看到 `equals()` 的“核心条件”是两个人的名字是否相同，所以我们的 `hashcode()` 方法应该保证两个名字相同的人的 `hashcode()` 返回值也一样，即我先前说的 hashcode 是 `name` 的函数。既然判断名字相同沿用了 `name` 的 `equals()` 方法（相等的核心条件与 `name` 相关），所以 `hashcode()` 方法沿用 `name` 的 `hashcode()` 方法也是可以的（满足 hashcode 是 `name` 的函数这个要求）。所以 `hashcode()` 方法可以这样实现：

```java
//这个方法沿用了 `name` 的 `hashcode()` 方法，则 hashcode
//会是关于 `name` 的“函数”，因此这里的实现满足要求
public int hashcode() {
     return name.hashcode();
 }
```

按照上面的实现，如果两个人“相等”，两个人的 `hashcode()` 也一定会相等。

但是这样就不正确了：

```java
public int hashcode() {
   return hometown.hashcode();
} 
```

 或者用一种更一般形式来表示：

```java
public int hashcode() {
   //`fun()` 是关于自变量 `var` 的函数，并且 `var` 不是 `name` 的函
   //数，所以 `fun()` 不会是关于 `name` 的函数，从而 hashcode
   //不会是关于 `name` 的函数,因此不满足条件
   return fun(var);
} 
```

因为两个人是否“相等”的“核心条件”是 `name` 是否相等而不与他们的 `hometown` 有关，所以如果按照上面的实现，假如有两个人 `name` 相等，但是他们的 `hometown` 不相等，就会导致 `equals()` 为 `true` 时，两个人的 hashcode 不相等，这就违背了 java 的规范，虽然 `hashcode()` 这样实现并不会影响 `equals()` 的结果（因为 `equals()` 方法的返回值并不依赖于两个对象的 hashcode 相等）。

现在考虑另外一种两个人是否相等的定义：如果两个人的 `name` 和 `hometown` 都相同，那么他们相等。

那么中规中矩的实现是这样的：

```java
public boolean equals(Object other) {

   if(other instanceof People) {
        if(! a.hashcode == b.hashcode()) {
             return false;
        }
        
        if(a.name.equals(b.name) && a.hometown.equals(b.hometown)) {
             return true;
       }

       return false;

    }  else {
        return false;
    }

}

public int hashcode() {
     return name.hashcode() + hometown.hashcode();
}
```

下面的实现 `hashcode()` 也是正确的（只与其中的一个变量有关，并且 hashcode 是那个变量的函数）：

```java
public int hashcode()  {
   return name.hashcode() ;
}
```

或者

```java
public int hashcode() {
    return hometown.hashcode(); 
}
```

但是这样就是不对的（与别的变量 `age` 有关，是 `age` 的函数）：

```java
public int hashcode() {
     return homtown.hashcode() + age.hashcode();
}
```

或者以一种更一般的形式来表现：

```java
public int hashcode() {
   // `fun1()` 是关于 `hometown` 的函数，`fun2()` 是关于 `age` 的函数，`age` 不是关于 `name` 的函数
   return fun1(hometown) + fun2(age);
}
```

像上面这样实现的话，如果有两个对象，它们的 `name` 和 `hometown` 相同，则它们通过 `equals()` 比较返回 `true`， 但是它们 `hashcode()` 的返回值却可能不相等，因为它们的 `age` 可能不相等（即他俩的 `age`通过 `equals()` 比较返回 `false`），从而它们的 `age.hashcode()`（或者 `fun2()`）的返回值不一定相等。

现在回到开头的问题，为什么我会猜测 `hashcode()` 方法的默认实现与地址相关，相信大家现在应该也都明白了吧，因为 `equals()` 默认实现的返回值根据的是两个对象的引用是否是共用同一内存地址，即核心条件涉及的变量是对象的内存地址，所以根据我们的总结，`hashcode()` 方法就应该是是对象的内存地址的函数，所以必须与内存地址有关。

以上仅仅是我个人的理解，如果总结得不准确还请大家留言告知。
