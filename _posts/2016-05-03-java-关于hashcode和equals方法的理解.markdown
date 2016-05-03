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

`equals()` 默认的实现是两个对象的引用指向，也即两个引用其实都是同一个对象的引用，换句话说也就是两个引用指向同一内存地址。
`hashcode()` 默认实现是一个和 `native` 方法相关的，我们可以不管他们，我的猜测是和这两个引用的所指对象的地址相关，事实上的确是这样，为什么我的猜测会是正确的呢，后面我会说原因。
java 文档对这两个方法的重写要求是：
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

`equals()` 方法的“核心条件”（两个对象的引用相等的条件）是比较两个字符串内容是否一样。但是为了程序效率，这个方法一开始并不会直接比较两个字符串的内容，而是先比较他们长度是否相等，再看他们的 `hashcode()` 是否相等（因为按照规范，如果两个对象相等，他们的 `hashcode()` 一定相等，所以可以在这里进行一次判断），最后才出“杀手锏”，逐个地比较他们的内容是否相等。

`hashcode()` 方法的实现是对每个字符的 ascii 码进行加权求和。

如果我们要重写这两个方法，应该首先重写前者。因为 `hashcode()` 方法的实现应该与 `equals()` 方法的“核心条件”涉及的变量有关且只能与这些变量有关（注意：应该至少与其中的一个变量有关）。就拿 `String` 来说，两个字符串是否相等的核心条件是这两个字符串的内容是否相等，所以字符串的 `hashcode` 应该是关于这个字符串内容的“函数”（数学意义上的概念），也就是说如果这两个字符串的内容相等，那么我们就应该让这两个字符串有相等的 `hashcode`，也即 `hashcode()` 的返回值相等。`String` 的 `hashcode()` 就满足这个条件，因为如果两个 `String` 的内容相同，那么他们的所有字符的加权和一定相等，`hashcode` 也就相等，这里 `hashcode` 可以看作是 `String` 的内容的函数，自变量是 `String` 的内容。

又例如，定义一个类 `People`，该类有两个对象 `a` 和 `b`，这个类的 `equals()` 方法的“核心实现”是他们的名字相等，那么 `equals()` 方法的代码可以这样实现：

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

我们看到 `equals()` 的“核心条件”是两个人的名字是否相同，所以我们的 `hashcode()` 方法应该保证两个名字相同的人的 `hashcode()` 返回值也一样，即我先前说的 `hashcode` 是 `name` 的函数。既然判断名字相同沿用了 `name` 的 `equals()` 方法，所以 `hashcode()` 方法沿用 `String` 的 `hashcode()` 方法也是可以的。所以 `hashcode()` 方法可以这样写：

```java
//这个方法是关于name的“函数”
public int hashcode() {
     return name.hashcode();
 }
```

这样的话，如果两个人“相等”，两个人的 `hashcode()` 也一定会相同。

但是这样就不正确了：

```java
public int hashcode() {
   return hometown.hashcode();
} 
```

 或者用一种更一般形式来表示：

```java
public int hashcode() {
   //fun() 是关于hometown自变量var的函数，并且var不是name 的函
   //数，fun(var)的返回值是int 类型
   return fun(var);
} 
```

因为两个人是否“相等”的“核心条件”是名字是否相等而不与他们的家乡有关。如果这样实现 `hashcode()`，那么如果有两个人名字相等，但是他们的 `hometown` 不相等，那么会导致 `equals` 为 `true` 时，两个人的 `hashcode()` 不一样。虽然 `hashcode()` 这样实现不会影响 `equals()` 的结果（因为 `equals()` 方法的返回值并不依赖两个对象的 `hashcode()` 是否相等），但是这违背了 java 的规范。

现在考虑另外一种判断两个人是否相等的定义：如果两个人的名字和家乡都相同，那么他们相等。

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

下面的实现 `hashcode()` 也是正确的（与其中的一个变量有关）：

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

但是这样就是不对的（与别的变量有关）：

```java
public int hashcode() {
     return homtown.hashcode() + age.hashcode();
}
```

或者以一种更一般的形式来表现：

```java
public int hashcode() {
   //fun1()是关于hometown的函数，fun2()是关于fun2的函数
   return fun1(hometown) + fun2(age);
}
```

因为这样的话就出现两个对象 `equals()` 返回 `true`， 但是 `hashcode()` 的返回值不同，他们的 `age.hashcode()`（或者 `fun2()`）的返回值不一定相等。

回到开头的问题，为什么我会猜测即 `hashcode()` 方法的默认实现与地址相关，相信大家现在应该也都明白了吧，因为 `equals()` 方法的默认实现是判断两个对象的引用是否指向同一对象（或者说同一内存地址），那么根据我们的总结，`hashcode()` 方法也应该是是内存地址的函数，所以必须与内存地址有关。

以上仅仅是我个人的理解，如果总结得不准确还请大家留言告知。
