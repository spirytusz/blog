---
title: Android View体系 - 布局篇
date: 2022-04-05 22:13:23
header-img: "cover.webp"
tags:
- Android
---

# 前言
在上篇文章[Android View体系 - 测量篇](android-view-measure/)中，我们知道：每个`View`都会接收来自父`View`的`MeasureSpec`来进行测量，在确保自身能够独立完成测量逻辑的同时，也能向下推导，促使其子`View`的测量。

与测量流程同为`View`三大流程的布局流程，是如何独立完成自身的布局流程，又能向下推导，促使孩子节点的布局的？本文将围绕着`View`布局流程的源码，尝试探索`View`的布局流程。

# View的布局
一棵`View`树的布局起源于`ViewRootImpl`的performTranversals方法：

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    performLayout(lp, mWidth, mHeight);
    ...
}
```

在performTraversals内部会调用performLayout方法，从根`View`开始测量：

```java
// ViewRootImpl.java

private void performLayout(WindowMa
        int desiredWindowHeight) {
    ...
    mInLayout = true;
    final View host = mView;
    ...
    try {
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        mInLayout = false;
        
        // 处理布局流程中requestLayout
        ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(
                mLayoutRequesters,
                false
        );
        ...
    } finally {
        ...
    }
    ...
}
```

在performLayout内部，逻辑大致可以分为两块：

1. 执行根`View`的layout方法开始布局；
2. 处理在布局过程中，当前`View`树的所有requestLayout请求；

第2块不是主流程，主要来看第1块：

```java
// View.java

public void layout(int l, int t, int r, int b) {
    ...
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        ...
    }
    
    ...
}
```

在`View`的layout方法中，主要的逻辑分为两块：

1. setFrame - 为当前`View`设置布局的位置
2. onLayout - 为当前`View`的子`View`布局

## setFrame
先看看`View`的setFrame方法。

```java
// View.java 

protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        ...
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        ...
    }
    
    return changed;
}
```

setFrame方法内部的逻辑相对较为简单，内部逻辑会与上次设置的left、top、right、bottom进行比对，如果发生了改变，就会设置新值，否则啥也不干。

## onLayout
再来看看`View`的onLayout方法：

```java
// View.java

protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

可以看到`View`的onLayout是一个空实现。这也可以理解，因为onLayout是用来布局子`View`的，一个`View`不一定会有子`View`，`ViewGroup`才有可能有。

# ViewGroup的布局
`ViewGroup`也是`View`，不同的是，它能够承载其他`View`。`ViewGroup`布局的入口也是layout方法，它重写了（也是唯一一个重写layout的`View`子类）layout方法：

```java
// ViewGroup.java 

public final void layout(int l, int t, int r, int b) {
    if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
        if (mTransition != null) {
            mTransition.layoutChange(this);
        }
        super.layout(l, t, r, b);
    } else {
        // record the fact that we noop'd it; request layout when transition finishes
        mLayoutCalledWhileSuppressed = true;
    }
}
```

从代码逻辑可以看到，如果一个`ViewGroup`没有通过suppressLayout方法来禁用布局，且动画没有在改变布局的位置时，就会调用超类`View`的layout方法，因此所做的逻辑大致与其超类`View`的基本一致。

## onLayout
`ViewGroup`的layout方法逻辑与`View`的基本一致，`View`的layout方法有两个关键的方法：setFrame和onLayout。setFrame方法`ViewGroup`没有重写，我们来看看`ViewGroup`的onLayout方法：

```java
// ViewGroup.java

protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```

可以看到，`ViewGroup`的onLayout方法变成了抽象方法，子类必须实现它，以完成布局`ViewGroup`子`View`的逻辑。

以`FrameLayout `为例：

```java
// FrameLayout.java

protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}

void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    final int count = getChildCount();
    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();
    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();
            int childLeft;
            int childTop;
            ...
            switch (...) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }
            switch (...) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

虽然代码很长，但是能看出来这段代码做了三件事：

1. 计算每个子`View`的left
2. 计算每个子`View`的top
3. 调用子`View`的layout方法布局子`View`

计算子`View`的left依托于`FrameLayout`的left、水平方向的gravity（左对齐、水平居中、右对齐）

计算子`View`的top依托于`FrameLayout`的top、竖直方向的gravity（顶部对齐、竖直居中、底部对齐）

计算完子`View`的left和top后，依据子`View`的measuredWidth和measuredHeight，就能计算出子`View`的right和bottom，因此就能调用子`View`的layout方法为子`View`进行布局。

# 坐标系、padding和margin
通读完`View`、`ViewGroup`和`FrameLayout`的布局代码，left、top、right、bottom这几个代表什么意思？
如果把一个`View`理解成一个矩形：

* left - 代表这个矩形的左边与父`View`左边的距离
* top - 代表这个矩形的上边与父`View`上边的距离
* right - 代表这个矩形的右边与父`View`左边的距离
* bottom - 代表这个矩形的下边与父`View`上边的距离

可以看到，left、top、right、bottom是基于子`View`相对于父`View`的相对坐标系上的定义。相对坐标系的好处在于

1. 父`View`可以对子`View`的布局位置进行限制;
2. 相比于绝对坐标系（以屏幕为参考系），相对坐标系在递归时更容易对参数进行处理

此外，在`View`、`ViewGroup`和`FrameLayout`的布局代码中，还出现了padding和margin：

* padding - 父`View`的空出的位置，压缩子`View`的布局空间
* margin - 子`View`主动让出的空间

一图胜千言：

![](frame_layout_child_layout.drawio.svg)

# View树的布局
了解了坐标系、padding和margin后，就可以从`FrameLayout`的布局代码，还原出整棵`View`树的布局流程：

1. 父`View`调用setFrame为自己圈定布局的矩形范围；
2. 以父`View`的left、top作为原点，根据padding将子`View`的布局限定在一个矩形区域内；
3. 在这个矩形区域内，根据子`View`的margin、measuredWidth和measuredHeight，布局子`View`；

总而言之，一棵`View`树的布局流程如图：

![](view_tree_layout.drawio.svg)

# 总结
`View`布局的总体流程相较于`View`测量的而言较为简单，主要分为是否有子`View`这两种情况：

* 有子`View` - `ViewGroup`会首先调用setFrame圈住一个(left, top, right, bottom)的矩形区域，再通过onLayout调用子`View`的layout方法为子`View`布局；
* 无子`View` - 没有子`View`的`View`只会调用setFrame方法，然后调用一般为空实现的onLayout方法。

而解决**画在哪**这个问题的关键在于setFrame方法，setFrame方法的参数left、top、right和bottom是建立在以当前`View`的父`View`为参考系的相对坐标系，从而使得`View`能够无需关心屏幕坐标及屏幕尺寸，能够在父`View`给定的(left, top, right, bottom)的约束下进行布局。







