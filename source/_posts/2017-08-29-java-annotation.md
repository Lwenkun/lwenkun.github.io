---
layout:     post
title:      "Java 注解学习总结"
subtitle:   "Java 注解学习总结"
date:       2017-08-29
catalog:  true
author:     李文坤
header-img: "img/post-bg-operating-system-conclusion.jpg"
tags:
    - Java
---

# Java 注解学习
## 前言
注解是在 JDk 5 时引入的，注解可以提供一些元数据，它们能为所注解的代码提供一些不属于程序本身的数据，从而可以对程序提供一些辅助功能。这些辅助功能包括编译时检查（错误和警告）、编译时和部署时的代码加工，以及运行时的代码处理。这三种注解分别被称为源码级别的注解（这些注解主要是在编辑代码时为程序员提供错误信息）；编译时注解：这种类型的注解一般需要配合注解处理器框架使用，可以在编译器编译项目时生成一些文件，包括 .java 源文件和 .xml 配置文件等；运行时注解：在程序运行的时候为程序提供一些功能，一般配合反射框架使用。
<!-- more -->
## 注解的定义和使用
在细讲这三种注解的区别前，先简单说下注解的定义和使用。

### 使用
注解的使用想必大家很熟悉了，如最常见的 Override 注解，这种注解没有配置参数，被称为标记注解：

```java
@Overide
int hashCode(){};
```

有参数的注解常见的有 SuppressWarnings 注解，这个注解用来关闭编译器的警告，如关闭 unchecked（未受检）警告和 deprecation（使用了已弃用方）警告：

```java
@SuppressWarnings("unchecked")
useDeprecatedMethod() {
    deprecatedMethod();
}
```
这个注解带有配置参数，并且参数值可以有多个，表示需要关闭多种类型的警告，比如需要同时关闭 unchecked 和 deprecation 警告：@SuppressWarnings({"unchecked", "deprecation"})。

### 定义

Java 类库中有很多预定义好的注解，如上面那两个，用户也可以自己定义注解，注解的定义是这样的：

```java
@Target(ElementType.Type)
@Documented
@@Inherited
@Repeatable
@Retention(RetentionPolicy.SOURCE)
public @interface Person {

    enum GENDER {MALE, FEMALE}

    String name();
    int id() default -1;
    GENDER gender() default GENDER.MALE;
    String[] hobit();
}
```
上面的例子中，定义了一个名为 Person 的注解。注意 Person 前面的 @interface，它是来声明注解用的，和 class 和 enum 是一个级别的东西，所有的注解都会自动继承 Annation 接口。我们发现这个注解定义上也有多个注解，他们被称为“元注解”，作用是注解其他的注解。那么它们含义是什么呢？首先看 Target 注解，它用来说明我们定义的注解的总用范围，它含有一个枚举类型的参数，可以取以下值：

- ElementType.ANNOTATION_TYPE 作用于注解类型上（元注解就是这种类型，而且这种类型的注解可以自注解，即自己作用在自己身上）。
- ElementType.CONSTRUCTOR 作用于构造器。
- ElementType.FIELD 作用于类属性。
- ElementType.LOCAL_VARIABLE 作用于局部变量。
- ElementType.METHOD 作用于方法。
- ElementType.PACKAGE 作用于包的声明。
- ElementType.PARAMETER 作用于方法参数。
- ElementType.TYPE 作用于接口，类和枚举。

@Documented 注解用来说明 javadoc 工具是否为该注解生成文档，因为注解默认是不生成文档的。@Inherited 注解表示的是该注解可从父类传给子类，比如用 Person 去注解一个类，那么这个类的子类即使没有 Person 所注解，但是由于 Person 的可继承性，子类也同样被隐式的被 Person 注解了。@Repeatable 表示该注解可以重复注解在某一个元素上，这个注解是在 java8 引入的。@Retention 注解用来说明该注解的级别，是源码级别的（编辑时注解），还是类级别的（编译时注解，这种注解会被编译器保留，但会被 JVM 忽略），运行时级别的（被 JVM 保留，并且在运行时被使用）。

@Person 含有四个配置参数，它们的声明看起来像是方法。需要注意的是注解的配置参数的类型只可以是基本数据类型、枚举、Class、String 和以上类型的数组类型。当注解只含有一个配置参数时，一般参数名直接用 "value"，因为注解本身的名称就很好的解释了 value 的意义。现在看看如何用 @Person 注解：

```java
@Person(name="Chance", id=16, gender=GENDER.MALE", hobit={"music", "movie", "game"})
public class Chance {
}
```
相信大家一目了然，就不细讲了。需要说明的是，这里使用名值对的形式表示注解。如果只有一个配置参数，那么写配置参数的时候可以将参数名称省略掉，就像 @SupperssWarnings 一样。并且要注意的是，注解的某些参数的配置参数后可以通过 default 关键字指定默认值，如果没有指定，那么使用该注解时一定要填写该参数值，否则编译器会报错。也就是说，没有默认值的参数是必填项。


## 源码级别的注解
之所以称之为源码级别的注解，就是注解信息仅仅会在源码中保留，不会编译到 class 文件中。这和它的作用相符：因为它仅仅是为编辑器提供信息而已。这种注解常见的如 @Override、@SuppressWarnings、@Deprecated 等。这种注解比较简单，就不细说了。

## 编译时注解
这种类型的注解要配合注解处理器来使用，什么是注解处理器？为什么要注解处理器？注解处理器顾名思义就是对注解进行处理的工具，为什么需要注解处理器呢？前面说了，注解其实对被注解的代码不会有任何影响，也就是说它们不会被项目中的代码识别，那么要想它们发挥作用，必定是需要能够识别它们的东西。编辑时注解器靠的就是编辑器来识别，而编译时处理器靠的便是注解处理器了，下面要说的运行时注解靠的就是反射框架了。如何定义注解处理器呢？


## 运行时注解
运行时注解可以反射框架来实现，比如对于上面那个例子，假设 @Person 注解是运行时注解，那么可以定义一个注解的运行时解析类在运行时获取注解的信息：

```java
public class AnnotationUtil {
   public void inject(Object target) {
       Class cls = target.getClass();
       if (cls.isAnnotationPresent(Person.class)) {
       	Person personAnnotation = cls.getAnnotation(Person.class);
       	System.out.print(personAnnotation.toString());
        //  System.out.print(personAnnotation.gender());
        //  System.out.print(personAnnotation.hobit());
        //  System.out.print(personAnnotation.id());
        //  System.out.print(personAnnotation.name());
       }
   }
}
```
然后这样定义 Chance：

```java
@Person(name="Chance", id=16, gender=GENDER.MALE", hobit={"music", "movie", "game"})
public class Chance {
	
	public Chanece() {
		AnnotationUtil.inject(this);
	}

	public static void main(String[] args) {
		Chance c = new Chance();
	}
}
```

然后打印出来的结果是这样的：

```java
@com.company.MyAnnotation(gender=MALE, id=16, name=Chance, hobit=[music, movie, game])
```
通过反射就可以拿到注解信息。如果注解作用在字段或者方法上，也是通过同样的方法获取注解的信息的，这里就不再说了。拿到注解信息后想干什么那就看自己的喜好了，很多第三方框架就是通过这种方法来获取注解信息然后实现一些看似很炫酷的功能。比如 Retrofit，刚接触的时候很疑惑，为什么在接口方法上使用注解就可以 “实现” 该接口呢？其实它就是在运行时获取注解信息然后根据这些信息通过动态代理的方式实现该接口。