---
title: Android View体系 - 绘制篇
date: 2022-04-10 20:22:44
header-img: "cover.jpg"
tags:
- Android
---

# 前言
`View`的绘制流程主要解决的是**画什么**的问题。在`View`树这一结构下，每个`View`是如何解决**画什么**的问题的？本文将从源码的角度，尝试理解`View`树的绘制流程。

# 术语
下文可能会出现这些术语，为了让读者更方便理解，先打通这几个术语：

## 硬件绘制
所谓的硬件绘制（硬件加速），指的是绘制的操作交给GPU，绘制的结果存储在显存中。在API 14及其以后默认开启硬件绘制。

## 软件绘制
所谓的软件绘制，是指绘制的操作交给CPU，底层使用`Skia`实现。

由于硬件加速并不支持所有的2D图形绘制指令（详见[硬件加速](https://developer.android.com/guide/topics/graphics/hardware-accel#unsupported)），于是官方提供了关闭硬件加速的方案，可以根据自身需求，关闭指定`View`，乃至整个App的硬件加速：

| 粒度 | 开启软件绘制的方式 |
| --- | --- |
| Application | `android:hardwareAccelerated="false"` |
| Activity | `android:hardwareAccelerated="false"` |
| Window | 无法停用硬件绘制 |
| View | `View.LAYER_TYPE_SOFTWARE` | 

## 绘制Layer
绘制Layer是专门针对`View`的术语，与软件绘制、硬件绘制没有任何关系，但依赖于软件绘制或硬件绘制。目前共有3种Layer：

* `LAYER_TYPE_NONE` - 默认的Layer，不是离屏缓冲；
* `LAYER_TYPE_SOFTWARE` - 使用CPU进行绘制，绘制结果存在`Bitmap`里，是离屏缓冲；
* `LAYER_TYPE_HARDWARE` - 如果开启硬件加速，则使用GPU进行绘制，绘制结果存在显存中，是离屏缓冲；否则与`LAYER_TYPE_SOFTWARE`行为一致。

为什么要引入绘制Layer这个术语？首先我们要明白，硬件绘制并不支持所有2D图形的操作。当遇到硬件绘制不支持的绘制操作时，为了兼容性，需要暂时让软件绘制来帮忙绘制。软件绘制完毕，返回`Bitmap`，硬件绘制再将这个`Bitmap`绘制进CPU的显存即可。

# 入口
`View`的绘制流程的入口在`ViewRootImpl`的performTraversals中：

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    performDraw();
    ...
}

private void performDraw() {
	...
	final boolean fullRedrawNeeded = mFullRedrawNeeded || mReportNextDraw;
	...
	boolean canUseAsync = draw(fullRedrawNeeded);
	...
}
```

在performTraversals中，会调用到performDraw，最终调用到draw方法：

```java
// ViewRootImpl.java

private boolean draw(boolean fullRedrawNeeded) {
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ...
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    }
    ...
    if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
            scalingRequired, dirty, surfaceInsets)) {
        return false;
    }
}
```

`ViewRootImpl`的draw方法大致逻辑如下:

* 如果当前开启了硬件加速，就调用`ThreadedRender`的draw方法，执行硬件绘制；
* 如果未开启硬件加速，则调用drawSoftware，执行软件绘制；

从`ViewRootImpl`的draw方法可以看到，一棵`View`树的绘制，可以有两种方式：

* 硬件绘制
* 软件绘制

这两种方式是二选一的，取决于`ThreadedRender`的isEnable方法。从`ThreadedRender`的源码可以知道，硬件加速是默认开启，并不可关闭的：

```java
// ThreadedRender.java

class ThreadedRender {

	...

    private boolean mEnabled;

    void setEnabled(boolean enabled) {
        mEnabled = enabled;
    }

    private void updateEnabledState(Surface surface) {
        if (surface == null || !surface.isValid()) {
            setEnabled(false);
        } else {
            setEnabled(mInitialized);
        }
    }

    boolean initialize(Surface surface) throws OutOfResourcesException {
        ...
        updateEnabledState(surface);
        ...
    }

    void updateSurface(Surface surface) throws OutOfResourcesException {
        updateEnabledState(surface);
        ...
    }

    @Override
    public void destroy() {
        ...
        updateEnabledState(null);
        ...
    }

    ...
}
```

看来默认逻辑是走`ThreadedRender`的draw方法，先看看`ThreadedRender`的draw方法

## 入口 - 硬件绘制

```java
// ThreadedRender.java

void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    ...
    updateRootDisplayList(view, callbacks);
    ...
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
	...
    updateViewTreeDisplayList(view);
    ...
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            ...
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            ...
        } finally {
            mRootNode.endRecording();
        }
    }
}

private void updateViewTreeDisplayList(View view) {
	...
    view.updateDisplayListIfDirty();
    ...
}
```

代码中出现了两个名词：`RenderNode`和DisplayList，它们是什么？`RenderNode`是用于构建硬件加速的层级结构，每一个`View`都有一个`RenderNode`，每个`RenderNode`持有一系列绘制指令，称之为DisplayList。

再来看看代码，最终会在`ThreadedRender`的updateRootDisplayList方法中调用：

* updateViewTreeDisplayList - 收集`ViewRootImpl.mView`的绘制指令：DisplayList
* RecordingCanvas.drawRenderNode - 绘制持有绘制指令的`RenderNode`

关键就是：updateViewTreeDisplayList方法该如何逐级搜集绘制指令。

## 入口 - 软件绘制
硬件绘制的入口分析完毕，再来看看软件绘制，软件绘制的入口在`ViewRootImpl`的drawSoftware：

```java
// ViewRootImpl.java

private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
	final Canvas canvas;
	...
	try {
		canvas = mSurface.lockCanvas(dirty);
	}
	... 
	try {
		mView.draw(canvas);
	} finally {
		surface.unlockCanvasAndPost(canvas);
		...
	}
	...
	return true;
}
```

在软件绘制的入口drawSoftware处，会调用`Surface`的lockCanvas方法，锁定`Canvas`，然后执行`View`的draw方法，开始逐级绘制。绘制完毕后，调用`Surface`的unlockCanvasAndPost方法，释放`Canvas`。

# 层级绘制
不论是硬件绘制还是软件绘制，都离不开逐级遍历`View`树。先来看看硬件绘制是如何收集绘制指令的。

## 层级绘制 - 硬件绘制
硬件绘制的层级遍历入口在`View`的updateDisplayListIfDirty方法：

```java
// View.java

public RenderNode updateDisplayListIfDirty() {
	final RenderNode renderNode = mRenderNode;
	...
	if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
        || !renderNode.hasDisplayList()
        || (mRecreateDisplayList)) {

		if (renderNode.hasDisplayList() ...) {

			dispatchGetDisplayList();

			return renderNode;
		}
		mRecreateDisplayList = true;

		int layerType = getLayerType();

		final RecordingCanvas canvas = renderNode.beginRecording(width, height);

		...

        if (layerType == LAYER_TYPE_SOFTWARE) {
            buildDrawingCache(true);
            Bitmap cache = getDrawingCache(true);
            if (cache != null) {
                canvas.drawBitmap(cache, 0, 0, mLayerPaint);
            }
        } else {
        	if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
        		dispatchDraw(canvas);
        		...
        	} else {
        		draw(canvas);
        	}
        }
        renderNode.endRecording();
	} else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

从代码中可以看出，以下三种情况会触发收集绘制指令：

* `mPrivateFlag`没有`PFLAG_DRAWING_CACHE_VALID`标志 - 缓存不可用
* !renderNode.hasDisplayList() - 没有构建过DisplayList
* mRecreateDisplayList - 主动重新构建DisplayList

如果上面三个条件都没满足，说明可以使用上一次的缓存，修改标志位，结束收集DisplayList。

否则，上面三个条件一旦满足其一，就会废弃上次绘制的缓存，并重新记录绘制指令DisplayList。记录DisplayList的过程中，根据`View`不同的layerType来执行不同的逻辑：

* `LAYER_TYPE_SOFTWARE` - 将`View`转化成`Bitmap`，记录到`RecordingCanvas`中
* `LAYER_TYPE_HARDWARE`, `LAYER_TYPE_NONE` - 根据`View`是否需要跳过绘制（一般为`ViewGroup`）来决定是调用dispatchDraw方法还是draw方法。

### 层级绘制 - 硬件绘制 - `LAYER_TYPE_SOFTWARE`
当layerType为`LAYER_TYPE_SOFTWARE`时，主要使用buildDrawingCache方法来将`View`的内容转化为`Bitmap`：

```java
// View.java

public void buildDrawingCache(boolean autoScale) {
    ...
    buildDrawingCacheImpl(autoScale);
    ...
}

private void buildDrawingCacheImpl(boolean autoScale) {
	bitmap = Bitmap.createBitmap(mResources.getDisplayMetrics(),
                width, height, quality);
    bitmap.setDensity(getResources().getDisplayMetrics().densityDpi);
    if (autoScale) {
    	mDrawingCache = bitmap;
    } else {
    	mUnscaledDrawingCache = bitmap;
	}
    ...
    Canvas canvas;
	if (attachInfo != null) {
    	canvas = attachInfo.mCanvas;
    	if (canvas == null) {
        	canvas = new Canvas();
    	}
    	canvas.setBitmap(bitmap);
	} else {
    	canvas = new Canvas(bitmap);
	}
	...
    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        dispatchDraw(canvas);
    } else {
        draw(canvas);
    }
    ...
    if (attachInfo != null) {
       	attachInfo.mCanvas = canvas;
    }
}
```

阅读代码可以看到，`View`使用buildDrawingCacheImpl来构建`Bitmap`。它是如何构建出`Bitmap`的？

1. 构造一个`Bitmap`，宽高与`View`的宽高相同，保存在mDrawingCache或mUnscaledDrawingCache中；
2. 如果是首次进入该方法，则构造一个`Canvas`实例，否则使用上次保存的`Canvas`实例；
3. 将bitmap设置进`Canvas`实例中；
4. `ViewGroup`执行dispatchDraw，`View`执行draw，将`View`的内容绘制在`Bitmap`中；

经过以上四个步骤，`View`的所有内容都已经绘制到了`Bitmap`之中。回到updateDisplayListIfDirty方法，将构造好的`Bitmap`绘制到`RecordingCanvas`中：

```java
// View.java

if (layerType == LAYER_TYPE_SOFTWARE) {
	buildDrawingCache(true);
	Bitmap cache = getDrawingCache(true);
	if (cache != null) {
    	canvas.drawBitmap(cache, 0, 0, mLayerPaint);
	}
}
```

### 层级绘制 - 硬件绘制 - 非`LAYER_TYPE_SOFTWARE`
非`LAYER_TYPE_SOFTWARE`的情况无非就是`LAYER_TYPE_HARDWARE`或`LAYER_TYPE_NONE`，在开启硬件加速的情况下，这两者可以认为是相同的layerType，可以认为就是`LAYER_TYPE_HARDWARE`。

```java
// View.java

if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
	dispatchDraw(canvas);
	...
} else {
	draw(canvas);
}
```

可以看到，当layerType为`LAYER_TYPE_HARDWARE`，`View`会将它的绘制指令以及其子`View`的绘制指令记录到`RecordingCanvas`中。

不难发现，为了解决硬件绘制的兼容性问题（硬件绘制并不支持所有的[绘制指令](https://developer.android.com/guide/topics/graphics/hardware-accel?hl=zh-cn#unsupported)），在硬件绘制时，通过引入`LAYER_TYPE_SOFTWARE`，将自身和子`View`的所有绘制指令交给`Canvas`，把自身和子`View`的内容绘制到`Bitmap`之中，最后再通过`RecordingCanvas.drawBitmap`方法，将对应`View`的内容记录到`DisplayList`中。

所以，在日常开发中，如果用到了不支持的绘制指令，需要手动将`View`的layerType设置为`LAYER_TYPE_SOFTWARE`，以达到获取正确绘制结果的目的。

话说回来，无论layerType是哪一种，在硬件绘制且缓存不可用时，最终都会执行以下语句：

```java
// View.java

public RenderNode updateDisplayListIfDirty() {
	...
	if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
    	dispatchDraw(canvas);
    	...
	} else {
    	draw(canvas);
	}
	...
}

public void draw(Canvas canvas) {
	...
	dispatchDraw(canvas);
	...
}
```

也就是说，绘制的请求是要传递给子`View`的，看看dispatchDraw内部做了什么：

```java
// ViewGroup.java

protected void dispatchDraw(Canvas canvas) {
	final int childrenCount = mChildrenCount;
	final View[] children = mChildren;

	...
	for (int i = 0; i < childrenCount; i++) {
		final View child = children[i];
  		if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
        	drawChild(canvas, child, drawingTime);
    	}
	}
	...
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

在dispatchDraw内部会遍历当前`View`的子`View`，挨个调用其`draw(Canvas, ViewGroup, long)`方法：

```java
// View.java

boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
	final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();
    boolean drawingWithRenderNode = mAttachInfo != null
            && mAttachInfo.mHardwareAccelerated
            && hardwareAcceleratedCanvas;
    final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
	...
	Bitmap cache = null;
	if (layerType == LAYER_TYPE_SOFTWARE || !drawingWithRenderNode) {
		cache = getDrawingCache(true);
	}
	...
	if (drawingWithRenderNode) {
    	renderNode = updateDisplayListIfDirty();
    	if (!renderNode.hasDisplayList()) {
        	renderNode = null;
        	drawingWithRenderNode = false;
    	}
	}
	...
    if (!drawingWithDrawingCache) {
        if (drawingWithRenderNode) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            ((RecordingCanvas) canvas).drawRenderNode(renderNode);
        } else {
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRA
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchDraw(canvas);
            } else {
                draw(canvas);
            }
        }
    } else if (cache != null) {
    	...
    	canvas.drawBitmap(cache, ...)
    }
    ...
}
```

`draw(Canvas, ViewGroup, long)`方法并不是我们熟悉的`draw(Canvas)`方法，`draw(Canvas, ViewGroup, long)`内部又调用了`draw(Canvas)`方法。

在`draw(Canvas, ViewGroup, long)`方法内部执行绘制逻辑时，首先会调用updateDisplayListIfDirty更新当前`View`的DisplayList；接着：

* 如果缓存不可用 - 执行`RecordingCanvas.drawRenderNode`；
* 否则 - 使用缓存，将缓存直接绘制到`RecordingCanvas`中；

可以看到，当一个`View`调用了updateDisplayListIfDirty之后，就有可能会执行`RecordingCanvas.drawRenderNode`，将`RenderNode`所包含的DisplayList的各种绘制指令给绘制到`RecordingCanvas`中。

### 层级绘制 - 硬件绘制 - 总结
总结一下，对于`View`树上的每一个节点`View`，在硬件绘制时，上级会首先调用其updateDisplayListIfDirty方法将绘制请求传递给它。当传递请求给到当前的节点`View`时，`View`会根据它自己的layerType来决定执行的逻辑：

* `LAYER_TYPE_SOFTWARE` - 使用缓存在`AttachInfo`的`Canvas`，如果没有就新建一个，将自身及子`View`的内容绘制到`Bitmap`中，再将`Bitmap`记录到硬件绘制专用的`RecordingCanvas`中；
* `LAYER_TYPE_HARDWARE`, `LAYER_TYPE_NONE`  - 使用`RecordingCanvas`记录绘制指令；

而后通过dispatchDraw继续向其子`View`递归，将绘制请求传递给子`View`，直至叶子节点。

最终都会调用`RecordingCanvas.drawRenderNode`，将`View`的内容渲染到渲染缓冲区中，等待渲染到屏幕上；

一图以蔽之，硬件绘制的流程（使用虚线代表向子`View`递归的过程）：
![](android_view_hardware_draw.drawio.png)

## 层级绘制 - 软件绘制
如果`ThreadedRenderer`的isEnable方法返回了false，说明关闭了硬件加速，就会采用软件绘制。软件绘制的入口在`ViewRootImpl`的drawSoftware方法中，它会直接调用`View`的`draw(Canvas)`方法：

```java
// View.java

private boolean drawSoftware(...) {
	...
	canvas = surface.lockCanvas(dirty);
	...
	mView.draw(canvas);
	...
	surface.unlockCanvasAndPost(canvas);
	...
}
```

与硬件绘制不同的是，这里的的`Canvas`实例是由`Surface`的lockCanvas方法提供的：

```java
// Surface.java

private final Canvas mCanvas = new CompatibleCanvas();

public Canvas lockCanvas(Rect inOutDirty) {
	...
	nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
	return mCanvas;
}
```

而`CompatibleCanvas`是`Canvas`的子类，只是对`Canvas`的一个方法进行了重写，不影响各种drawXXX
方法：

```java
// Surface.java

private final class CompatibleCanvas extends Canvas {
	@Override
	public void setMatrix(Matrix matrix) { ... }

	@Override
	public void getMatrix(Matrix m) { ... }
}
```

而在硬件绘制过程中，使用到的`Canvas`是`RecordingCanvas`，`RecordingCanvas`对`Canvas`的各种drawXXX方法进行了重写：

```java
// RecordingCanvas.java
public final class RecordingCanvas extends DisplayListCanvas {
	public void drawCircle(...) { ... }

	public void drawRoundRect(...) { ... }

	public void drawTextureLayer(...) { ... }

	public void drawRenderNode(...) { ... }

	public void drawWebViewFunctor(...) { ... }

	public void drawGLFunctor2(...) { ... }
}

// DisplayListCanvas.java
public abstract class DisplayListCanvas extends BaseRecordingCanvas { ... }

// BaseRecordingCanvas.java
public class BaseRecordingCanvas extends Canvas {

	...

	public final void drawArc(...) { ... }

	public final void drawBitmap(...) { ... }

	public final void drawARGB(...) { ... }

	public final void drawBitmapMesh(...) { ... }

	public final void drawCircle(...) { ... }

	public final void drawColor(...) { ... }

	public final void drawLine(...) { ... }

	public final void drawOval(...) { ... }

	public final void drawPaint(...) { ... }

	public final void drawText(...) { ... }

	...
}
```

可以认为在软件绘制过程中，使用的`Canvas`与硬件绘制的完全不同，说明硬件绘制和软件绘制的实现是不同的。

回到drawSoftware方法，drawSoftware方法调用`View`的`draw(Canvas)`方法执行绘制逻辑。`draw(Canvas)`方法上文硬件绘制时已经分析过，只不过这里传递的`Canvas`是普通的`Canvas`。

一图以蔽之（虚线代表向子`View`递归）：

![](android_view_software_draw.drawio.png)


总体而言，一次`View`树的完整绘制流程如图所示（虚线代表向子`View`递归）：
![](android_view_software_hardware_draw.drawio.png)

# 总结
`View`的绘制分为硬件绘制和软件绘制，对于`View`树上的每一个节点`View`:

* 软件绘制

  使用`Canvas`实例进行绘制，通过`draw(Canvas)`来绘制自身，通过`dispatchDraw(Canvas)`来绘制子`View`；

* 硬件绘制

  调用updateDisplayListIfDirty方法构建DisplayList，构建DisplayList的流程与软件绘制无异，区别在于绘制使用的`Canvas`实例是硬件绘制专用的`RecordingCanvas`。

硬件绘制相比软件绘制好处多多，在API 14及其以上的系统默认开启，但硬件绘制并[不支持]((https://developer.android.com/guide/topics/graphics/hardware-accel#unsupported))所有的绘制指令，如果你不幸使用到了这些绘制指令，你就需要避免使用硬件绘制来执行这些绘制指令。

官方提供了三种粒度来[控制硬件加速](https://developer.android.com/guide/topics/graphics/hardware-accel#controlling)：

* Application
* Activity
* View

我们可以直接在AndroidManifest.xml上直接关闭整个应用或单个Activity的硬件绘制，但为了兼容少许几个硬件绘制不支持的绘制指令而关闭Activity乃至整个应用的硬件加速显然是不明智的。于是官方提供了`LayerType`这个概念来控制`View`使用硬件绘制还是软件绘制。

如果当前处于硬件绘制的流程，指定`LayerType`为`LAYER_TPYE_SOFTWARE`，该`View`就会使用软件绘制的逻辑绘制出`Bitmap`，再把`Bitmap`交给硬件绘制逻辑进行绘制，以避开硬件绘制不支持的绘制指令。

如果当前处于软件绘制的流程，则不区分`LayerType`。

至于`View`是如何解决**画什么**的问题，其实是通过`Canvas`实例的各种drawXXX方法来绘制各种图形，最终通过渲染呈现在屏幕上。














