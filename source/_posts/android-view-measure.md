---
title: Android View体系 - 测量篇
date: 2022-03-22 20:39:18
header-img: "cover.webp"
tags:
- Android
---

# 前言
`View`的三大流程之一：测量要解决的问题就是**画多大**的问题。我们都知道，Android的`View`结构是一个多叉树，在这么一个数据结构之下，对于每个节点`View`，它们是如何独立解决**画多大**这个问题呢？

本文将从源码的角度，分析`View`的测量代码，尝试探索`View`是如何解决**画多大**的问题。

# View的测量
从上篇文章：[Android View体系 - performTraversals篇](/android-view-perform-traversals)可以知道，测量的入口在`ViewRootImpl`的performTraversals方法内的performMeasure方法，点进去看看：

```java
// ViewRootImpl.java

public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();
...
private void performTraversals() {
    ...
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    ...
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}
```

performTraversals内部调用了performMeasure方法，而performMeasure又调用了`View`的measure方法：

```java
//  View.java

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
}
```

首先它是`public final`的，外部可以调用，但不允许子类重写，主导`View`的测量流程：

```java
// View.java

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
    
    final boolean specChanged = widthMeasureSpec != mOldWidthM
            || heightMeasureSpec != mOldHeightMeasureSpec;
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
            && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
            && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
    final boolean needsLayout = specChanged
            && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
            
     if (forceLayout || needsLayout) {
         mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
         
         int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
         if (cacheIndex < 0 || sIgnoreMeasureCache) {
             onMeasure(widthMeasureSpec, heightMeasureSpec);
             mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
         } else {
             long value = mMeasureCache.valueAt(cacheIndex);
             setMeasuredDimensionRaw((int) (value >> 32), (int) value);
             mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
         }
         ...
     }
     ...
     mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
             (long) mMeasuredHeight & 0xffffffffL);
}
```

从代码中可以看出，只有`forceLayout || needsLayout`为true，才会有可能去执行测量逻辑。

* forceLayout - 取决于是否有`PFLAG_FORCE_LAYOUT `这个flag，一般在requestLayout方法调用后会带这个flag
* needsLayout - 取决于传递过来的`MeasureSpec`:
	* specChanged - `MeasureSpec`是否有改变
	* isSpecExactly - 宽高都为`match_parent`
	* matchesSpecSize - 宽高是否跟上次的测量结果相同

进入到if分支，如果是强制布局，就会走测量流程：调用onMeasure；否则就会使用上次的缓存，调用setMeasuredDimensionRaw保存结果。

最后将测量结果保存到缓存中，测量结束。

# MeasureSpec
在测量过程中，频繁出现`MeasureSpec`的身影，`MeasureSpec`是什么？我认为是父`View`对子`View`的约束，子`View`需要在这个约束之下进行测量。`MeasureSpec`的结构如下：

![](measurespec_struct.svg)

可以从图中看出：

* size ∈ [1, 2^30^ -1]
* mode ∈ { EXACTLY, AT_MOST, UNSPECIFIED }

size代表尺寸，mode是`MeasureSpec`内部定义的与`LayoutParams`相关的三种模式，如下：

| MeasureSpec.mode | 对应的LayoutParams | 含义 |
|---|---|---|
| `EXACTLY ` | `MATCH_PARENT`、指定值 | 测量前`View`就知道它该多大 |
| `AT_MOST` | `WRAP_CONTENT` | 测量前`View`不知道该多大，自适应模式，尺寸不能超过size |
| `UNSPECIFIED` | / | 父`View`对子`View`的尺寸不做限制 |

mode和size的解封装，只需要使用位操作即可：

```java
// View.MeasureSpec.java
private static final int MODE_SHIFT = 30;

private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

public static int makeMeasureSpec(
    @IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, 
    @MeasureSpecMode int mode
) {
    ...
    return (size & ~MODE_MASK) | (mode & MODE_MASK);
}

@MeasureSpecMode
public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}

public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
}
```

## onMeasure
measure方法是final的，不允许子类重写。为了子类能够自定义他们自己的测量逻辑，`View`暴露了一个protected的onMeasure方法，允许子类自定义自己的测量流程，但不允许外部调用。

onMeasure方法的默认实现，仅仅只是设置了一下测量结果：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    ...
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
    
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

# ViewGroup的测量
如果说没有子`View`的`View`是叶子节点，那么`ViewGroup`就是有孩子节点的`View`。

既然`View`的数据结构是一个多叉树，那么包含有子`View`的`ViewGroup`是如何测量的呢？

显而易见，`ViewGroup`需要先测量它的所有子`View`，然后根据测量结果，才能测量自身。如果把`View`当做一棵树来看，测量的过程就是后序遍历。

因此，对于每个`View`节点，其测量过程必然是这样的：

![](view_group_measure.drawio.svg)


检查`ViewGroup`的代码发现，`ViewGroup`并没有重写onMeasure方法，需要找`ViewGroup`的子类验证父`View`与子`View`的测量逻辑。以`FrameLayout`为例：

```java
// FrameLayout.java

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    int maxHeight = 0;
    int maxWidth = 0;
    
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        maxWidth = Math.max(maxWidth,
                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
        maxHeight = Math.max(maxHeight,
                child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
        ...
    }
    
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
        resolveSizeAndState(maxHeight, heightMeasureSpec,
                childState << MEASURED_HEIGHT_STATE_SHIFT));
    ...
}
```

`FrameLayout`在测量的时候，先会遍历其子`View`，调用measureChildWithMargins方法，挨个进行测量，找到最大的高度和宽度，最后调用setMeasuredDimension，将最大高度和最大宽度设置为测量结果，测量基本结束。

从整个过程来看，关键方法就是measureChildWithMargins，其作用就像它的命名一样，带上margin测量子`View`：

```java
// ViewGroup.java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

查看measureChildWithMargins方法发现，它会计算出新的`MeasureSpec`，然后调用子`View`的measure方法，把`MeasureSpec`传递给子`View`去测量。而计算出新的`MeasureSpec`的关键逻辑，都藏在了getChildMeasureSpec方法内：

```java
// View.java

public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

getChildMeasureSpec方法不长，但它在父子`View`的测量中起着承上启下的关键作用，因为它是父`View`（`ViewGroup`）对子`View`测量约束的直接体现。

为什么这么说？查看这个方法的入参，大致可以分为两部分:

* spec - 父`View`的`MeasureSpec`
* padding, childDimension - 子`View`的`LayoutParams`

本质而言，这个方法就是就是将父`View`的`MeasureSpec`和子`View`的`LayoutParams`转化成新的`MeasureSpec`，最后将这个新的`MeasureSpec`通过子`View`的measure方法传递给子`View`，以达到约束子`View`测量的目的。这个新的`MeasureSpec`就是父`View`对子`View`测量行为的约束。

那么，getChildMeasureSpec是如何将父`View`的`MeasureSpec`和子`View`的`LayoutParams`转化成新的`MeasureSpec`呢？这张表格可以概括：

<font size="1">
<table cellspacing="0" cellpadding="0" style="border-collapse:collapse;" id="get_child_measure_spec">
    <tbody>
    <tr>
        <td class="diagonalFalling">
            <span style="float:left;margin-top:20px;">childLayoutParam</span>
            <span style="float:right">parentMode</span>
        </td>
        <td style="text-align:center">EXACTLY</td>
        <td style="text-align:center">AT_MOST</td>
        <td style="text-align:center">UNSPECIFED</td>
    </tr>
    
    <tr>
        <td style="text-align:center">match_parent</td>
        <td style="text-align:center">EXACTLY + size</td>
        <td style="text-align:center">AT_MOST + size</td>
        <td style="text-align:center">UNSPECIFIED + size</td>
    </tr>
    <tr>
        <td style="text-align:center">wrap_content</td>
        <td style="text-align:center">AT_MOST + size</td>
        <td style="text-align:center">AT_MOST + size</td>
        <td style="text-align:center">UNSPECIFIED + size</td>
    </tr>
    <tr>
        <td style="text-align:center">指定值</td>
        <td style="text-align:center">EXACTLY + childDimension</td>
        <td style="text-align:center">EXACTLY + childDimension</td>
        <td style="text-align:center">EXACTLY + childDimension</td>
    </tr>
</tbody></table>
</font>

其中：

* size是父`View`的尺寸减去padding
* childDimension是子`View`的`LayoutParams`

通过getChildMeasureSpec将父`View`的`MeasureSpec`和子`View`的`LayoutParams`转化成新的`MeasureSpec`后，父`View`在measureChildWithMargins方法中，直接调用子`View`的measure方法，子`View`开始测量。

更一般地，一个`View`在收到测量请求后，会首先计算出新的`MeasureSpec`，然后再传递给子`View`，子`View`又开始测量；如此往复，直至传递到叶子节点 —— 一个没有子`View`的`View`。

叶子节点`View`测量完毕后，调用setMeasuredDimension将测量结果保存下来，叶子节点`View`测量完毕，此后调用链开始回归。

调用链回归是自底向上的，从底层到顶层逐级测量并调用setMeasuredDimension将测量结果保存下来，直至回归到顶级`View`。

因此，一棵`View`树的测量，大致可以分为两个流程：

* 自顶向下传递测量请求，传递计算出来的新的`MeasureSpec`
* 自底向上逐级测量，并设置测量结果

以上过程可以用一张图来概括：

![](view_recursively_draw.drawio.svg)

# 打破约束？
管中窥豹，从`FrameLayout`的测量逻辑中，我们可以发现一棵`View`树的测量是：

* 从根节点到叶子结点层层传递约束、发起测量请求
* 从叶子结点到根节点执行测量保存结果

那如果子`View`打破这种约束会怎么样？或者说子`View`能不能打破这个约束？

## MEASURED_STATE_TOO_SMALL
还是以`FrameLayout`为例，测量结束后，最终会调用setMeasuredDimension设置测量结果：

```java
// FrameLayout.java

setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
```

`FrameLayout`在测量完毕后，调用resolveSizeAndState方法将测量结果重新包装一次，而后再将其设置为测量结果，看看resolveSizeAndState内部做了什么：

```
// View.java

public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

入参是：

* size - 期望的测量结果
* measureSpec - 父`View`传递给`FrameLayout`的`MeasureSpec`
* childMeasuredState - `FrameLayout`子`View`的测量状态

其返回值取决于父`View`给的`MeasureSpec`和期望的测量结果：

* 当父`View`的测量模式是`AT_MOST`时：
    
    父`View`在测量之前不知道自己多大，但父`View`的父`View`又把它的尺寸限制在了specSize以内。如果此时期望的测量结果**size > specSize**，就会被带上**MEASURED\_STATE\_TOO\_SMALL**的标记；
    

* 当父`View`的测量模式是`EXACTLY`时：
   
    父`View`在测量前就知道自己有多大，`FrameLayout`的测量结果就是父`View`传递给它的specSize；


* 当父`View`的测量模式是`UNSPECIFIED`时：
    
    父`View`对`FrameLayout`的尺寸不做限制，要多大就多大；

从代码逻辑上来看，`FrameLayout`只有在父`View`的测量模式是`AT_MOST`的时候，才有可能打破约束。打破约束的后果就是测量结果被带上**MEASURED\_STATE\_TOO\_SMALL**的标记，用于告诉父`View`给定的空间太小。

写个布局验证一下：
```xml
<LinearLayout
    android:layout_width="200px"
    android:layout_height="200px"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent">
    <com.example.myapplication.CustomFrameLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">
        <ImageView
            android:id="@+id/iv"
            android:layout_width="300px"
            android:layout_height="300px" />
    </com.example.myapplication.CustomFrameLayout>
</LinearLayout>
```

重写`FrameLayout`的onMeasure方法，打印出测量结果：

```kotlin
class CustomFrameLayout {
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)

        Log.d(TAG, "onMeasure() >>> width=$measuredWidth, height=$measuredHeight, state=$measuredState")
        findViewById<View>(R.id.iv)?.apply {
            Log.d(TAG, "onMeasure() >>> iv_width=$measuredWidth, iv_height=$measuredHeight, iv_state=$measuredState")
        }
    }
}
```

从日志来看，`FrameLayout`确实对这种情况不对子`View`的大小做处理，而是带上**MEASURED\_STATE\_TOO\_SMALL**的标记向上通知它的父`View`。

```
CustomFrameLayout: onMeasure() >>> width=200, height=200, state=16777472
CustomFrameLayout: onMeasure() >>> iv_width=300, iv_height=300, iv_state=0
CustomFrameLayout: onMeasure() >>> width=200, height=200, state=16777472
CustomFrameLayout: onMeasure() >>> iv_width=300, iv_height=300, iv_state=0
```

然而，打破约束的结果是：

* 子`View`在屏幕中的显示面积为200px*200px
* measuredWidth和measureHeight都为300px

即：

* 测量上，父`View`在测量上没有进一步约束子`View`
* 布局和绘制上，父`View`对子`View`做了进一步约束

## “重写”getChildMeasureSpec
但并不是所有的`ViewGroup`都像`FrameLayout`一样没有约束子`View`的测量结果。例如`RelativeLayout`，上面的`CustomFrameLayout`换成`CustomRelativeLayout`，观察日志：

```
CustomRelativeLayout: onMeasure() >>> width=200, height=200, state=0
CustomRelativeLayout: onMeasure() >>> iv_width=200, iv_height=200, iv_state=0
CustomRelativeLayout: onMeasure() >>> width=200, height=200, state=0
CustomRelativeLayout: onMeasure() >>> iv_width=200, iv_height=200, iv_state=0
```

咦，`ImageView`的measuredWidth和measuredHeight都成200px了。

阅读代码发现，其实`RelativeLayout`“重写”了`View`的静态方法getChildMeasureSpec：

```java
private int getChildMeasureSpec(int child
        int childSize, int startMargin, i
        int endPadding, int mySize) {
    ...
    if (childSize >= 0) {
        childSpecMode = MeasureSpec.EXACTLY;
        if (maxAvailable >= 0) {
            childSpecSize = Math.min(maxAvailable, childSize);
        } else {
            childSpecSize = childSize;
        }
    } else if (childSize == LayoutParams.MATCH_PARENT) {
        ...
    } else if (childSize == LayoutParams.WRAP_CONTENT) {
        ...
    }
    ...
    return MeasureSpec.makeMeasureSpec(childSpecSize, childSpecMode);
}
```

在`RelativeLayout`子`View`的尺寸为指定值的情况下，`RelativeLayout`会以maxAvailable和子`View`的最小值作为specSize，传递给子`View`测量 —— 相当于把子`View`约束在`RelativeLayout`的尺寸下测量。

## 结论

本质而言，`FrameLayout`和`RelativeLayout`对子`View`约束的根本不同在于传递给子`View`的`MeasureSpec`不同：

| 布局 | parentMode = AT_MOST<br> childDimension 为指定值 |
| --- | --- |
| FrameLayout | EXACTLY + childDimension |
| RelativeLayout | EXACTLY + Math.min(childDimension, maxAvailable) |


子`View`能否打破约束取决于父`View`计算子`View``MeasureSpec`的逻辑，需要根据父`View`来具体讨论。具体而言：

`FrameLayout`的子`View`可以打破约束，使得子`View`的测量结果比`FrameLayout`的还要大；

`RelaytiveLayout`的子`View`不可以打破约束，如果`RelaytiveLayout`是`AT_MOST`并且子`View`的期望尺寸比`RelativeLayout`的`MeasureSpec.size`还要大，`RelaytiveLayout`会通过传递给子`View`的`MeasureSpec`，进而限制子`View`的大小。

# 总结
从整体来看，一棵`View`的测量，涉及到传递和回归的过程：

* 传递是指父`View`发起子`View`的测量请求，并传递约束`MeasureSpec`的过程
* 回归是指请求测量和传递`MeasureSpec`到叶子节点后，叶子节点开始测量，并逐级向上回归测量的过程

从个体来看，每个`View`都会接受来自父`View`的约束，`View`会根据约束以及自身的`LayoutParams`独立进行测量。

关于父`View`对子`View`的约束，主要是使用`MeasureSpec`对子`View`的尺寸进行限制，但这种约束，从测量结果上来看，有强制的和非强制的：

* 非强制限制：子`View`的测量尺寸有可能比父`View`的要大 —— 一个例子：`FrameLayout`
* 强制限制：子`View`的测量尺寸必然比父`View`的要小 —— 一个例子：`RelativeLayout`

具体的约束逻辑体现在当前`ViewGroup`的getChildMeasureSpec上。

最终，`View`通过传递约束，回归测量来解决**画多大**的问题。
