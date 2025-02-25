---
title: Android View体系 - performTraversals篇
date: 2022-03-15 19:58:43
header-img: "cover.webp"
tags:
- Android
---

# 前言
上面文章：[Android View体系 - 启动篇](/android-view-startup)中，我们知道`View`的三大流程都是performTraversals方法调用的。

performTraversals作为`View`三大流程的入口方法，只要子`View`执行了`requestLayout`，就必然会调到`ViewRootImpl`的performTraversals。如此重要的方法，除了协调测量、布局和绘制这三大流程以外，performTraversals还做了什么？

本篇文章将聚焦performTraversals方法，试图梳理其大致的流程，了解performTraversals的主要职责。

# 确认窗口大小
performTraversals作为`View`三大流程的入口方法，不仅在应用正常运行的时候会被调用，启动的时候也会调用。

因此必然有涉及到确认窗口大小的逻辑，这也是performTraversals第一步要做的。

```java
// ViewRootImpl.java

private void performTraversals() {
    final View host = mView;
    ...
    int desiredWindowWidth;
    int desiredWindowHeight;
    ...
    WindowManager.LayoutParams lp = mWindowAttributes;
    ...
    Rect frame = mWinFrame;
    if (mFirst) {
        ...
        mLayoutRequested = true;
        ...
        final Configuration config = mContext.getResources().getConfiguration();
        if (shouldUseDisplaySize(lp)) {
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
               || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
            desiredWindowWidth = dipToPx(config.screenWidthDp);
            desiredWindowHeight = dipToPx(config.screenHeightDp);
        } else {
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
        }
    } else {
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            ...
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }
    ...
}

private static boolean shouldUseDisplaySize(final WindowManager.LayoutParams lp) {
    return lp.type == TYPE_STATUS_BAR_ADDITIONAL  // 窗口类型：状态栏不在屏幕顶部
            || lp.type == TYPE_INPUT_METHOD       // 窗口类型：输入法相关，用于输入法应用
            || lp.type == TYPE_VOLUME_OVERLAY;    // 窗口类型：音量窗口
}
```

首先拿到窗口的大小，即mWinFrame，这个变量在setView的时候与`WindowManagerService`跨进程调用的时候初始化的。

从代码可以看出，窗口大小的确认分两种情况，一种是首次进入performTraversals，另一种是非首次进入performTraversals。

## 首次调用performTraversals
先看看首次进入的情况。

首次进入performTraversals又分为三种情况：

1. 特殊窗口类型
2. 顶级`View`的宽高为`WRAP_CONTENT`
3. 一般情况

### 特殊窗口类型
一般用于系统应用，不做讨论

### 顶级View的宽高为WRAP_CONTENT

因为顶级`View`不一定是`DecorView`，有可能是我们自己加的悬浮窗，又或者是一个`Dialog`，所以需要适配宽高为`WRAP_CONTENT`的情况。

这种情况下，宽高会被设置为去除状态栏的屏幕宽高。

### 一般情况
一般情况下，`LayoutParams`为(`MATCH_PARENT`, `MATCH_PARENT`)。

这种情况下，宽高会被设置为`WindowManagerService`返回的宽高，即frame变量，一般与屏幕宽高相同。

## 非首次调用performTraversals
再来看看非首次进入的情况。

非首次进入相对较为简单，只是跟旧的结果比较。如果不同，就将标志位`windowSizeMayChange `置为true。

确认窗口大小的总体流程如下：

![perform_traversals_compute_frame_size.drawiow](perform_traversals_compute_frame_size.drawio.png)

# 预测量
为什么需要预测量？

首先需要明确，顶级`View`不一定是`DecorView`，也有可能是我们展示的一个`Dialog`、悬浮窗，又或者是其他情形。

简单阅读`Dialog`的代码发现，其实它最终还是调用`WindowManager`的addView方法，最终走到`WindowManagerGlobal`，将`Dialog`的`View`与新建的`ViewRootImpl`绑定在一起，然后添加到当前的窗口。

从`Dialog`的代码可以知道，一个窗口下是可以有多个`ViewRootImpl`的，这些`ViewRootImpl`所管理的顶级`View`不一定是`DecorView`。所以`ViewRootImpl`在设计时，不能只考虑`DecorView`这种情况，还要考虑`Dialog`、悬浮窗等情况。

`DecorView`和其他`View`有什么区别？一个重要的区别是，`DecorView`需要撑满窗口，而其他`View`不一定要撑满窗口。

以`Dialog`为例，当`Dialog`上有一行很长的文字，但却撑满了屏幕的宽度，用户体验不好。用代码来描述这种情况就是：

1. `Dialog`的宽度设置为`WRAP_CONTENT`；
2. 上面的`desiredWindowWidth `很接近屏幕尺寸；
3. `Dialog `使用`desiredWindowWidth `和`desiredWindowHeight `去测量；

为了避免这种用户体验不佳的情况，performTraversals需要根据给定的顶级`View`优化`desiredWindowWidth `和`desiredWindowHeight `，这也是performTraversals的职责之一：预测量。

背景介绍完毕，我们来看看代码：

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        ...
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    ...
}
```
layoutRequested为true，走if分支，最终会调用到measureHierarchy方法。看看measureHierarchy方法：

```java
// ViewRootImpl.java

private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;
    
    boolean goodMeasure = false;
    ...
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
        ...
        final DisplayMetrics packageMetrics = res.getDisplayMetrics();
        res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
        int baseSize = 0;
        if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
            baseSize = (int)mTmpValue.getDimension(packageMetrics)
        }
        ...
        if (baseSize != 0 && desiredWindowWidth > baseSize) {
            childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight,
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  // ①
            ...
            if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                goodMeasure = true;
            } else {
                baseSize = (baseSize+desiredWindowWidth)/2; 
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  // ②
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                }
            }
        }
    }
    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  // ③
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
    
    return windowSizeMayChange;
}
```

可以看到measureHierarchy内部有①、②、③处调用performMeasure进行测量操作。

##  预测量①
如果`View`的宽度是`WRAP_CONTENT`的，就会进行第一次预测量。

在第一次预测量，会做最乐观的估计，先以baseSize（`R.dimen.config_prefDialogWidth`）作为宽度，让`View`去测量。

如果baseSize足够`View`去展示，就使用baseSize作为宽度，预测量结束；否则就进行第二次预测量。
	
## 预测量②
如果给定的baseSize不足以让`View`展示，就会进行第二次预测量。如何知道给定的baseSize不足以让`View`测量？

如果`View`对给定的baseSize不满意，就会反馈在state中，可以调用getMeasuredWidthAndState方法来获取。如果测量结果与`View.MEASURED_STATE_TOO_SMALL`按位与的结果不为0，说明`View`对给定的大小不满意，即给定的尺寸不够展示
	
第二次进一步扩大了baseSize的大小，以原有baseSize和desiredWindowWidth的均值为baseSize，再一次进行尝试测量。
	
经过测量后，如果给定的尺寸足以让`View`展示，就以给定的尺寸为宽度，并将goodMeasure置为true，预测量结束；否则就进入第三次预测量。
	
## 预测量③
前两次预测量发现给定的baseSize都不足以让`View`展示，或者是`View`的宽度不是`wrap_content`，因此使用接近屏幕宽度的desiredWindowWidth进行测量，将desiredWindowWidth设置为宽度结果，然后结束流程。

预测量的主要流程如图：

![perform_traversals_permeasure](perform_traversals_permeasure.drawio.png)


# 三大流程 - 测量
预测量完毕，顶级`View`的尺寸已经确认完毕，可以进行测量步骤：

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    if (mFirst || windowShouldResize || viewVisibilityChanged || cutoutChanged || params != null
            || mForceNextWindowRelayout) {
        ...
        if (mWidth != frame.width() || mHeight != frame.height()) {
            mWidth = frame.width();
            mHeight = frame.height();
        }
        
        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(...);
            if (focusChangedDueToTouchMode || ...) {
                ...
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
                ...
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                ...
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;
                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                
                if (measureAgain) {
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }
            }
        }
    } else {
        maybeHandleWindowMove(frame);
    }
    ...
}
```

能够进行测量，需要满足六个条件其中之一：

| 条件 | 含义 | 备注 |
|---|---|---|
| mFirst | 首次进行performTraversals | - |
| windowShouldResize | 窗口尺寸是否需要改变 | 跟预测量有关 |
| viewVisibilityChanged | `View`的可见性是否改变 | - |
| cutoutChanged | 刘海屏的缺口是否改变 | - |
| params != null | 窗口的lp是否改变 | - |
| mForceNextWindowRelayout | WMS是否强制改变了窗口尺寸 | - |

紧接着如果不是stop状态，就正式开始测量步骤。

## 测量 - 不带权重
如果布局不带有权重（weight），那只需一次测量即可。

## 测量 - 带有权重
经过一次测量后，如果发现`LayoutParams`中带有宽度权重 or 高度权重，则需要根据测量的宽度 or 高度乘以权重值，然后再执行一次测量。

# 三大流程 - 布局
经过测量后，就可以进行布局。

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ...
    } 
    ...
}
```

# 三大流程 - 绘制
经过布局后，就可以进行绘制。

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    mFirst = false;
    
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    
    if (!cancelDraw) {
        ...
        performDraw()
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        }
        ...
    }
}
```

# 总结
总体而言，performTraversals的逻辑大致可以分为五块：

* 确认窗口大小
* 预测量
* 测量
* 布局
* 绘制

首先，为了确认窗口大小，performTraversals会根据`LayoutParams`的宽高及其各种标志位，配合WMS给定的窗口大小，最终计算出可供`View`展示的大小；

然后呢，为了优化顶级`View`不是`DecorView`下的展示体验，performTraversals会进行预测量，经过预测量，会给出一个合理的尺寸，让`View `进行测量；

经过预测量后，接下来就是`View`的三大流程：测量、布局以及绘制。

此后performTraversals结束。
