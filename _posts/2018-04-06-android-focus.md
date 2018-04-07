---
layout:     post
title:      "View 的焦点机制"
subtitle:  ""
date:       2018-04-06
author:     "lwenkun"
catalog:  true
header-img: "img/post-bg-android-how-apps-get-system-services.jpg"
tags:
    - Android
    - View
    - Focus
---

# View 的焦点机制

**约定：文章中的 View 有时是指狭义的 View.class，有时指的是 View.class 和 ViewGruop.class 的统称，具体含义根据上下文而定**。

## 和焦点相关的 xml 属性

在 xml 中，有两个比较重要的属性和焦点有关，它们是 focusable 和 focusableInTouchMode。前者决定这个 View 是否可获取焦点，如果它的值为 false，那么它就和焦点无缘了；后者决定这个 View 在触屏模式下是否可获取焦点，如果它的值为 false，那么即使 focusable 的值为 true，在触屏模式下它也无法获取焦点。比如 Button，如果我们通过外接键盘进行操作，我们会发现 Button 是可以获得焦点的，但是在触屏模式下，Button 是不可获取焦点的。所以我们可以知道 Button 的 focusable 属性默认为 true，而 focusableInTouchMode 属性为 false。

一个 View 能够获取焦点不代表它拥有焦点，因此，即使你在 xml 中指定这两个属性为 true，在窗口显示时这个 View 也不一定能立马获取焦点。因为窗口上还存在其他能够获取焦点的 View，例如 EditText，它的这两个属性的默认值都为 true。虽然一个界面上可以有多个 View 拥有获得焦点的能力，但是每个时刻只有一个 View 能够真正的获取焦点。

除了上面两个属性外，还有一些和焦点转移相关的属性，比如 nextFocusUp，nextFocusDown，nextFocusFoward 等。当我们通过外接键盘进行操作时，我们有时需要通过上下左右来控制焦点，nextFocusUp，nextFocusDown 等就是用来指明下一个焦点将转移到哪个视图上。当我们填写表单时，点击回车键可以将焦点转移到下一个 EditText 上，这时候我们就可以通过 nextFocusFoward 来指定这个 EditText。你会发现就算我们不指明这些属性系统都能够将焦点转移到合适的视图上，这是因为系统会通过计算帮你找到下一个能够获取焦点的最佳视图。关于系统的焦点转移算法可以参考 FocusFinder 这个类。


## 和焦点相关的方法

和焦点相关的方法主要有 hasFocus()，isFocused()，requestFocus()，unFocused()，clearFocus()，setFocusable() 和 setFocusableInTouchMode()。最后两个方法对应的就是 xml 中的那两个属性，我们主要讲下前面几个方法。

### hasFocus

首先是 hasFocus() 方法，它在 View 和 ViewGroup 中的定义分别如下：

```java
// View.java
public boolean hasFocus() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0;
}
// ViewGroup.java
public boolean hasFocus() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0 || mFocused != null;
}
```
PFLAG_FOCUSED 这个标志位用来标识这个 View 是否具有焦点，mFocused 是 ViewGroup 特有的变量，它用记录自身具有焦点或者后代具有焦点的子 View，如果不存在这样的子 View， mFocus 的值就为 null。通过对比可以发现，hasFocus() 在 View 中的含义是这个 View 是否具有焦点，在 ViewGroup 中的含义是该 ViewGroup 是否具有焦点或者它的后代是否有焦点。

### isFocused

isFocused() 只在 View 中有定义，其定义如下：

```java
public boolean isFocused() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0;
}
```
这个方法的和 hasFocus() 有点相似，不过它在 View 和 ViewGroup 中的含义都是一样的，表示这个 View 本身是否具有焦点。

### requestFocus

requestFocus() 在 View 中的定义如下：

```java
public final boolean requestFocus() {
    return requestFocus(View.FOCUS_DOWN);
}

public final boolean requestFocus(int direction) {
    return requestFocus(direction, null);
}

public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
    return requestFocusNoSearch(direction, previouslyFocusedRect);
}

private boolean requestFocusNoSearch(int direction, Rect previouslyFocusedRect) {
    // need to be focusable
    if ((mViewFlags & FOCUSABLE) != FOCUSABLE
            || (mViewFlags & VISIBILITY_MASK) != VISIBLE) {
        return false;
    }
    // need to be focusable in touch mode if in touch mode
    if (isInTouchMode() &&
        (FOCUSABLE_IN_TOUCH_MODE != (mViewFlags & FOCUSABLE_IN_TOUCH_MODE))) {
           return false;
    }
    // need to not have any parents blocking us
    if (hasAncestorThatBlocksDescendantFocus()) {
        return false;
    }
    handleFocusGainInternal(direction, previouslyFocusedRect);
    return true;
}
```


对于 View 来说，requestFocus() 的返回值的含义是这个 View 是否成功获取了焦点。从上面的代码可知，如果这个 View 不可获取焦点，或者这个 View 不可见，或者这个 View 在触屏模式下不可获取焦点，那么这个方法返回 false，也就是说这个 View 获取不了焦点；否则调用 handleFocusGainInternal() 方法之后返回 true。由此在可见 handleFocusGainInternal() 方法中，View 成功获取了焦点。

requestFocus() 方法在 ViewGroup 中的定义如下：

```java
public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
    if (DBG) {
        System.out.println(this + " ViewGroup.requestFocus direction="
                + direction);
    }
    int descendantFocusability = getDescendantFocusability();
    switch (descendantFocusability) {
        case FOCUS_BLOCK_DESCENDANTS:
            return super.requestFocus(direction, previouslyFocusedRect);
        case FOCUS_BEFORE_DESCENDANTS: {
            final boolean took = super.requestFocus(direction, previouslyFocusedRect);
            return took ? took : onRequestFocusInDescendants(direction, previouslyFocusedRect);
        }
        case FOCUS_AFTER_DESCENDANTS: {
            final boolean took = onRequestFocusInDescendants(direction, previouslyFocusedRect);
            return took ? took : super.requestFocus(direction, previouslyFocusedRect);
        }
        default:
            throw new IllegalStateException("descendant focusability must be "
                    + "one of FOCUS_BEFORE_DESCENDANTS, FOCUS_AFTER_DESCENDANTS, FOCUS_BLOCK_DESCENDANTS "
                    + "but is " + descendantFocusability);
    }
}
```
ViewGroup 覆盖了 View 的 requestFocus() 方法，它会根据 descendantFocusability 的值的不同进行不同的处理，descendantFocusability 有三个可能的值：第一个是 FOCUS_BLOCK_DESCENDANTS，它的意思是对焦点进行拦截，不给子 View 进行请求焦点的机会，而是自己请求；第二个是 FOCUS_BEFORE_DESCENDANTS，让自己先去请求焦点，如果没能成功获得焦点才让子 View 去请求焦点；第三个是 FOCUS_AFTER_DESCENDANTS，让子 View 先去请求焦点，如果子 View 没能成功请求到焦点才由自己请求。ViewGruop 的这个属性的默认值为 FOCUS_BEFORE_DESCENDANTS。

当 descendantFocusability 为 FOCUS_BLOCK_DESCENDANTS 时，通过调用父类 View 的 requestFocus() 方法执行为自己请求焦点的逻辑；当值为 FOCUS_BEFORE_DESCENDANTS 时，先为自己请求焦点，如果未能成功，那么再调用 onRequestFocusInDescendants() 方法，这个方法从名字就能知道是为子 View 请求焦点，或者说是把请求焦点的任务交给了后代。因此，对于 ViewGroup 来说，这个方法的返回值和 View 有点不同：如果返回 true，说明这个 ViewGroup 获取了焦点或者它的某个后代成功地获取了焦点；否则，这个 ViewGroup 未能成功获取焦点并且它的子 View 也未能成功获取焦点。

现在我们看下 onRequestFocusInDescendants() 的定义：

```java
protected boolean onRequestFocusInDescendants(int direction,
        Rect previouslyFocusedRect) {
    int index;
    int increment;
    int end;
    int count = mChildrenCount;
    if ((direction & FOCUS_FORWARD) != 0) {
        index = 0;
        increment = 1;
        end = count;
    } else {
        index = count - 1;
        increment = -1;
        end = -1;
    }
    final View[] children = mChildren;
    for (int i = index; i != end; i += increment) {
        View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
            if (child.requestFocus(direction, previouslyFocusedRect)) {
                return true;
            }
        }
    }
    return false;
}
```

这个方法会遍历所有的子 View，寻找第一个自身或者后代能够成功获取焦点的 View，如果找到了，停止后续的遍历并返回 true；如果不存在这样的 View，返回 false。

从上面的分析可知，requestFocus() 方法不保证一定能获取焦点，还要看该 View 或者 ViewGroup 是否具有获取焦点的能力。对于 View 来说，要想在触屏模式下拥有获取焦点的能力，首先要可见，其次要保证自己的 focusable 和 focusableInTouchMode 这两个属性的值为 true；而对于 ViewGroup 来说，这种能力除了上面说的这些点之外，还要保证在 FOCUS_AFTER_DESCENDANTS 模式下，后代没有成功获取焦点。

### unFocus

这个方法是内部方法，应用程序不能直接调用，它在 View 和 ViewGroup 中有不同的定义：

```java
// View.java
void unFocus(View focused) {
    if (DBG) {
        System.out.println(this + " unFocus()");
    }
    clearFocusInternal(focused, false, false);
}

void clearFocusInternal(View focused, boolean propagate, boolean refocus) {
    if ((mPrivateFlags & PFLAG_FOCUSED) != 0) {
        mPrivateFlags &= ~PFLAG_FOCUSED;
        if (propagate && mParent != null) {
            mParent.clearChildFocus(this);
        }
        onFocusChanged(false, 0, null);
        refreshDrawableState();
        if (propagate && (!refocus || !rootViewRequestFocus())) {
            notifyGlobalFocusCleared(this);
        }
    }
}

// ViewGroup.java
@Override
void unFocus(View focused) {
    if (DBG) {
        System.out.println(this + " unFocus()");
    }
    if (mFocused == null) {
        super.unFocus(focused);
    } else {
        mFocused.unFocus(focused);
        mFocused = null;
    }
}
```
对于 View 来说，这个方法仅仅是调用了 clearFocusInternal() 方法，最终达到的效果是清除了 PFLAG_FOCUSED 这个标志位，也就是清除了自身的焦点。而对于 ViewGroup 来说，如果自己的 mFocused  变量为 null 的话，就调用父类 View 的 unFoucs() 方法，执行取消自身焦点的逻辑；否则，调用 mFocused 指向的那个 View 的 unFocus() 方法，并且将 mFocused 置为 null。结合 mFocused 的含义来看，对于 ViewGroup 来说，这个方法达到的效果就是：如果自身有焦点（自身具有焦点的 ViewGroup 的 mFocused 值为 nul），那么将自身的焦点清除；否则通过递归调用将焦点路径（关于焦点路径在后面会有介绍）上从该节点开始的所有后续中间节点的 mFocused 置为 null，并将焦点路径的终止节点（即具有焦点的那个 View）的焦点清除。

### clearFocus

这个方法只在 View 中有定义：

```java
public void clearFocus() {
    if (DBG) {
        System.out.println(this + " clearFocus()");
    }
    clearFocusInternal(null, true, true);
}
```
它也会调用 clearFocusInternal() 方法，只不过传入的参数和 unFocus() 中的不同。其作用是如果此 View 有焦点的话，清除自身的焦点并将焦点路径删除。

## 焦点路径

我们知道，对于一个窗口来说，其内部所有的 View 都是通过树的形式来组织的。每一个 View 都是这棵树的一个节点，DecorView 是这棵树的根节点，其他 ViewGroup 是这棵树的中间节点，View 是这棵树的子节点。前面说了，一个窗口中只能有一个 View 获取焦点，我们可以把从根节点到这个获取焦点的节点路径称之为焦点路径。很显然，焦点路径的起始节点是根节点，中间节点是各种类型的 ViewGroup，终止节点是那个拥有焦点的 View。

在每次重新确定焦点视图后，系统都会建立一条这样一条焦点路径。系统是怎样标识这条路径的呢？就是前面讲到的 mFocused 变量。mFocused  的含义之前说了，指向的是这个 ViewGroup 的自身具有焦点或者后代具有焦点的子 View。而中间节点和根节点都是 ViewGroup 的子类，它们都持有 mFocused 变量，因此从根节点开始，通过 mFocused 变量可以到达最终拥有焦点的那个节点。因为一个窗口中只有一个焦点视图，因此焦路径也只有一条，而焦点视图是会发生变化的，所以可以推测，系统在建立新的焦点路径时，会将原来的那条焦点路径删除。

现在我们来分析系统建立和删除焦点路径的过程。我们知道，View 成功获取焦点是在 handleFocusGainInternal() 中进行的，因此我们可以以这个方法为切入点追踪系统建立焦点路径的过程。handleFocusGainInternal() 定义如下：

```java
void handleFocusGainInternal(@FocusRealDirection int direction, Rect previouslyFocusedRect) {
    if (DBG) {
        System.out.println(this + " requestFocus()");
    }
    if ((mPrivateFlags & PFLAG_FOCUSED) == 0) {
        mPrivateFlags |= PFLAG_FOCUSED;
        View oldFocus = (mAttachInfo != null) ? getRootView().findFocus() : null;
        if (mParent != null) {
            mParent.requestChildFocus(this, this);
            updateFocusedInCluster(oldFocus, direction);
        }
        if (mAttachInfo != null) {
            mAttachInfo.mTreeObserver.dispatchOnGlobalFocusChange(oldFocus, this);
        }
        onFocusChanged(true, direction, previouslyFocusedRect);
        refreshDrawableState();
    }
}
```
这个方法做的事很简单：将这个 View 的 PFLAG_FOCUSED 标志位设为 1，也就是让其成功地获取了焦点；然后调用其父 View 的 requestChildFocus() 方法。这个方法有两个参数，第一个参数传入的是调用父 View 该方法的那个子 View；后面那个参数传入的是具有焦点的那个 View。因为此 View 既是调用父 View 方法的 View，又是具有焦点的 View，所以两个参数传入的都是自身。这个方法的定义在 ViewGroup 中：

```java
public void requestChildFocus(View child, View focused) {
    if (DBG) {
        System.out.println(this + " requestChildFocus()");
    }
    if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
        return;
    }
    // Unfocus us, if necessary
    super.unFocus(focused);
    // We had a previous notion of who had focus. Clear it.
    if (mFocused != child) {
        if (mFocused != null) {
            mFocused.unFocus(focused);
        }
        mFocused = child;
    }
    if (mParent != null) {
        mParent.requestChildFocus(this, focused);
    }
}
```

这个方法的逻辑很清晰：先是调用父类 View 的 unFoucs() 方法执行清除自身焦点的逻辑，
当然，可能这个 ViewGroup 可能本来就不具有焦点；再判断这个 ViewGroup 的 mFocused 变量和调用此方法的子 View 是不是同一个，如果不是，那么就在 mFocused 不为空的前提下调用其 unFocus() 方法，再将子 View 的值赋给 mFocused；最后，如果这个 ViewGroup 的 mParent 不为空，递归调用其 requestChildFocus() 方法，并分别将自身和具有焦点的那个 View 作为参数传入。

结合之前对 unFocus() 的分析可知：如果不考虑递归，requestChildFocus() 的效果是：将该 ViewGroup 的 mFocused 变量置为调用此方法的子 View，并清除自身可能拥有的焦点；如果考虑递归，这个方法的效果是，清除自身可能拥有的焦点，并将上一条焦点路径上从该 ViewGroup 节点开始的所有后续中间节点的 mFocused 变量置为 null，将上一条焦点路径的终止节点的焦点清除，且将从根节点开始一直到此节点的这条路径上的所有节点的 mFocused 值赋为该节点在此路径上的后继节点。因为最开始调用 requestChildFocus() 方法的 View 是那个具有焦点的 View，因此，最终的效果便是：清除上一次的焦点路径，并建立一条从根节点到具有焦点的 View 的焦点路径。

## 确定焦点的过程

我们知道，如果我们新建一个只有一个 EditText 的 Activity，这个 EditText 在 Activity 启动时会自动获取焦点，那么系统是如何让这个 EditText 获取焦点的呢？现在我们来分析下。

由之前的分析可知，View 通过是否设置标志位 PFLAG_FOCUSED 来标志自己是否拥有焦点，而在 View 的源码中发现只有 handleFocusGainInternal() 方法给 View 设置了这个标志位，因此可以认为 handleFocusGainInternal() 就是给 View 设置焦点的方法。我们可以通过 Android Studio 调试器的调用栈来分析这个方法的调用过程。我们在 Activity 中新建一个 EditText，因为当 Activity 启动时这个 EditText 会获取焦点，因此它的 handleFocusGainInternal() 方法会被调用。我们在 handleFocusGainInternal() 方法处设置断点，以调试模式启动这个 app。我们会发现程序一启动，就在断点处终止了，我们看看此时调试器中的方法栈：

![](/img/in-post/post_android_view_focus/post-1.png)

根据方法栈我们可以一直追踪到 ViewRootImpl#focusableViewAvailable()，是这个方法触发了焦点的确定过程：

```java
@Override
public void focusableViewAvailable(View v) {
    checkThread();
    if (mView != null) {
        if (!mView.hasFocus()) {
            if (sAlwaysAssignFocus) {
                v.requestFocus();
            }
        } else {
            // the one case where will transfer focus away from the current one
            // is if the current view is a view group that prefers to give focus
            // to its children first AND the view is a descendant of it.
            View focused = mView.findFocus();
            if (focused instanceof ViewGroup) {
                ViewGroup group = (ViewGroup) focused;
                if (group.getDescendantFocusability() == ViewGroup.FOCUS_AFTER_DESCENDANTS
                        && isViewDescendantOf(v, focused)) {
                    v.requestFocus();
                }
            }
        }
    }
}
```

这个方法调用了参数 v 的 requestFocus() 方法，v 的实际类型是 DecorView，它就是根布局。因此，系统确定焦点的过程其实就是简单的调用了根部局的 requestFocus() 方法。因此，结合前面对 requestFocus() 和 onRequestFocusInDescendants() 方法的分析可知，根部局的这个方法会通过深度优先的方式调用视图树中各节点的 requestFocus() 方法。在时间顺序上，第一个该方法返回 true 的节点在成功地获取了焦点的同时会遍历终止，然后走建立新焦点路径和删除旧焦点路径的流程。在此节点之后 requestFocus() 方法返回 true 的节点将成为焦点路径上的中间节点，最后一个返回 true 的节点便是焦点路径的起始节点，也就是根节点。

## 总结

对于 View 来说，是否具有焦点将影响它的绘制，如果这个 View 是 EditText，是否具有焦点还会决定它能否获取键盘输入。如果我们想通过代码在某个没有焦点的 EditText 上弹出键盘，仅仅通过 InputMethodManager#showSoftInput() 是不够的，还需在此之前调用其 requestFocus() 方法让其获取焦点。而对于 ViewGroup 来说，焦点通常不会对绘制和输入带来影响，但是它需要根据自身特点处理后代的焦点问题，比如，对于 ViewPager，在换页的时候，如果不重新分配焦点的话，焦点可能还会停留在上一个页面中，因此需要调用下一个页面的根部局的 requestFocus() 让它或者它的后代拥有获取焦点的机会，我们可以在 ViewPager 的源码中看到这种处理。

