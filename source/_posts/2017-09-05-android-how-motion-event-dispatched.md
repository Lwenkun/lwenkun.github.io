---
layout:     post
title:      "Android 事件分发规律总结"
subtitle:   "Android 事件分发规律总结"
date:       2017-09-05
catalog:  true
author:     李文坤
header-img: "img/post-bg-android-hot-fix.jpg"
tags:
    - Android
    - View
    - 事件分发
---

# Android 事件分发规律总结

### 定义
事件序列：手指接触屏幕开始到离开屏幕为止产生的事件为一次事件序列。

## 规律一
从父视图的角度来看，无论他的子视图是 View 还是 ViewGroup，对父视图来说都是透明的。它只知道如果当 ACTION\_DOWN 事件传给子视图后子视图的 dispatchTouchEvent () 返回 true，说明子视图想要处理这个事件，那么以后的事件就都交给它，不管以后子视图的 dispatchTouchEvent() 返回的是 true 还是 false; 如果 ACTION\_DOWN 事件传给子视图后子视图返回的是 false，那么以后的事件再也不会传给子视图（子视图没有处理 ACTION\_DOWN 就视作它不想处理该事件序列）。如果一个子视图的 dispatchTouchEvent() 在处理 ACTION\_DOWN 时返回 true ，只要父视图没有拦截事件, 那么该事件序列中的后续事件都会传入该视图中，也就是传入该视图的 dispatchTouchEvent() 方法中，就算手指的触摸区域已经超出了该视图的范围。并且对于该事件序列的之后所有事件，即使该视图的 dispatchTouchEvent() 返回 false 事件也同样会继续传入该视图，唯一的影响是会将该事件原路返回，最终落到 activity.onTouchEvent() 中。因此建议如果消耗了该事件，除非有特殊情况需要处理，最好不要返回 false，否则上级视图以为下面的视图没有处理事件从而自己处理。
<!-- more -->
## 规律二
父视图有两种角色，一种是 ViewGroup , 另一种是 View。当父视图的角色是 ViewGroup 时，父视图的主要任务就是分派事件给子视图；当父视图的角色为 View 时，它和普通的 View 一样处理事件。
父视图在两种情况下会将事件拿给自己处理：
1）如果没有子视图能够处理事件，那么只能由父视图自己处理，此时会把事件传给 super.dispatchTouchEvent()，也就是父类方法来处理（注意是父类，不是父视图）。
2）假如本来整个事件序列都是要传给子视图的，但是中途父视图想要拦截的话，从父视图决定拦截开始之后的所以事件都不会再交给子视图处理了，但是决定拦截时的那一次的事件会还是会交给子视图的，只不过是事件的类型是 ACTION\_CANCEL，这样做的原因我认为可能是通知一下子视图父视图拦截了你的事件，免得子视图事件被拦截了还蒙在鼓里（其实就是让开发者可以对这种情况加以处理）。从开始拦截的那一次开始，之后的所有事件（不包括拦截的那一次）都不会传给子视图了，都传入 super.dispatchOnToucnEvent() 中，即交给父视图给自己处理。

## 规律三
如果事件是交给自己处理，不管是子视图还是父视图，只要事件交给了自己处理，那么事件的处理流程都是一样的（这个流程在 View 中定义），下面来分析是如它们何处理事件的：
如果视图将事件交给自己处理，那么事件首先都是从 dispatchTouchEvent() 传入的（父视图是 super.dispatchTouchEvent()），这个方法的逻辑是：如果该视图注册了OnTouchListener 监听器，并且监听器不为 null，那么就把事件交给监听器中的 onTouch() 方法。下一步就是看 onTouch() 的返回值了，如果返回值为 true，那么说明 onTouch() 消费了该事件，事件的传递到此为止， dispatchTouchEvent() 返回 true；如果返回值为 false，那么说明 onTouch() 方法并没有消耗该事件，那么进行下一步的事件分发。下一步事件会传给 onTouchEvent 方法，这个方法是最常见也最常用的方法，对于一个 自定义 View 来说，这个方法是根据事件来更改 View 的行为的主要场所。dispatchTouchEvent() 同样需要根据 onTouchEvent() 的返回值来决定下一步的事件转发流程。如果 onTouchEvent() 返回 false，那么 dispatchTouchEvent() 返回的就是 false，否则返回的就是 true。这个过程可以用如下代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent e) {
    return (onTouchListener != null) ? (onTouchListener.onTouch(e) ? true : onTouchEvent(ev)) : onTouchEvent(e);
}
```

之前已经说到，如果当父视图接收到 ACTION\_DOWN 事件时，如果没有子视图能够处理该事件，那么后续的事件就不会给该子视图了，并且该事件会交给父视图处理，即传入父视图的 super.dispatchTouchEvent() 方法中。子视图有没有处理该事件就是根据 dispatchTouchEvent() 的返回值来看的。返回 true 表示处理，false 表示未处理。如果子视图对于 ACTION\_DOWN 事件进行了处理（dispatchTouchEvent() 返回true），那么之后的事件都会给它处理，不管子视图之后 dispatchTouchEvent() 是否返回 true，也就是说，父视图会忽略子视图接下来的 dispatchTouchEvent() 的返回值，只要事件继续发生就都传给该子视图。但是，返回 false 会导致事件从该子视图返回给上级视图，上级视图也不会处理该事件，该事件会一直上传，直至传到 Activity 中的 onTouchEvent() 方法中。 

## 规律四
关于事件的拦截：
父视图可以在任何时候拦截本应该传递子视图的事件，拦截的依据就是 onInterceptTouchEvent() 的返回值，true 表示拦截，false 表示不拦截，父视图每次分发事件时会根据这个方法的返回值决定是否开始拦截：如果返回 false，即不拦截该事件，父视图下次分发事件时依然会调用这个方法来判断是否开始拦截事件；如果返回 true，说明该事件需要拦截，那么以后的事件，包括该事件都会直接交由父视图的 super.dispatchTouchEvent() 方法处理，而不会再传入子视图的 dispatchTouchEvent() 方法中 。但是开始拦截的那个事件还是会交给那个子视图，只是事件的类型变为了 ACTION\_CANCEL。

## 规律五
事件在系统中的传递流程：ViewRootImpl => DecorView(调用的方法是 dispatchTouchEvent()) => WindowCallback(即Activity) => PhoneWindow => DecorView(调用的方法是 super.dispatchTouchEvent)

## 规律六
子视图可以调用父视图的 requestDisallowInterceptTouchEvent() 方法来向父视图请求不拦截事件。但是这个方法只对除 ACTION\_DOWN 之外的其他事件有效，如果父视图选择拦截 ACTION\_DOWN，子视图即使在此之前调用此方法也是没有任何用的。并且需要注意，如果父视图在此方法调用之前就已经对事件进行了拦截，那么这个方法也同样无效。

## 规律七
父视图如何找到能够接受某个事件的子视图？

假设在父视图中的事件坐标是 (x, y)

1）首先要转换成相对子视图的坐标，转换的步骤如下：

首先要考虑父视图是否发生滚动，其次需要考虑子视图是否发生平移，如果用（x', y'）表示转换后的坐标，那么有：

x' = x + scrollX - child.left - child.translationX;<br>
y' = y + scrollY - child.top - child.translationY;

其实在源码中上述变换过程分为两步进行：

1. 第一步不考虑平移：x' = x + scrollX - child.left; 
2. 第二步通过 TransformationInfo（内含变换矩阵） 将平移考虑进去：x' = x + scrollX - child.left - child.translationX;

2）然后根据相对子视图的坐标判断该点是否落在子视图的范围内。判断的依据是：x' < width && y' < height && x' >= 0 && y' >= 0（width 和 height 是子视图的宽高）。