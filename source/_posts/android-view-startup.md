---
title: Android View体系 - 启动篇
date: 2022-03-12 16:33:29
header-img: "cover.webp"
tags:
- Android
---

# 前言
点击Launcher的应用图标，到应用的主页展示到屏幕的过程中，Android系统是如何将画面渲染出来的？

本文将分析应用启动时View相关的代码，理解启动期间View的相关代码，理解`View`、`Window`、`WindowManager`、`ViewRootImpl`和`Activity`之间的关系。

# 启动
我们都知道Android应用进程的入口是`ActivityThread`的main方法。在经过与`system_server`的一系列IPC交互后，最终AMS会发起`EXECUTE_TRANSACTION`的命令，启动category为`android.intent.category.LAUNCHER`的Activity，走到了`ActivityThread.performLaunchActivity`方法:

```java
// ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    activity.attach(...);
    ...
    mInstrumentation.callActivityOnCreate(...);
    ...
}
```

从源码可以看到，在应用启动时，会调用到Activity的attach方法，之后马上就会回调Activity的onCreate方法，看看attach内部做了什么：

```java
// Activity.java
final void attach(...) {
    ...
    mWindow = new PhoneWindow(...);
    ...
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        ...
    );
    ...
}
```

可以看到，调用到Activity的attach方法时，会为Activity创建一个`Window `，并为这个`Window `设置好`WindowManager `。

设置的`WindowManager `是哪个类的实例？点进去看看：

```java
// Window.java
public void setWindowManager(WindowManager wm, ...) {
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
 
// WindowManagerImpl.java
private WindowManagerImpl(Context context, Window parentWindow) {
    mContext = context;
    mParentWindow = parentWindow;
 }

public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```

原来`WindownManager `是通过`WindowManagerImpl`的`createLocalWindowManager`创建而来的，而`WindowManagerImpl`的parentWindow，就是上面Activity在attach时创建的Window，即`PhoneWIndow`。


此后，performLaunchActivity结束，Activity的状态置为`ON_CREATE`。

AMS又再发起`EXECUTE_TRANSACTION`的命令，回调Activity的onStart方法，此时Activity开始可见:

```java
// ActivityThread.java
public void handleStartActivity(...) {
    ...
    updateVisibility(r, true /* show */);
    ...
}
```
至此，Activity开始可见，准备开始获取焦点。

# 获取焦点
AMS发起`EXECUTE_TRANSACTION `的命令，试图将Activity的生命周期扭转为`RESUMED `状态，于是代码就走到了`ActivityThread`的`handleResumeActivity`方法：

```java
// ActivityThread.java
public void handleResumeActivity(...) {
    ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        final Activity a = r.activity;
        ...
        View decor = r.window.getDecorView();
        ...
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        ...
        if (!a.mWindowAdded) {
            wm.addView(decor, l);
        }
        ...
    }
    ...
}
```

在`handleResumeActivity`中，从启动的activity中获取`DecorView`和`WindowManager`之后，会将`DecorView `纳入`WindowManager`的管理。看看addView内部做了什么：

```java
// WindowManagerImpl.java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    mGlobal.addView(view, params, mParentWindow, ...);
}
```

mGlobal是`WindowManagerGlobal `，如其命名，是一个进程下的全局WindowManager，维护`View`和`Window`的关系。
看看`WindowManagerGlobal `的addView内部做了什么：

```java
// WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params, ...) {
    ...
    ViewRootImpl root = new ViewRootImpl(view.getContext(), display);
    ...
    mViews.add(view);
    mRoots.add(root);
    ...
    root.setView(view, wparams, ...);
}
```
`WindowManagerGlobal`内部创建了一个`ViewRootImpl`实例，并把view，也就是`DecorView`给设置到`ViewRootImpl`里面去。setView十分重要，内部请求重新布局，并且Window开始获取焦点：

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, ...) {
    ...
    // Schedule the first layout -before- adding to the window
    // manager, to make sure we do the relayout before receiving
    // any other events from the system.
    requestLayout();
    ...
    // 获取焦点，最终回调DecorView的onWindowFocusChanged
    mWindowSession.addToDisplayAsUser(...);
    ...
}
```
经过`WindowSession`的addToDispalyAsUser方法后，Window获取到了焦点，回调到`DecorView `的onWindowFocusChanged。在获取焦点前，有一个关键的requestLayout方法，内部`Choregrapher`请求VSync信号后，就开始的View的三大流程：测量、布局和绘制。先看看VSync信号的请求流程。

# VSync信号
requestLayout十分重要，内部先开启Handler的同步屏障，而后`Choregrapher`开始等待vsync信号，信号到达，回调`TraversalRunnable`，开始执行View树的遍历：

```java
// ViewRootImpl.java

public void requestLayout() {
    ...
    scheduleTraversals();
    ...
}

void scheduleTraversals() {
    ...
    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    ...
}

class TraversalRunnable implements Runnable {
    public void run() {
        doTraversal();
    }
}
```

关键在postCallback，`Choregrapher`会在合适的时机回调`TrasversalRunnable`，会在什么时机？点进去看看：

```java
// Choregrapher.java
private void postCallbackDelayedInternal(int callbackType, ...) {
    ...
    mCallbackQueues[callbackType].addCallbackLocked(...);
    ...
    if (dueTime <= now) {
        scheduleFrameLocked(now);
    }
    ...
}

private void scheduleFrameLocked(long now) {
    ...
    scheduleVsyncLocked();
    ...
}

private final FrameDisplayEventReceiver mDisplayEventReceiver;

private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}

private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable { ... }
```

从代码可以看出，postCallback内部调用到了scheduleFrameLocked，请求VSync信号。`mDisplayEventReceiver`内部调用了native方法，在此不做赘述。

VSync信号到达，会回调到`FrameDisplayEventReceiver`的onVsync方法：

```java
// Choregrapher.FrameDisplayEventReceiver.java
public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
    ...
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}

public void run() {
    ...
    doFrame(...);
}
```

VSync信号到达，`FrameDisplayEventReceiver`会发送一个带有callback的异步消息到Handler中，而后就会回调到`FrameDisplayEventReceiver`的run方法，执行`Choregrapher`的doFrame方法：

```java
// Choregrapher.java
void doFrame(...) {
    ...
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    ...
}

void doCallback(int callbackType, long frameTimeNanos) {
    ...
    callbacks = mCallbackQueues[callbackType] ...;
    ...
    for (CallbackRecord c = callbacks; c != null; c = c.next) {
         ...
         c.run(frameTimeNanos);
         ...
    }
}
```

在doFrame内部调用到doCallbacks，doCallbacks内部`Choregrapher`会找到`CALLBACK_TRAVERSAL`类型的callback，并回调他的run方法，在`CallbackRecord `内部会调用到真正的callback，也就是前面传入的`mTraversalRunnable `。

至此，开始调用`ViewRootImpl `的doTraversal方法，进一步调用performTraversal方法，正式进入View的三大流程: 测量、布局以及绘制：

```java
// ViewRootImpl.java
void doTraversal() {
    ...
    mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
    ...
    performTraversals();
    ...
}

private void performTraversals() {
    ...
    performMeasure();
    ...
    performLayout();
    ...
    performDraw();
    ...
}
```

# 总结

从上面的分析可知，一个`Activity`对应一个`Window`，这个`Window`就是`PhoneWindow`，而其管理类便是`WindowManager`，对应`WindowManagerImpl`。这个`WindowManagerImpl`内部持有一个`WindowManagerGlobal`，统一管理当前应用进程的所有`Window `。

在`Activity`的Resume阶段，通过`WindowManagerGlobal`的addView将`DecorView`和`ViewRootImpl`绑定在一起，形成一个View树，最终调用requestLayout，启动View（`DecorView`）的三大流程：测量、布局以及绘制。

在启动阶段的整体流程：

1. 在`Activity`回调onCreate前，`ActivityThread`会为`Activity`创建一个`Window`，并为这个`Window`设置一个`WindowManager`;
2. 在`Activity`回调onStart后，`ActivityThread`会把这个`Activity`设置为可见；
3. 在`Activity`回调onResume后，会通过`WindowManagerGlobal`创建一个`ViewRootImpl`，此后通过`ViewRootImpl`请求第一次布局，在通过`Choregrapher`请求第一次VSync信号；之后收到VSync信号后，执行performTraversals，正式进入View的三大流程: 测量、布局以及绘制。


![android-view-startup](android-view-startup.svg)






