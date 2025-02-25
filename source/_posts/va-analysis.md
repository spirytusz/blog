---
title: 浅谈虚拟框架VirtualApp原理 & 检测方案
subtitle:
date: 2022-01-04 21:25:16
author:     "spirytusz"
header-img: "cover.jpg"
tags:
- Android
- 插件化
---

# 前言
[VirtualApp](https://github.com/asLody/VirtualApp)是一款运行于Android系统的虚拟框架，允许在其中创建虚拟空间，并在这个虚拟空间中运行其他应用，并对该应用具有完全的控制能力。

本文将配合VirtualApp的源码，简单介绍VirtualApp免安装启动apk的Activity的基本原理，以及相关的检测方案。


# 名词约定
|  名词   |  简称  |  备注  |
|  ----  |  ----  |  ----  |
| VirtualApp | VA | - |
| VActivityManager | VAM | VirtualApp自身的ActivityManger |
| VActivityManagerService | VAMS | VirtualApp自身的ActivityMangerService |
| VPackageManager | VPM | VirtualApp自身的PackageManager |
| VPackageManagerService | VPMS | VirtualApp自身的PackageManagerService |
| ActivityManager | AM | android sdk 的ActivityManger |
| ActivityManagerService | AMS | android sdk 的ActivityMangerService |
| PackageManager | PM | android sdk 的PackageManager |
| PackageManagerService | PMS | android sdk 的PackageManagerService |
| 虚拟应用 | - | 运行在VirtualApp内的应用 |

# 几个问题
在一个虚拟空间内免安装启动apk的Activity，从VA作者的角度来看，他需要做什么，我觉得不过以下几个问题：

1. 如何解析apk包内的四大组件信息？
2. 启动应用时，如何解决代码加载和资源加载的问题？
3. 启动应用后，如何启动四大组件？
4. 启动应用后，如何实现对app的完全Hook能力？

下面开始分析VirtualApp的安装和启动流程，以解答上面的问题。

# 安装
在VA内部安装虚拟应用，最终会调用VPMS的`installPackage`方法，这里撇去兼容性代码，保留关键流程：

```
// VPackageInstallerService
public synchronized InstallResult installPackage(String path, int flags, boolean notify) {
    ...
    File packageFile = new File(path);
    ...
    VPackage pkg = null;
    try {
        // 1. 反射创建android.pm.PackageParser对象，解析出apk包的四大组件以及其他相关信息
        // 保存在VPackage对象中
        pkg = PackageParserEx.parsePackage(packageFile);
    } catch (Throwable e) {
        e.printStackTrace();
    }
    ...
    // 2. 把so库复制到/data/data/io.virtualapp/virtual/$packageName/lib 目录下
    File appDir = VEnvironment.getDataAppPackageDirectory(pkg.packageName);
    File libDir = new File(appDir, "lib");
    NativeLibraryHelperCompat.copyNativeBinaries(new File(path), libDir);
    ...
    // 3. 保存前面通过android.pm.PackageParser解析出来的信息
    PackageParserEx.savePackageCache(pkg);
    PackageCacheManager.put(pkg, ps);
    ...
    // 4. 把apk文件复制到/data/data/io.virtualapp/virtual/$packageName 目录下
    File privatePackageFile = new File(appDir, "base.apk");
    FileUtils.copyFile(packageFile, privatePackageFile);
}
```

![安装流程](安装流程.drawio.png)

可以看到，在VA内部安装虚拟应用，VA主要做了这几件事

1. 反射创建`android.pm.PackageParser`实例，解析虚拟应用apk包的四大组件以及其他信息；
2. 把so库复制到对应包的虚拟路径下；
3. 保存、持久化部分apk包数据到硬盘内；
4. 把apk包复制到对应的虚拟路径下；

在内部安装虚拟应用，核心逻辑全部交给反射创建`android.pm.PackageParser`实例实现，VirtualApp只是做了so文件和apk文件的拷贝，并持久化了信息。

其中需要持久化的信息包括appId、包名、so库路径等关键信息，方便下次启动时，重新使用`android.pm.PackageParser`实例解析内部安装应用信息。

经过VA内部安装的逻辑，我们已经可以拿到虚拟应用内的四大组件信息，进而拿到启动的Intent，开始具备启动能力。接下来看看VirtualApp是如何处理启动的逻辑。

# 启动
VA在启动的时候预埋了一些逻辑。一言以蔽之，VA通过注入实例 + 动态代理 + 四大组件插桩的形式，将虚拟应用运行在自己的进程内。先来看看VA预埋的代码：

## 注入对象
启动虚拟应用的关键之一，就是对`ActivityThread.mH`的`mCallback`字段的实例进行替换:

```
// HCallbackStub.java
public class HCallbackStub implements Handler.Callback, IInjector {
    @Override
    public void inject() throws Throwable {
        otherCallback = getHCallback();
        // 将this，也就是HCallbackStub注入到ActivityThread的mH实例的mCallback字段中
        mirror.android.os.Handler.mCallback.set(getH(), this);
    }
    
    private static Handler getH() {
        // 获取单例ActivityThread的mH实例
        return ActivityThread.mH.get(VirtualCore.mainThread());
    }

    private static Handler.Callback getHCallback() {
        try {
            Handler handler = getH();
            // 获取单例ActivityThread的mH实例中的mCallack实例
            return mirror.android.os.Handler.mCallback.get(handler);
        } catch (Throwable e) {
            e.printStackTrace();
        }
        return null;
    }
    
    @Override
    public boolean handleMessage(Message msg) {
        ...
    }
}
```

可以看到，VA通过反射，注入自己实现的`Handler.Callback`到`ActivityThread.mH.mCallback`中，以达到

1. 拦截消息
2. 处理消息
3. 决定是否转发给mH

的作用。

通过`handleMessage`的返回值，可以决定是否转发给mH，具体原因看源码便可以知道：

```
// Handler.java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
               // 通过callback 的 handleMessage返回值，可以决定是否转发给Handler
                return;
            }
        }
        handleMessage(msg);
    }
}
```

![HCallback_perview.drawio.png](HCallback_perview.drawio.png)

## 动态代理
注入实例可以做到方法拦截，是因为Handler对外提供了Callback接口，允许开发者对其执行流程进行控制。并不通用。动态代理更为通用一些，它能代理接口方法，并返回一个经过代理的实例给你。来看看VA使用动态代理做了什么。以Hook Activity启动为例：

```
// MethodProxies.java
class MethodProxies {
    static class StartActivity extends MethodProxy {

        @Override
        public String getMethodName() {
            return "startActivity";
        }
        
        public Object call(Object who, Method method, Object... args) throws Throwable {
            ...
            int res = VActivityManager.get().startActivity(intent, activityInfo, resultTo, options, resultWho, requestCode, VUserHandle.myUserId());
            ...
            return res;
        }
    }
}
```

这里省略了很多细节，只保留最关键的部分。可以看到，这里是对`startActivity `方法进行拦截，并把这个逻辑转发到VAM中。是对哪个实例的`startActivity `进行拦截? 看看初始化流程:

```
// ActivityManagerStub.java
// runtime级别的注解，运行时会把MethodProxies内所有的类实例化，并加到一个表里面
@Inject(MethodProxies.class)
public class ActivityManagerStub extends MethodInvocationProxy<MethodInvocationStub<IInterface>> {
    @Override
    public void inject() throws Throwable {
        if (BuildCompat.isOreo()) {
            //Android Oreo(8.X)
            // 拿到ActivityManager中的IActivityManagerSingleton对象
            Object singleton = ActivityManagerOreo.IActivityManagerSingleton.get();
            // 将这个对象的mInstance，替换成我们自己的代理对象，即ProxyInterface
            Singleton.mInstance.set(singleton, getInvocationStub().getProxyInterface());
        } else {
            if (ActivityManagerNative.gDefault.type() == IActivityManager.TYPE) {
                // 同理
                ActivityManagerNative.gDefault.set(getInvocationStub().getProxyInterface());
            } else if (ActivityManagerNative.gDefault.type() == Singleton.TYPE) {
                // 同理
                Object gDefault = ActivityManagerNative.gDefault.get();
                Singleton.mInstance.set(gDefault, getInvocationStub().getProxyInterface());
            }
        }
    }
}
```

可以看到，这里实际上是将代理对象注入到`ActivityManager`内的单例`IActivityManagerSingleton`（8.0以下是`gDefault`）的`mInstance`字段中。通过注入代理对象，实现对指定方法：

1. 拦截；
2. 决定是否转发；

的目的。

## 插桩四大组件
虚拟应用的四大组件，必然是没有声明到宿主应用的AndroidManifest中的。这就会带来一个问题，启动一个没有声明在AndroidManifest的组件，是会引起当前进程崩溃的。

对此，VA的解决方法是，在AndroidManifest中声明了一些插桩用的四大组件，统共运行在100个进程内。

```
<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C0"
    android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale"
    android:process=":p0"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme" />

<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C1"
    android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale"
    android:process=":p1"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme" />
    
    ...
    
<provider
    android:name="com.lody.virtual.client.stub.StubContentProvider$C0"
    android:authorities="${applicationId}.virtual_stub_0"
    android:exported="false"
    android:process=":p0" />

<provider
    android:name="com.lody.virtual.client.stub.StubContentProvider$C1"
    android:authorities="${applicationId}.virtual_stub_1"
    android:exported="false"
    android:process=":p1" />
    
    ...
```
Service的启动比较特殊，所以不需要声明桩Service。

Broadcast也比较特殊，如果声明在AndroidManifest中，相当于静态注册了，所以没有声明桩Broadcast。

在启动apk内的一个组件时，先根据其运行的进程新建桩组件，并把需要启动的apk组件信息序列化到桩组件的intent中，发送给AMS，然后经过AMS的操作后，调用桩组件进程的`IApplicationThread`，通过`Handler`切线程，到达`ActivityThread.mH`中，在走到VirtualApp实现埋好的`HCallbackStub`中，在`HCallbackStub`中，从intent中提取出真正需要启动的组件，然后启动即可。


## 获取可启动的Intent
这是调用VA的接口启动已经内部安装的虚拟应用示例代码。

```
public void launchTargetApp(String packageName, int userId) {
    Intent targetIntent = VirtualCore.get().getLaunchIntent(packageName, userId)
    if (targetIntent != null) {
        VirtualCore.get().startActivity(intent)
    }
}
```

先看看VA内部是如何获取具有启动能力的intent。
调用`VirtualCore. getLaunchIntent `，最终会走到`VPMS`的`queryIntentActivities`方法：

```
public List<ResolveInfo> queryIntentActivities(Intent intent, String resolvedType, int flags, int userId) {
    ...
    // 尝试从intent内获取ComponentName
    ComponentName comp = intent.getComponent();
    ...
    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
        // 	1. 在这里会通过VPMS内部维护的包列表，以component.packageName为key
        // 获取对应activityInfo并返回
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            final ResolveInfo ri = new ResolveInfo();
            ri.activityInfo = ai;
            list.add(ri);
        }
        return list;
    }
    ...
    final String pkgName = intent.getPackage();
    if (pkgName == null) {
        // 2. 通过intent-filter，获取category为LAUNCHER的activityInfoList
        return mActivities.queryIntent(intent, resolvedType, flags, userId);
    }
    final VPackage pkg = mPackages.get(pkgName);
    if (pkg != null) {
        // 3. 同样也是通过intent-filter，获取category为LAUNCHER的activityInfoList，只不过增加了包名的过滤条件
        return mActivities.queryIntentForPackage(intent, resolvedType, flags, pkg.activities, userId);
    }
    return Collections.emptyList();
}
```
这里会有三种获取intent的逻辑

1. 指定component
2. 指定intent内部的intent-filter（通过intent.addCategory()指定）
3. 指定包名+指定intent-filter


在`VirtualCore. getLaunchIntent `内部，指定了包名和值为LAUNCHER 的 category，所以这里会走第3种逻辑，根据给定的`VPackage `，过滤掉`category`不是`LAUNCHER`的activity，返回一个只有一个元素的List给调用方。

获取了Intent之后，接下来就是调用VAMS启动activity了。


## 真正的启动逻辑
获取了具有启动能力的Intent后，调用`VirtualCore.startActivity`，最终调用了`VAMS`的`startActivity`方法，把启动任务交给了`ActivityStack`的`startActivityLocked`。

```
// VActivityManagerService.java
int startActivityLocked(int userId, Intent intent, ActivityInfo info, IBinder resultTo, Bundle options,
                        String resultWho, int requestCode) {
    TaskRecord reuseTask = null;
    // 通过启动模式、Intent中所带的flags来确定可以在现有的哪个任务栈启动
    ...
    if (reuseTask == null) {
        // 没有可用的任务栈，就在新的任务栈中启动
        startActivityInNewTaskLocked(userId, intent, info, options);
    } else {
        // 把可用的任务栈移动到前台
        mAM.moveTaskToFront(reuseTask.taskId, 0);
        ...
        // 根据ActivityInfo的processName，分配一个对应进程的桩activity
        // 再将intent内的component替换成桩activity
        // 启动桩activity所对应的进程
        destIntent = startActivityProcess(userId, sourceRecord, intent, info);
        // 在对应进程启动桩activity
        // 最终调到realStartActivityLocked中
        startActivityFromSourceTask(reuseTask, destIntent, info, resultWho, requestCode, options);
    }
    return 0;
}

private Intent startActivityProcess(int userId, ActivityRecord sourceRecord, Intent intent, ActivityInfo info) {
    // 根据activity的进程，分配一个进程
    ProcessRecord targetApp = mService.startProcessIfNeedLocked(info.processName, userId, info.packageName);
    ...
    Intent targetIntent = new Intent();
    // 根据进程的vpid，找到对应的桩activity
    targetIntent.setClassName(VirtualCore.get().getHostPkg(), fetchStubActivity(targetApp.vpid, info));
    ...
    // 把原始的activityInfo保存到桩activity对应的intent中，这里是targetIntent
    StubActivityRecord saveInstance = new StubActivityRecord(intent, info,
            sourceRecord != null ? sourceRecord.component : null, userId);
    saveInstance.saveToIntent(targetIntent);
    return targetIntent;
}

private void realStartActivitiesLocked(IBinder resultTo, Intent[] intents, String[] resolvedTypes, Bundle options) {
    Class<?>[] types = IActivityManager.startActivities.paramList();
    Object[] args = new Object[types.length];
    ...
    // 直接走本地的ActivityManager启动桩activity
    IActivityManager.startActivities.call(ActivityManagerNative.getDefault.call(),
                (Object[]) args);
}
```

这里主要做了几件事：

1. 查询当前所有的任务栈，是否有可供这个activity启动的任务栈；
2. 没有就新建一个，有调AMS的方法把这个栈移到前台；
3. 根据activityInfo的包名和进程名，分配一个虚拟的pid，即为vpid；
4. 根据vpid，获取对应的桩Activity Intent；
5. 把需要启动的activity的信息塞入到这个intent中；
6. 调用AMS启动桩activity


经过AMS的一系列操作，桩activity对应的进程已经启动。此时这个进程做了下面的事情：

1. 进入到ActivityThread的main方法中，调用attach通知AMS我已经启动了；
2. AMS通过IBinder token回调，告诉这个进程需要启动桩activity；
3. 通过IPC回到桩activity进程的IApplicationThread；
4. 通过Handler回调到主线程，进入到预先埋好的HCallbackStub中

此时逻辑走到了VirtualApp预先埋好的代码，来看看HCallbackStub做了什么：

```
// HCallbackStub.java
@Override
public boolean handleMessage(Message msg) {
    if (LAUNCH_ACTIVITY == msg.what) {
        if (!handleLaunchActivity(msg)) {
            return true;
        }
    }
    return false;
}

private boolean handleLaunchActivity(Message msg) {
    Object r = msg.obj;
    Intent stubIntent = ActivityThread.ActivityClientRecord.intent.get(r);
    // 反序列化真正需要启动的activity信息
    StubActivityRecord saveInstance = new StubActivityRecord(stubIntent);
    Intent intent = saveInstance.intent;
    ActivityInfo info = saveInstance.info;
    ...
    if (!VClientImpl.get().isBound()) {
        // apk的application还没有初始化，先初始化
        // 主要是创建一个application实例，修改进程名，以及回调一些生命周期，等等；
        VClientImpl.get().bindApplication(info.packageName, info.processName);
        // 把这个消息插入到消息队列头部
        getH().sendMessageAtFrontOfQueue(Message.obtain(msg));
        // 不让Handler处理
        return false;
    }
    ...
    // 将classloader设置进去
    ClassLoader appClassLoader = VClientImpl.get().getClassLoader(info.applicationInfo);
    intent.setExtrasClassLoader(appClassLoader);
    // 替换intent
    ActivityThread.ActivityClientRecord.intent.set(r, intent);
    // 替换需要启动的Activity
    ActivityThread.ActivityClientRecord.activityInfo.set(r, info);
    return true;
}
```

![HCallbackStub启动流程](./HCallbackStub启动流程.drawio.png)

在这里，HCallbackStub主要做了一下几件事：

1. 如果没有初始化application，初始化它；
2. 反序列化出真正需要启动的activity；
2. 初始化apk中的application，以及执行其他应用级别的逻辑；
3. 替换ActivityClientRecord中的intent和activityInfo


至此，剩下的启动逻辑，都交由android sdk接管，是系统启动activity的流程。

启动Activity中涉及到VA中的总流程：

![启动流程.drawio.png](启动流程.drawio.png)

# 问题解答
回顾一下上文提出的问题

1. 如何解析apk包内的四大组件信息？
2. 启动应用时，如何解决代码加载和资源加载的问题？
3. 启动应用后，如何启动四大组件？
4. 启动应用后，如何实现对app的完全Hook能力？

## 解析apk包
解析apk的四大组件信息，VPMS通过调用android sdk内的`PackageParser`来解析apk内的四大组件信息，然后将包名、apk文件路径，so库文件路径序列化到本地，以供下次启动时重新调用`PackageParser `，恢复四大组件的信息。

## 代码加载
解决代码加载问题，关键是拿到apk包所对应的LoadedApk对象的实例。

LoadedApk是什么？

LoadedApk对象是APK文件在内存中的表示。 Apk文件的相关信息，诸如Apk文件的代码和资源，甚至代码里面的Activity，Service等组件的信息我们都可以通过此对象获取。

在启动四大组件前，VirtualApp会在HCallbackStub内检查apk的application是否有初始化，如果未初始化，则初始化它：

```
// VClientImpl.java
private void bindApplicationNoCheck(String packageName, String processName, ConditionVariable lock) {
    AppBindData data = new AppBindData();
    // 初始化applicationInfo
    data.appInfo = VPackageManager.get().getApplicationInfo(packageName, 0, getUserId(vuid));
    // 初始化进程名
    data.processName = processName;
    ...
    mBoundApplication = data;
    // 获取包的context，这个context的classloader，能加载apk中的类
    Context context = createPackageContext(data.appInfo.packageName);
    ...
    Object boundApp = fixBoundApp(mBoundApplication);
    mBoundApplication.info = ContextImpl.mPackageInfo.get(context);
    // 注入LoadedApk，
    // data.info就是mBoundApplication.info，
    // mBoundApplication.info就是context的mPackageInfo
    mirror.android.app.ActivityThread.AppBindData.info.set(boundApp, data.info);
    ...
    // 初始化apk内的application
    mInitialApplication = LoadedApk.makeApplication.call(data.info, false, null);
    // 注入到ActivityThread中的mInitialApplication字段
    mirror.android.app.ActivityThread.mInitialApplication.set(mainThread, mInitialApplication);
    ...
}

private Context createPackageContext(String packageName) {
    try {
        Context hostContext = VirtualCore.get().getContext();
        // CONTEXT_INCLUDE_CODE 代表包括代码
        // CONTEXT_IGNORE_SECURITY 代表忽略安全警告
        return hostContext.createPackageContext(packageName, Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
    } catch (PackageManager.NameNotFoundException e) {
        e.printStackTrace();
        VirtualRuntime.crash(new RemoteException());
    }
    throw new RuntimeException();
}
```

![bindApplication流程](./bindApplication流程.drawio.png)

VA能够初始化apk包中的application，最关键的就是调用android sdk 的 `createPackageContext `方法。通过这个方法，可以拿到LoadedApk对象，进而初始化application。

四大组件也大同小异，以Activity为例：

```
private boolean handleLaunchActivity(Message msg) {
    ActivityInfo info = saveInstance.info;
    ...
    // 这里把activityInfo给设置进去了
    ActivityThread.ActivityClientRecord.activityInfo.set(r, info);
    // 把这个message转发给mH处理
    return true;
}
```

这里把activityInfo替换之后，转发给mH，mH转发给`performLaunchActivity`处理:

```
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ...
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    java.lang.ClassLoader cl = appContext.getClassLoader();
    // 通过classloader加载对应的activity类，然后反射创建
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
}
```

![activity创建](./activity创建.drawio.png)

同样这里也带上了`CONTEXT_INCLUDE_CODE`来加载`LoadedApk`对象，经过这个逻辑，便可以使用`LoadedApk`加载并初始化appContext，此时appContext的classloader，便有了加载activity类的能力。

## 资源加载
资源加载的问题，LoadedApk也是关键。
在`performLaunchActivity`中，会为activity创建一个context：

```
 /**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ...
    ContextImpl appContext = createBaseContextForActivity(r);
    ...
}

private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
    ContextImpl appContext = ContextImpl.createActivityContext(
        this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
    ...
    return appContext;
}

static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
        Configuration overrideConfiguration) {
    ...
    // Create the base resources for which all configuration contexts for this Activity
    // will be rebased upon.
    context.setResources(resourcesManager.createBaseTokenResources(activityToken,
            packageInfo.getResDir(),
            splitDirs,
            packageInfo.getOverlayDirs(),
            packageInfo.getApplicationInfo().sharedLibraryFiles,
            displayId,
            overrideConfiguration,
            compatInfo,
            classLoader,
            packageInfo.getApplication() == null ? null
                    : packageInfo.getApplication().getResources().getLoaders()));
    context.mDisplay = resourcesManager.getAdjustedDisplay(displayId,
            context.getResources());
    return context;
}
```
资源同样也是依赖于LoadedApk，而LoadedApk已事先创建完毕，资源加载就能正常往下走。

![资源加载](./资源加载.drawio.png)


## Hook能力
因为虚拟应用是运行在VA自己的进程内，所以理论上是有完全Hook能力的。

# 检测
通过阅读源码发现VA有以下特点：

1. 虚拟应用是运行在VA的进程内；
2. appDir路径包含VA的appDir路径
3. 一些关键的对象被替换成了代理；

（VA还有很多特点，这里仅列出一部分。）

根据这三个特点，可以有如下方案，检测当前应用是否运行在VA下：

## 检测关键对象是否被替换
VA通过替换一些关键对象，实现对流程的控制，以AM为例，可以检测AM是否被替换：

```
private fun isAmProxy(): Boolean {
    val clazz = Class.forName("android.app.ActivityManager")
    val field = clazz.getDeclaredField("IActivityManagerSingleton")
    field.isAccessible = true
    val singleton = field.get(null)
    val singletonClazz = Class.forName("android.util.Singleton")
    val get = singletonClazz.getDeclaredMethod("get")
    val am = get.invoke(singleton)
    return am is Proxy
}
```

正常的环境，AM不可能是一个代理实例。通过判断AM是否是Proxy，便可直接判断环境是否正常。


## 检测同一个uid下的所有进程
VA将虚拟应用运行在它自己的进程下。通过这个特点，我们可以对当前同一个uid的进程进行遍历。如果出现了其他包的包名，就可以断定环境不正常：

```
private fun runningBadEnvironment(): Boolean {
    val am = getSystemService(ACTIVITY_SERVICE) as ActivityManager
    val runningProcesses = am.runningAppProcesses
    runningProcesses.forEach {
        // 这里可以加个白名单，防止误伤
        if (!it.processName.contains(packageName)) {
            return true
        }
    }
    return false
}
```

## 检测appDir的所有父路径是否有读写权限
VA将虚拟应用的dataDir目录放到其dataDir的子目录下。我们可以利用这一点来检测。

```
private fun appDirAccessible() {
    var parent = File(dataDir.parent ?: "")
    var accessible = false
    while (parent.absolutePath != File.separator) {
        accessible = accessible or parent.canRead()
        parent = File(parent.parent ?: File.separator)
    }
    return accessible
}
```

这里通过检查appDir目录的所有父目录是否有读权限。如果有读权限，说明环境不正常。

# 总结
通过以上介绍可以看出，VA通过替换系统本地代理，以及关键流程中的实例替换，提供虚拟应用运行时对外交互的能力，使得虚拟应用能够运行到自己的容器中，达到虚拟化的目的。


# 参考文献
1. [VirtualApp](https://github.com/asLody/VirtualApp)
2. [Android 插件化原理解析——插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)
3. [Android 插件化原理解析——Activity生命周期管理](http://weishu.me/2016/03/21/understand-plugin-framework-activity-management/)
4. [Android 应用多开对抗实践](https://bbs.pediy.com/thread-255212.htm)



