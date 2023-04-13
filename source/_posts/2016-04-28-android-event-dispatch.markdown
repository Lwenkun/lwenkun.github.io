---
layout:     post
title:      "安卓事件传递机制的理解"
subtitle:   "安卓事件传递机制"
catalog:  true
date:       2016-04-28
author:     李文坤
header-img: "img/post-bg-android-event-dispatch.png" 
tags:
    - Android
    - View
---

# 安卓事件传递机制的理解 #

## 事件产生的源头以及预处理 ##

当我们的手指点击（滑动）屏幕时，我们的手指就可能触发了一系列的事件，这些事件队列的排列顺序可以这样表示：{ `ACTION_DOWN`, `ACTION_MOVE`, `ACTION_MOVE`, `ACTION_MOVE`, ... , `ACTION_UP` }，这样表示最开始我们触发的事件是`ACTION_DOWN`，然后是一至多个`ACTION_MOVE`事件（其实点击时也会触发`ACTION_MOVE`事件，因为屏幕很灵敏），当手指离开屏幕的瞬间，触发了一个 `ACTION_UP` 事件。

事件既然产生了，那怎么传递呢？

<!-- more -->

先别急，事件产生了，但还没获取呢。首先，消息获取模块会将事件通过pipe管道传递到客户端，然后 `InputQueue` 中的 `next()` 函数内部调用 `nativePollOnce()` 函数，该函数仅仅是一个 `DISPATCH_POINTER` 的异步消息，消息处理函数是 `deliverPointerEvent()` 函数。执行完该函数后，调用 `finishInputEvent()` 向消息获取模块发送一个回执，以使其进行下一个消息派送，真正完成绘制的代码是 native C++ 写的。

在 `deliverPointerEvent()` 中，会进行一些转换，包括物理像素到逻辑像素的转换和屏幕坐标到视图坐标的转换，这都是为消息的派发做好准备。

## 事件的传递 ##

执行了转换后，我们的事件即event便开始了漫长的旅途。

如果用一句话来概括event的漫长路途的话便是：

**`deliverPointerEvent()` 通过执行 `mView.dispatchTouchEvent()` 将消息派发给根视图（这个根视图具体是谁我们等下在后面会提到），之后mView便会将事件派发到整个 `View` 树。**

太抽象笼统了是吧？那我们便来细细分析，你可以泡上一杯茶或咖啡，跟随 event 看看他在漫长旅途中发生的了那些有趣的事。

### 事件在进入View树之前的传递 ###

前面说了 event 的第一站便是 `mView.dispatchTouchEvent()`，该函数是在`ViewRoot`（注意 `ViewRoot` 不是 `View` 类，而是继承 `Handle` 类，用于对整个视图数的控制）中调用的，`mView` 有两种类型，对于应用窗口来说，`mView` 是一个 `PhoneWindow` 的 `DecorView` 类型，对于非应用窗口而言，`mView` 是一般的 `ViewGroup` 类型。

在 `DecorView` 中，会判断是否存在 `Callback` 对象，这个 `Callback` 对象是谁呢？哈哈，竟然是 `Activity`，其实不奇怪，我们看源码就可以知道 `Activity` 实现了 `Window.Callback` 这个回调接口。如果不存在 `Callback` 对象的话，那么就直接调用 `DecorView` 父类 `ViewGroup` 中的 `dispatchTouchEvent()` 方法。

我们首先看如果存在 `Callback` 对象即 `Activity` 时，event是在 `Activity` 中经历了什么。

在 `Activity` 中，event 被传入了它的 `dispatchTouchEvent()` 方法：
如果 event 的消息类型是 `ACTION_DOWN`，那么就调用 `onUserInteraction()`，这个方法是个空方法，什么也不干，只是让应用程序一个处理消息的机会，应用如果想在消息传递的最初始阶段想做些什么的话，就可以在这个函数中实现；
然后这个方法中又会调用 `Activity` 关联的 `Window` 类对象的 `superDispatchTouchEvent()` 方法；
如果 `Window` 类没有消耗该消息，那么 `Activity` 就会调用自己的 `onTouchEvent()` 方法，该方法也默认什么都不做，留给应用程序实现。

`Window` 类中在event身上又发生了什么呢，我们往下看：

`Window` 类的 `superDispatchEvent()` 方法中，会调用 `mDecor` 的 `superDispatchTouchEvent()` 方法，这个方法又会调用 `super.dispatchTouchEvent()`，即调用自己父类的 `dispatchTouchEvent()` 方法。看到这里我们猛然回头发现这是到了 `DecorView` 中如果不存在 `Callback` 对象时要走的另一条支路。

为什么要这样做呢？很明显，这是让 event 在视图树中旅行之前给 `Activity` 一个处理它的机会。很明显，这一阶段虽然 event 已经进入 `View` 树的根视图中了，但是主要还是在 `Activity` 中被处理，所以还是把这阶段归结为在 `Activity` 中的传递

之后开始才是 event 真正在视图树中的旅途了，它进入了视图树这个庞大的王国之中，各种有趣的事即将发生。

### 事件在View树中的传递 ###
 
我们知道 `DecorView` 是一个 `ViewGroup`，所以我们从 `ViewGroup` 分析。

event 进入 `ViewGroup` 后，`ViewGroup` 便通过 `dispatchTouchEvent()` 方法将它派送，它先会被派送到 `onInterceptTouchEvent()` 中（决定是否将其拦截），这个方法的意思是拦截，在 `ViewGroup` 中，它的默认实现是：

```java
public void onInterceptTouchEvent(MotionEvent event) {
   retrun false;
}
```

这是不拦截意思。

关于这个方法官方文档有段很长的话（本来想翻译的，但是想想怕翻译得不好坏了原本的意思），大家还是自己翻译看看

```java
/** 
 * Implement this method to intercept all touch screen motion events.  This 
 * allows you to watch events as they are dispatched to your children, and 
 * take ownership of the current gesture at any point. 
 * 
 * <p>Using this function takes some care, as it has a fairly complicated 
 * interaction with {@link View#onTouchEvent(MotionEvent) 
 * View.onTouchEvent(MotionEvent)}, and using it requires implementing 
 * that method as well as this one in the correct way.  Events will be 
 * received in the following order: 
 * 
 * <ol> 
 * <li> You will receive the down event here. 
 * <li> The down event will be handled either by a child of this view 
 * group, or given to your own onTouchEvent() method to handle; this means 
 * you should implement onTouchEvent() to return true, so you will 
 * continue to see the rest of the gesture (instead of looking for 
 * a parent view to handle it).  Also, by returning true from 
 * onTouchEvent(), you will not receive any following 
 * events in onInterceptTouchEvent() and all touch processing must 
 * happen in onTouchEvent() like normal. 
 * <li> For as long as you return false from this function, each following 
 * event (up to and including the final up) will be delivered first here 
 * and then to the target's onTouchEvent(). 
 * <li> If you return true from here, you will not receive any 
 * following events: the target view will receive the same event but 
 * with the action {@link MotionEvent#ACTION_CANCEL}, and all further 
 * events will be delivered to your onTouchEvent() method and no longer 
 * appear here. 
 * </ol> 
 * 
 * @param ev The motion event being dispatched down the hierarchy. 
 * @return Return true to steal motion events from the children and have 
 * them dispatched to this ViewGroup through onTouchEvent(). 
 * The current target will receive an ACTION_CANCEL event, and no further 
 * messages will be delivered here. 
 */  
```

而 `ViewGroup` 中的 `dispatchTouchEvent()` 的源码是这样的

```java
public boolean dispatchTouchEvent(MotionEvent ev) {  
    final int action = ev.getAction();  
    final float xf = ev.getX();  
    final float yf = ev.getY();  
    final float scrolledXFloat = xf + mScrollX;  
    final float scrolledYFloat = yf + mScrollY;  
    final Rect frame = mTempRect;  
    boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;  
    if (action == MotionEvent.ACTION_DOWN) {  
        if (mMotionTarget != null) {  
            mMotionTarget = null;  
        }  
        if (disallowIntercept || !onInterceptTouchEvent(ev)) {  
            ev.setAction(MotionEvent.ACTION_DOWN);  
            final int scrolledXInt = (int) scrolledXFloat;  
            final int scrolledYInt = (int) scrolledYFloat;  
            final View[] children = mChildren;  
            final int count = mChildrenCount;  
            for (int i = count - 1; i >= 0; i--) {  
                final View child = children[i];  
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE  
                        || child.getAnimation() != null) {  
                    child.getHitRect(frame);  
                    if (frame.contains(scrolledXInt, scrolledYInt)) {  
                        final float xc = scrolledXFloat - child.mLeft;  
                        final float yc = scrolledYFloat - child.mTop;  
                        ev.setLocation(xc, yc);  
                        child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
                        if (child.dispatchTouchEvent(ev))  {  
                            mMotionTarget = child;  
                            return true;  
                        }  
                    }  
                }  
            }  
        }  
    }  
    boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||  
            (action == MotionEvent.ACTION_CANCEL);  
    if (isUpOrCancel) {  
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;  
    }  
    final View target = mMotionTarget;  
    if (target == null) {  
        ev.setLocation(xf, yf);  
        if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
            ev.setAction(MotionEvent.ACTION_CANCEL);  
            mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        }  
        return super.dispatchTouchEvent(ev);  
    }  
    if (!disallowIntercept && onInterceptTouchEvent(ev)) {  
        final float xc = scrolledXFloat - (float) target.mLeft;  
        final float yc = scrolledYFloat - (float) target.mTop;  
        mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        ev.setAction(MotionEvent.ACTION_CANCEL);  
        ev.setLocation(xc, yc);  
        if (!target.dispatchTouchEvent(ev)) {  
        }  
        mMotionTarget = null;  
        return true;  
    }  
    if (isUpOrCancel) {  
        mMotionTarget = null;  
    }  
    final float xc = scrolledXFloat - (float) target.mLeft;  
    final float yc = scrolledYFloat - (float) target.mTop;  
    ev.setLocation(xc, yc);  
    if ((target.mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
        ev.setAction(MotionEvent.ACTION_CANCEL);  
        target.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        mMotionTarget = null;  
    }  
    return target.dispatchTouchEvent(ev);  
}  
```

结合官方文档和源码我解释一下：

**1.** 如果 `disallowIntercept` 为 `false` （默认是 `false`，子视图通过 `requestDisallowInterceptTouchEvent(true)` 可不允许父视图拦截消息），并且 `onInterceptTouchEvent()` 返回 `true`，那么 `ViewGroup` 就会拦截事件，事件不会给子视图处理，而是交由自己父类的 `dispatchTouchEvent()` 来处理，即 `View` 的 `diapatchTouchEvent()` 来处理，这也就相当与ViewGroup将事件交给了自己来处理。`View` 的 `dispatchTouchEvent()` 长什么样呢？我们看看源码：

```java
public boolean dispatchTouchEvent(MotionEvent event) {  
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
            mOnTouchListener.onTouch(this, event)) {  
        return true;  
    }  
    return onTouchEvent(event);  
}
```

安卓5.0之后该方法的实现略带复杂，但是基本思想还是一样的，我们不妨就拿这个简单的来分析。在这个方法中会判断这个 `View` 的 `OnTouchListener` 回调接口是否存在，`View` 是不是 `ENABLE` 状态，是的话就会继续执行到第三个条件，第三个条件其实执行了 `OnTouchListener` 的 `onTouch()` 方法，这个方法是我们调用 `view.setOnTouchListener()` 时实现的。如果这个方法返回了 true， 那么 `dispatchEvent()` 方法也返回 `true`，`dispatchEvent()` 方法执行完毕；如果回调接口不存在，或者这个 `View` 是 `DISABLE` 状态，或者 `onTouch()` 返回 `false`（即 `onTouch()` 未消耗该 event）时， 便会执行 `onTouchEvent()` 方法。如果 `onTouchEvent()` 返回 `false` （即这个方法也未消耗 event)时，`dispatchTouchEvent()` 返回 `false`，从而表明 `View` 未消耗这个事件；反之 `onTouchEvent()` 返回 `true` 即消耗了这个事件时，`dispatchTouchEvent()` 也返回 `true`，表明 `View` 消耗了该事件。

从上面的分析，我们可以看出，一个 event 在 `View` 中，首先会被 `dispatchTouchEvent()` 派发给 `onTouch()` 处理（前提是注册了 `onTouch()` 事件），如果 `onTouch()` 返回 `true`（消耗了该 event），event 在 `View` 中的旅程就结束了，`dispatchTouchEvent()`返回 `true`；如果没消耗就会继续派发给  `onTouchEvent()` 来处理，`dispatchTouchEvent()` 会将 `onTouchEvent()` 的返回值作为自己的返回值。

接下来我们自然想知道，event 在 `onTouchEvent()` 方法中会发生什么呢，我们看看源码：

```java
public boolean onTouchEvent(MotionEvent event) {  
    final int viewFlags = mViewFlags;  
    if ((viewFlags & ENABLED_MASK) == DISABLED) {  
        // A disabled view that is clickable still consumes the touch  
        // events, it just doesn't respond to them.  
        return (((viewFlags & CLICKABLE) == CLICKABLE ||  
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));  
    }  
    if (mTouchDelegate != null) {  
        if (mTouchDelegate.onTouchEvent(event)) {  
            return true;  
        }  
    }  
    if (((viewFlags & CLICKABLE) == CLICKABLE ||  
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {  
        switch (event.getAction()) {  
            case MotionEvent.ACTION_UP:  
                boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;  
                if ((mPrivateFlags & PRESSED) != 0 || prepressed) {  
                    // take focus if we don't have it already and we should in  
                    // touch mode.  
                    boolean focusTaken = false;  
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {  
                        focusTaken = requestFocus();  
                    }  
                    if (!mHasPerformedLongPress) {  
                        // This is a tap, so remove the longpress check  
                        removeLongPressCallback();  
                        // Only perform take click actions if we were in the pressed state  
                        if (!focusTaken) {  
                            // Use a Runnable and post this rather than calling  
                            // performClick directly. This lets other visual  
                            // state of the view update before click actions start.  
                            if (mPerformClick == null) {  
                                mPerformClick = new PerformClick();  
                            }  
                            if (!post(mPerformClick)) {  
                                performClick();  
                            }  
                        }  
                    }  
                    if (mUnsetPressedState == null) {  
                        mUnsetPressedState = new UnsetPressedState();  
                    }  
                    if (prepressed) {  
                        mPrivateFlags |= PRESSED;  
                        refreshDrawableState();  
                        postDelayed(mUnsetPressedState,  
                                ViewConfiguration.getPressedStateDuration());  
                    } else if (!post(mUnsetPressedState)) {  
                        // If the post failed, unpress right now  
                        mUnsetPressedState.run();  
                    }  
                    removeTapCallback();  
                }  
                break;  
            case MotionEvent.ACTION_DOWN:  
                if (mPendingCheckForTap == null) {  
                    mPendingCheckForTap = new CheckForTap();  
                }  
                mPrivateFlags |= PREPRESSED;  
                mHasPerformedLongPress = false;  
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());  
                break;  
            case MotionEvent.ACTION_CANCEL:  
                mPrivateFlags &= ~PRESSED;  
                refreshDrawableState();  
                removeTapCallback();  
                break;  
            case MotionEvent.ACTION_MOVE:  
                final int x = (int) event.getX();  
                final int y = (int) event.getY();  
                // Be lenient about moving outside of buttons  
                int slop = mTouchSlop;  
                if ((x < 0 - slop) || (x >= getWidth() + slop) ||  
                        (y < 0 - slop) || (y >= getHeight() + slop)) {  
                    // Outside button  
                    removeTapCallback();  
                    if ((mPrivateFlags & PRESSED) != 0) {  
                        // Remove any future long press/tap checks  
                        removeLongPressCallback();  
                        // Need to switch from pressed to not pressed  
                        mPrivateFlags &= ~PRESSED;  
                        refreshDrawableState();  
                    }  
                }  
                break;  
        }  
        return true;  
    }  
    return false;  
}  
```

在 `onTouchEvent()` 方法中我们看到，如果这个 `View` 是可点击的，那么对于 `ACTOIN_UP` 事件，我们就发送一个异步消息来处理点击事件，如果异步消息没有发送成功，那么就会立即执行点击事件，在源码中也就是 `performClick()` 方法，`performClick()` 方法是怎样的呢，如果你看了源码就知道则个方法里执行的是我们 `View` 注册点击事件，即 `OnClickListener的onClick()` 方法。当然 `ACTOIN_UP` 有还有可能会触发长按事件 `onLongClick()`，这里就不详细介绍了。

**2.** 如果 `disallowIntercept` 为 `false` 并且 `ViewGroup` 没有拦截 event（即返回 `false` ），或者 `disallowIntercept` 为 `true`，那么事件就会传给 `ViewGroup` 中被点击的那个子视图（这里包括前面所讲的子/父视图都包括 `View` 和 `ViewGroup`），这样就回到了事件在 `View` 或者 `ViewGroup` 的传递过程了，我们就可以按照前面的分析用递归思想理解后续的过程。

需要注意的是，在 `ViewGroup` 的 `dispatchTouchEvent()` 方法中，我们能够知道，如果当事件  `ACTION_DOWN` 没有找到目标子视图（可能原因是没有点击到任何子视图或者虽然有子视图被点击但是该子视图没有消耗该事件，即子视图的 `dispatchTouchEvent()` 方法返回 `false`）时，`ViewGroup` 会将事件交给自己处理，并且之后的 `ACTION_MOVE` 和 `ACTION_UP` 事件都不会交给任何子视图处理，也是全交给自己处理，也即前面讲的交给 `super.dispatchTouchEvent()` 来处理。

## 说明 ##

**1. 一个视图是否消耗该事件，表现在 `dispatchTouchEvent()` 的返回值上，如返回 `true` 表示消耗 `false` 表示未消耗；而一个消息处理方法（指的是 `ouTouch()` 和 `onEventTouch()`，不包括  `onInterceptTouchEvent()`)是否消耗该事件表现在该方法的返回值上，例如 `onTouch()` 方法返回 `true` 表示消耗 `false` 表示未消耗。**

**2. 如果某个视图的某个消息处理方法消耗该事件，会使得该视图的 `dispatchTouchEvent()` 方法返回 `true`，即该视图消耗了该事件；如果某个视图的所有消息处理方法都返回 `false`，那么这个视图的 `dispatchTouchEvent()` 方法返回 `false`，即该视图没有消耗该事件。**

到了这里安卓事件的传递就应该结束了，但我在这之中省略了对 `ViewGroup` 是如何找到被点击的子视图的的分析，这个我会在之后分享。以上便是我对安卓 `View` 的事件传递机制的理解，有什么疏漏或者错误的地方欢迎大家指出。

参考文章：<br/>
[http://blog.csdn.net/guolin_blog/article/details/9097463](http://blog.csdn.net/guolin_blog/article/details/9097463 "郭霖的博客")、[http://blog.csdn.net/guolin_blog/article/details/9153761](http://blog.csdn.net/guolin_blog/article/details/9153761 "郭霖的博客")

参考书籍：《Android 内核剖析》柯元旦 著














