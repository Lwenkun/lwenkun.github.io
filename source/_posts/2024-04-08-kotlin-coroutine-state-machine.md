---
layout:     post
title:      探究 Kotlin 协程的状态机
subtitle:  
date:       2024-04-08
author:     "Chance"
catalog:  true
tags:
    - Kotlin
    - 协程
---

# 前言

Kotlin 中的协程是无栈协程（话说 Kotlin 能实现有栈线程吗🤔），网上很多文章都说无栈协程一般都是通过状态机实现的，刚开始听到这个状态机的时候觉得有点玄乎，今天反编译一下 Kotlin 代码，看看这个状态机到底是个什么鬼。

# 反编译 Kotlin

```kotlin
fun main() {
    runBlocking {
        val result = fun1()
        println(result)
    }
}

suspend fun fun1(): Int {
    var localInt = 0

    localInt += fun2()

    localInt += fun3()

    return localInt
}

suspend fun fun2(): Int {
    return 1
}


suspend fun fun3(): Int {
    delay(1000)
    return 1
}
```
<!-- more -->

这是一段使用了协程的 Kotlin 代码。在 `main` 方法中，通过 `runBlocking` 方法开启协程，协程的逻辑很简单，调用 `fun1()` ，然后将其结果打印出来。重点是 `fun1()` 函数，`fun1()` 是一个 `suspend` 方法，它定义了一个局部变量 `localInt`，然后依次执行了 `fun2()` 和 `fun3()` 并将结果累加到 `localInt` 中，最后将 `localInt` 返回。

其中 `fun2()` 是一个披着 `suspend` 外衣的普通方法，IDE 中会出现 warning 提示说 `suspend` 关键字是多余的，暂时保留它，看看最后会编译成什么样。`fun3()` 内调用了 `delay() `，`delay()` 方法是 `suspend` 的元凶之一，调用链上游的方法都因为它是 `suspend`，才都变成 `suspend`。

例子很简单，但涵盖了协程运行时的几个重要的场景：协程的启动，协程中调用 `suspend` 方法，`suspend` 方法中调用普通方法，`suspend` 方法中调用 `suspend` 方法。接下来将以上代码编译后再反编译为 Java 代码。

（反编译 Kotlin 代码是没法使用传统的反编译工具来完成的，需要在 IDEA 中打开 Kotlin 字节码文件，然后点击 工具 -> Kotlin -> 反编译为  Java 来完成。）

## main

先看 `main` 方法：

```java
public static final void main() {
      BuildersKt.runBlocking$default((CoroutineContext)null, (Function2)(new Function2((Continuation)null) {
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            Object var10000;
            switch (this.label) {
               case 0:
                  ResultKt.throwOnFailure($result);
                  Continuation var4 = (Continuation)this;
                  this.label = 1;
                  var10000 = TestKt.fun1(var4);          // fun1()
                  if (var10000 == var3) {
                     return var3;
                  }
                  break;
               case 1:
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break;
               default:
                  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            int result = ((Number)var10000).intValue();
            System.out.println(result);
            return Unit.INSTANCE;
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
            return (Continuation)(new <anonymous constructor>($completion));
         }

         @Nullable
         public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
            return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
         }

         // $FF: synthetic method
         // $FF: bridge method
         public Object invoke(Object p1, Object p2) {
            return this.invoke((CoroutineScope)p1, (Continuation)p2);
         }
      }), 1, (Object)null);
   }
```

没想到 Kotlin 中的几行代码，反编译为 Java 后代码有这么长。反编译器一般都比较”死板“，有些地方乍看晦涩难懂，但仔细研究一下就知道，它用蹩脚的代码展示了极为简单的逻辑。

`runBlocking$default()` 接收四个参数，其他几个参数看不懂，但第二个参数貌似有点东西。第二个参数是一个 `Function2` 对象，`Function2` 是 Kotlin 库中的一个接口，有一个 `invoke()` 方法，定义如下：

```kotlin
public interface Function2<in P1, in P2, out R> : Function<R> {
    /** Invokes the function with the specified arguments. */
    public operator fun invoke(p1: P1, p2: P2): R
}
```

第二个参数其实是一个继承自 `kotlin.coroutines.jvm.internal.SuspendLambda` 的对象，只不过它同时实现了 `Function2` 接口。之所以反编译器把它编译成 `Function2`  对象，是因为 `runBlocking$default()` 方法签名中，第二个参数就是 `Function2` 类型的。 `SuspendLambda` 的继承链是：`SuspendLambda` -> `ContinuationImpl` -> `BaseContinuationImpl` -> `Continuation`。

既然 `runBlocking$default()` 方法接受的是 `Function2` 类型的参数，那它应该只和 `Function2` 的方法打交道，因此从 `invoke()` 方法入手：

```kotlin
@NotNull
public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
   return (Continuation)(new <anonymous constructor>($completion));
}

@Nullable
public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
     return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
}
```

它调用了 `create()` 方法创建了一个对象，紧接着调用这个对象 `invokeSuspend()` 方法。这里的 `undefinedtype` 其实就是 `Function2` 参数自身的实际类型，`anonymous constructor` 其实就是它自身的构造方法。可能是匿名类的缘故，反编译器没法表示出来。

`Function2` 对象在自己的 `inovke()` 方法中创建了另一个同类型的对象，然后调用了这个对象的 `invokeSuspend()` ，那它为什么不直接调用自己的 `invokeSuspend()` 方法？这点我没搞明白，也许是 Kotlin 编译器的遵循了一些死板的编译规则导致的，先不管，把注意力转移到 `invokeSuspend()` 方法上来。

在 `invokeSuspend()` 里面，最终调用了 `fun1()` 方法。但不是直接调用，而是套了一个 `switch case` 判断。等等，`switch case` ，这不就是实现一个状态机的典型语法吗？如果它是状态机，那 `label` 应该就是这个状态机的状态了。再定睛一看，在 `case 0`  块中，`label` 被置为 1 了，状态转移，好吧，状态机实锤了。也就是说，Kotlin 中 `runBlocking` 方法的 block 里面的代码，被套在了状态机里执行：label 为 0 的时候，执行的是 `fun1()`；`lable` 为 1 的时候，执行的是 `System.out.println()`。

有点意思。我现在有点迫不及待地想看看 `fun1()` 的反编译结果。

## fun1

```kotlin
   @Nullable
   public static final Object fun1(@NotNull Continuation var0) {
      Object $continuation;
      label27: {
         if (var0 instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)var0;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label27;
            }
         }

         $continuation = new ContinuationImpl(var0) {
            int I$0;
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return TestKt.fun1((Continuation)this);
            }
         };
      }

      Object var10000;
      int localInt;
      int var2;
      Object var3;
      label22: {
         Object $result = ((<undefinedtype>)$continuation).result;
         Object var6 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
         switch (((<undefinedtype>)$continuation).label) {
            case 0:
               ResultKt.throwOnFailure($result);
               localInt = 0;
               var2 = localInt;
               ((<undefinedtype>)$continuation).I$0 = localInt;
               ((<undefinedtype>)$continuation).label = 1;
               var10000 = fun2((Continuation)$continuation);
               if (var10000 == var6) {
                  return var6;
               }
               break;
            case 1:
               var2 = ((<undefinedtype>)$continuation).I$0;
               ResultKt.throwOnFailure($result);
               var10000 = $result;
               break;
            case 2:
               var2 = ((<undefinedtype>)$continuation).I$0;
               ResultKt.throwOnFailure($result);
               var10000 = $result;
               break label22;
            default:
               throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
         }

         var3 = var10000;
         localInt = var2 + ((Number)var3).intValue();
         var2 = localInt;
         ((<undefinedtype>)$continuation).I$0 = localInt;
         ((<undefinedtype>)$continuation).label = 2;
         var10000 = fun3((Continuation)$continuation);
         if (var10000 == var6) {
            return var6;
         }
      }

      var3 = var10000;
      localInt = var2 + ((Number)var3).intValue();
      return Boxing.boxInt(localInt);
   }
```

`fun1()` 编译之后多了一个 `Continuation` 类型的参数 `var0`，回头看下 `main` 方法，`main()` 调用 `fun1()` 的时候，把自身传了进去。`fun1()` 方法有点长，先看看 `label27` 这个代码块：

```kotlin
Object $continuation;
label27: {
   if (var0 instanceof <undefinedtype>) {
      $continuation = (<undefinedtype>)var0;
      if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
         ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
         break label27;
      }
   }
   $continuation = new ContinuationImpl(var0) {
      int I$0;
      // $FF: synthetic field
      Object result;
      int label;
      @Nullable
      public final Object invokeSuspend(@NotNull Object $result) {
         this.result = $result;
         this.label |= Integer.MIN_VALUE;
         return TestKt.fun1((Continuation)this);
      }
   };
}
```

`<undefinedtype>` 其实就是下面那个 `ContinationImpl` 的匿名子类，因此 `instanceof` 表达式肯定是为假，因为 `var0` 是 `SuspendLambda` 对象。因此 `if` 块会被跳过，直接执行下面的赋值语句：创建一个 `ContinuationImpl` 对象并赋给了 `$continuation`。该对象接收 `var0`  作为其构造函数的参数，它同样实现了 `invokeSuspend()` 方法，在 `invokeSuspend()` 方法里，又调用外层的 `fun1()`。好家伙，给绕晕了。

可以看出，首次执行 `fun1()` 的时候，`fun1()` 的参数 `var0` 是上游方法传来的 `Continuation` 对象（后面称其为 `SuspendLambda`），`$continuation `会被赋值为一个`ContinationImpl` 对象（后面称其为 `Continuation1`），该对象持有 `SuspendLambda`。后续 `fun1()` 被调用时，参数 `var0` 则是第一次执行时创建的 `Continuation1`，由于在调用前执行了 `this.label |= Integer.MIN_VALUE`  因此两层 `if` 判断都为真，`break label27` 会执行，从而直接跳出了 `label27` 块。

现在来看 `fun1()` 剩下部分的逻辑。毕竟是反编译，结果并不是那么容易读懂，于是将其改写成如下等价代码：

```kotlin
Object var10000;
int localInt;
int var2;
Object var3;

Object $result = ((<undefinedtype>)$continuation).result;
Object var6 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
switch (((<undefinedtype>)$continuation).label) {
    case 0:
        /* 对应 localInt = 0; fun2Result = fun2() */
    	ResultKt.throwOnFailure($result);
    	localInt = 0;
    	var2 = localInt;
    	((<undefinedtype>)$continuation).I$0 = localInt;
    	((<undefinedtype>)$continuation).label = 1;
    	var10000 = fun2((Continuation)$continuation);
    	if (var10000 == var6) {
        	return var6;
    	}
        // 注意这里没有 break;
    case 1:
        /* 对应 localInt += fun2Result; fun3Result = fun3() */
    	var2 = ((<undefinedtype>)$continuation).I$0;
    	ResultKt.throwOnFailure($result);
    	var10000 = $result;
    	var3 = var10000;
    	localInt = var2 + ((Number)var3).intValue();
    	var2 = localInt;
    	((<undefinedtype>)$continuation).I$0 = localInt;
    	((<undefinedtype>)$continuation).label = 2;
    	var10000 = fun3((Continuation)$continuation);
    	if (var10000 == var6) {
        	return var6;
    	}
		// 这里也没有 break;
    case 2:
		/* 对应 localInt += fun3Result; return localInt */
    	var2 = ((<undefinedtype>)$continuation).I$0;
    	ResultKt.throwOnFailure($result);
    	var10000 = $result;
    	var3 = var10000;
    	localInt = var2 + ((Number)var3).intValue();
    	return Boxing.boxInt(localInt);
    default:
    	throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
}
```

又看到老朋友 `switch case` 了，没错，`fun1()` 也是个状态机。编译后的 `fun1()` 将逻辑分成了三块，很显然，是因为两个 `fun2()`  `fun3()` 这两个 `suspend` 函数导致的。`IntrinsicsKt.getCOROUTINE_SUSPENDED()` 这行代码特别值得关注，此方法返回的是名为 `COROUTINE_SUSPENDED` 的单例对象。在前两个 `case` 块中，分别将 `fun2()` 和 `fun3()` 的返回值和它进行了比较，如果相等，则将这个值返回，否则就继续执行下一个 `case` 块。接下来看下 `fun2()` 和 `fun3()`。

## fun2 & fun3

```kotlin
@Nullable
public static final Object fun2(@NotNull Continuation $completion) {
   return Boxing.boxInt(1);
}
@Nullable
public static final Object fun3(@NotNull Continuation var0) {
   Object $continuation;
   label20: {
      if (var0 instanceof <undefinedtype>) {
         $continuation = (<undefinedtype>)var0;
         if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
            ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
            break label20;
         }
      }
      $continuation = new ContinuationImpl(var0) {
         // $FF: synthetic field
         Object result;
         int label;
         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            this.result = $result;
            this.label |= Integer.MIN_VALUE;
            return TestKt.fun3((Continuation)this);
         }
      };
   }
   Object $result = ((<undefinedtype>)$continuation).result;
   Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
   switch (((<undefinedtype>)$continuation).label) {
      case 0:
         ResultKt.throwOnFailure($result);
         ((<undefinedtype>)$continuation).label = 1;
         if (DelayKt.delay(1000L, (Continuation)$continuation) == var3) {
            return var3;
         }
         break;
      case 1:
         ResultKt.throwOnFailure($result);
         break;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
   }
   return Boxing.boxInt(1);
}
```

`fun2()` 就普普通通的一个函数，返回的是 `Boxing.boxInt(1)`，和 `COROUTINE_SUSPENDED` 不相等，因此 `fun1()` 执行完 `case 0` 块后会继续执行 `case 1` 块。

再看看 `fun3()`，这家伙就是` fun1()` 的翻版，它逻辑就没必要赘述了，重点看它 `case 0` 块中的这几行代码：

```kotlin
if (DelayKt.delay(1000L, (Continuation)$continuation) == var3) {
     return var3;
}
```

`DelayKt.delay()` 有两个参数，第一个参数不用说了，第二个参数前面已经分析过了，是 `fun3()` 内创建的 `ContinuationImpl` 对象（后面称其为 `Continuation3`）。继续追踪下去会发现，`DelayKt.delay()` 会将一个延时任务插入到事件循环中，1000ms 延时之后，`Continuation3` 的 `resumeWith()` 方法会被调用。

`resumeWith()` 是 `Continiuation` 接口的唯一方法，该方法在 `BaseContinuationImpl` 中有个 final 实现：

```kotlin
internal abstract class BaseContinuationImpl(
    // This is `public val` so that it is private on JVM and cannot be modified by untrusted code, yet
    // it has a public getter (since even untrusted code is allowed to inspect its call stack).
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    // This implementation is final. This fact is used to unroll resumeWith recursion.
    public final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
            // can precisely track what part of suspended callstack was already resumed
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }

    protected abstract fun invokeSuspend(result: Result<Any?>): Any?

    ......
}
```

`BaseContinuationImpl` 有一个 `Continuation` 类型的字段 `completion`，并在构造方法中初始化，`fun1()` 和 `fun3()` 创建 `ContinuationImpl` 时传入的 `var0` 就是赋给了这个 `completion`。`BaseContinuationImpl` 的方法体是一个 `while` 循环。其主要逻辑如下：

1. 调用 `invokeSuspend()`，判断返回结果，如果是 `COROUTINE_SUSPENDED`，就直接返回。否则无论成功还是失败，都会将结果封装在 `Result` 中给 `outcome`。
2. 接下来判断 `completion` 是不是  `BaseContinuationImpl` 类型，是的话就将 `current` 的值赋为 `completion`，也就是上游的 `Continuation`，将 `param` 赋为 `outcome`。这是什么意思呢？注释里其实已经解释了：用循环来展开递归。其实就是将尾递归转化成了循环，这应该是基于性能方面的考量。

    （Android 里面 `View` 的某些方法也有类似的骚操作，但后面好像又改成了递归，我觉得是因为循环可读性差不好维护，而且还有点违反面向对象的设计，除非真的对性能有很大的影响否则没必要）
3. `else` 块中是循环的出口，如注释所说，这时候已经到达了顶层，没有上游 `Continuation` 了。

为容易理解，可以把循环还原成递归：下游的 `Continuation` 的 `invokeSuspend()` 获取到结果后，调用上游 `Continuation` ，即 `completion` 的 `resumeWith` 方法，直到最顶层的 `Continuaion`。

重新梳理一下协程执行的整个过程：

1. 从 `SuspendLambda`（block）开始，因为 `label` 为 0，执行 `case 0` 代码块：将 `label` 置为 1 后，调用函数 `fun1()` ，并将自身传给了 `fun1()`。
2. `fun1()` 中构造了 `Continuation1`，并将 `SuspendLambda` 作为它的 `completion`。读取 `Continuation1` 的 `label` 字段，因为 `label == 0`，因此执行 `case 0` 代码块：
    * 初始化 `localInt`；
    * 将 `localInt` 保存到 `Continuation1` 的 `I$0` 字段中；
    * 将 `Continuation1` 的 `label` 置为 1；
    * 调用函数 `fun2()`。
  
3. `fun2()` 直接返回 `Boxing.boxInt(1)` 给 `fun1()`，`fun1()` 将这个值保存在，这个值和 `COROUTINE_SUSPENDED` 不相等，因此会继续执行 `fun1()` 的 `case 1` 块： 
    * 用 `Continuation1` 的 `I$0` 字段恢复 `localInt` 的值；
    * `localInt += Boxing.boxInt(1)`；
    * 将 `localInt` 保存到 `Continuation1` 的 `I$0` 字段中；
    * 将 `Continuation1` 的 `label` 置为 2；
    * 调用函数 `fun3()`。
  
4. `fun3()` 中构造了 `Continuation3`，并将 `Continuation1` 作为它的 `completion`。接下来读取 `Continuation3` 的 `label` 字段，因为 `label  == 0`，因此执行 `case 0` 块：
    * 将 `Continuation3` 的 `label` 置为 1 ；
    * 调用 `delay()` 方法。
  
5. `delay()` 向事件循环中插入一个延时任务，并立即返回 `COROUTINE_SUSPENDED` 给 `fun3()`，`fun3()` 将这个值返回给 ` fun1()`，`fun1()` 继续将这个值返回给 `SuspendLambda`，此时 `SuspendLambda` 的 `case 0` 块执行完毕。
6. 延时任务到期后，会调用 `Continuation3` 的 `resumeWith()` 方法，`fun3` 再次被调用并返回 `Boxing.boxInt(1)`。因为这个值不等于 `COROUTINE_SUSPENDED`，因此 `Continuation3` 会拿着这个值去调用其 `completion` 也就是  `Continuation1` 的 `resumeWith()` 方法。
7. `Continuation1` 的 `resumeWith()` 调用自身 `invokeSuspend()` 方法，`invokeSuspend()` 将值保存在 `result` 字段中之后，将 `Continuation1` 自身作为参数再次调用 `fun1()`。
8. `fun1()` 再次执行，从 `Continuation1` 读取 `label` 值，此时 `label` 为 2，执行 `case 2` 块：
    * 用 `Continuation1` 的 `I$0` 字段恢复 `localInt` 的值；
    * 读取 `Continuation1` 的 `result` 字段获取 `fun3()` 的返回结果 `Boxing.boxInt(1)`；
    * `localInt += Boxing.boxInt(1)`；
    * 将返回 `localInt` 返回给 `invokeSuspend()`。

9. `invokeSuspend()` 将结果返回给 `resumeWith()`，此结果不为 `COROUTINE_SUSPENDED`，因此执行 `completion` 也就是 `SuspendLambda` 的 `resumeWith()`，并将结果传给它。
10. `SuspendLambda` 的 `resumeWith()` 方法调用自身的 `invokeSuspend()` 方法，此时 `label` 为 1，执行 `case 1` 块：将结果打印出来。协程结束。

可以得出以下几个基本事实：

* 每一个 `suspend` 方法都和一个 `Continuation` 对象关联着；（`fun2()` 这种并没有真正 `suspend` 的方法除外）
* 当一个方法返回 `COROUTINE_SUSPENDED` 时，其实就是就是告诉调用者自己将会挂起（暂停），这个返回值会导致整个调用链结束，调用链上的所有方法也都被挂起；
* 下游方法恢复时，会通过调用上游方法的关联的 `Continuation` 对象的 `resumeWith()` 方法，触发上游方法的恢复。

最后画了一张图帮助理解：

![Kotlin 协程](/img/in-post/post_kotlin_coroutine_state_machine/kotlin_coroutine.svg)

# 结语

 Kotlin 协程中的所谓状态机，其实就是 Kotlin 为 `suspend` 方法生成的 `Continuation` 对象，严格来说是 `Continuation` 对象和方法共同构成了状态机：方法执行状态机的状态转移逻辑，`Continuation` 负责存储状态，方法如何执行由 `Continuation` 中的状态决定。

`Contiuation` 其实在无栈协程中充当了栈帧的作用：

* 保存了局部变量，比如 `Continuation` 中的 `I$0` 字段；
* 保存了方法中断后的返回地址，比如 `label`；
* 通过 `completion` 字段引用上游方法的 `Continuation`，构成了 `Continuation` 链，也就是 `suspend` 方法专属的 ”调用栈“。

‍
