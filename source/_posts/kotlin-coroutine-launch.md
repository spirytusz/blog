---
title: 浅谈Kotlin协程(2)
subtitle: 协程的启动和执行
date: 2022-08-20 20:54:40
header-img: "cover.webp"
tags:
	- Kotlin
    - Coroutine
---

# 前言
理解Kotlin协程的启动和执行是理解Kotlin协程各种概念的基石。

本文是《深入理解Kotlin协程》的读书笔记，记录一些我对Kotlin协程启动的源码阅读流程。

# 从入口函数说起
`kotlinx-coroutine-core`提供了一系列方法来启动协程，其中就有`launch`方法：

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    ...
}
```

上篇文章[浅谈Kotlin协程(1)——Kotlin协程上下文是什么](../kotlin-coroutine-context)中我们已经理解了`CoroutineScope`与`CoroutineContext`，这篇文章就不做赘述。剩下两个要素：

- `CoroutineStart`——启动模式，是个枚举，默认是`CoroutineStart.DEFAULT`；

  | 启动模式 | 含义 |
  |---|---|
  | DEFAULT | **立即调度**，协程体执行前如果被取消，将会立即进入取消状态 |
  | ATOMIC | **立即调度**，与`DEFAULT`不同的是，如果协程体执行前被取消，只会在协程体的第一个挂起点后进入取消状态 |
  | LAZY | 主动调用start方法才会开始调度 |
  | UNDISPATCHED | **立即执行**协程体，直至第一个挂起点 |

  `CoroutineStart`的作用和实现咱们暂且先按下不表。

- `suspend CoroutineScope.() -> Unit`——协程体。是`BaseContinuationImpl`的子类型，下文会提及。

我们以默认的启动模式为例，并指定`CoroutineContext`为`EmptyCoroutineContext`:

```kotlin
val coroutineScope: CoroutineScope = ...

coroutineScope.launch {
    println("Hello Kotlin Coroutine!")
}

fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    // ①
    val newContext = newCoroutineContext(context)
    // ②
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    // ③
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

## 包装CoroutineContext
①处中对传入的协程上下文进行了包装（简化了无关代码）：

```kotlin
fun CoroutineScope.newCoroutineContext(
    context: CoroutineContext
): CoroutineContext {
    ...
    return if (context !== Dispatchers.Default && context[ContinuationInterceptor] == null) {
        context + Dispatchers.Default
    } else {
        context
    }
}
```

可以看出，newCoroutineContext的作用是：如果上下文中没有调度器，就添加默认的调度器，否则原样返回。

## 创建协程
②处会根据协程的启动模式中创建一个协程。因为我们用的是默认的协程启动模式，所以会创建一个类型为`StandaloneCoroutine`的协程：

```kotlin
class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(
    parentContext, 
    initParentJob = true, 
    active = active
) { ... }

abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
    ...

    /**
     * Starts this coroutine with the given code [block] and [start] strategy.
     * This function shall be invoked at most once on this coroutine.
     * 
     * * [DEFAULT] uses [startCoroutineCancellable].
     * * [ATOMIC] uses [startCoroutine].
     * * [UNDISPATCHED] uses [startCoroutineUndispatched].
     * * [LAZY] does nothing.
     */
    public fun <R> start(
        start: CoroutineStart, 
        receiver: R, 
        block: suspend R.() -> T
    ) {
        start(block, receiver, this)
    }
}
```

出现的类中，他们互相之间的关系为：
![](abstract_coroutine_uml.drawio.png)

## 启动协程
协程创建完毕，接下来在③处启动协程。由于启动模式是`DEFAULT`，根据`AbstractCoroutine`的注释，会调用`startCoroutineCancellable`方法。阅读了`CoroutineStart`的源码，发现它重写的`invoke`操作符：

```kotlin
enum class CoroutineStart {
    public operator fun <R, T> invoke(
        block: suspend R.() -> T, 
        receiver: R, 
        completion: Continuation<T>
    ): Unit = when (this) {
        DEFAULT -> block.startCoroutineCancellable(receiver, completion)
        ATOMIC -> block.startCoroutine(receiver, completion)
        UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
        LAZY -> Unit // will start lazily
    }
}
```

继续跟进`startCoroutineCancellable`方法：

```kotlin
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R, completion: Continuation<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
) {
    createCoroutineUnintercepted(receiver, completion)
      .intercepted()
      .resumeCancellableWith(Result.success(Unit), onCancellation)
}
```

首先，`startCoroutineCancellable`方法的receiver是协程体本身，就是我们调用`launch`是传入的lambda表达式。

其次，它有三个参数

- receiver: R——即为协程作用域`CoroutineScope`；
- continuation: Continuation\<T\>——即为实现了`AbstractCoroutine`的`StandaloneCoroutine`实例；
- onCancellation: ((cause: Throwable) -> Unit)? 暂且按下不表


再次，在`startCoroutineCancellable`中会调用三个方法，我们逐个分析。

### createCoroutineUnintercepted

首先我们需要知道一个背景知识：只要是被suspend关键词所修饰的lambda表达式，其类型均为`BaseContinuationImpl`：

```kotlin
val suspendLambda = suspend { }
fun Class<*>.isSubClassOf(that: Class<*>): Boolean {
    return if (this == Any::class.java) {
        false
    } else {
        this == that || this.superclass.isSubClassOf(that)
    }
}
val baseContinuationImpl =
    Class.forName("kotlin.coroutines.jvm.internal.BaseContinuationImpl")

// Output: true
println(suspendLambda::class.java.isSubClassOf(baseContinuationImpl))
```

再来看看`createCoroutineUnintercepted`方法：

```kotlin
fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    return if (this is BaseContinuationImpl)
        create(receiver, completion)
    else {
        ...
    }
}
```

因为`this`是被suspend修饰的lambda，那么这里必定会走if分支。并且，因为kotlin编译器为我们生成了synthetic方法，create方法事实上已经被我们传入的协程体所重写。编译完毕后，反编译成java代码再查看：

```java
public final void test() {
    BuildersKt.launch$default(
        (CoroutineScope)GlobalScope.INSTANCE, 
        null, 
        null, 
        new SuspendLambdaAnonymousClass(null)
    );
}

static final class SuspendLambdaAnonymousClass extends SuspendLambda 
    implements Function2<...> {
    ...
}
```

协程体本质还是匿名内部类，经过编译后编译器会生成一个内部类`SuspendLambdaAnonymousClass`，它是`SuspendLambda`的子类，所以`SuspendLambdaAnonymousClass`本身也是一个`Continuation`。多个类的关系如下：
![](continuation_impl_uml.drawio.png)

其中，`<<anonymous suspend lambda>>` 就是我们传入的lambda，即协程体。

我们再来看看生成的create方法：

```java
/**
 * @param param1Object receiver: CoroutineScope
 * @param param1Continuation completion: Continuation<T>
 */
public Continuation<Unit> create(
    Object param1Object, 
    Continuation<?> param1Continuation
) {
    return (Continuation<Unit>) new SuspendLambdaAnonymousClass(
        (Continuation)param1Continuation
    );
}
```

其本质还是将传入的`Continuation`（`StandaloneCoroutine`）做一次包装，包装成`BaseContinuationImpl`类型。前后者的关系如下图所示：

![](base_wrap_abstract_coroutine.drawio.png)

### intercepted
接下来会调用`intercepted`方法，从方法的名字就可以看出这个是拦截协程用的。

```kotlin
actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
```

跟进`ContinuationImpl`的代码后发现确实如此，它会将传递过来的`Continuation`再度包装成`DispatchedContinuation`：

```kotlin
internal abstract class ContinuationImpl(
    ...
): BaseContinuationImpl(...) {
    fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
}
```

因为此时协程上下文中唯一的拦截器是`Dispatchers.Default`，也就是协程调度器，协程调度器在拦截协程的时候又把传入的`Continuation`再包装一遍：

```kotlin
public abstract class CoroutineDispatcher : ContinuationInterceptor {
    override fun <T> interceptContinuation(
        continuation: Continuation<T>
    ): Continuation<T> = DispatchedContinuation(this, continuation)
}
```

所以此时的`Continuation`相当于：

![](dispatched_wrap_base_wrap_abstract_coroutine.drawio.png)

### resumeCancellableWith
最终来到了`resumeCancellableWith`方法，查看它的源码：

```kotlin
public fun <T> Continuation<T>.resumeCancellableWith(
    result: Result<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Unit = when (this) {
    is DispatchedContinuation -> resumeCancellableWith(result, onCancellation)
    else -> resumeWith(result)
}
```

继续跟进`DispatchedContinuation`的`resumeCancellableWith`方法：

```kotlin
internal class DispatchedContinuation<in T>(
    val dispatcher: CoroutineDispatcher,
    val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), Continuation<T> by continuation {
    inline fun resumeCancellableWith(
        result: Result<T>,
        noinline onCancellation: ((cause: Throwable) -> Unit)?
    ) {
        if (dispatcher.isDispatchNeeded(context)) {
            dispatcher.dispatch(context, this)
        } else {
            resumeUndispatchedWith(result)
        }
    }

    inline fun resumeUndispatchedWith(result: Result<T>) {
        continuation.resumeWith(result)
    }
}
```

可以看到，`DispatchedContinuation`会首先让调度器去判断是否需要调度，所谓的调度即为切线程环境。

- 如果需要调度，就让调度器去切线程，并把需要执行的代码——协程体一并传进去，以便新的线程能够执行；
- 如果不需要调度，则马上执行协程体。

> 事实上调度器是对执行者的一个封装，所谓的执行者就是真正执行协程体的线程。
> 
> - 例如基于事件循环的主线程调度器`Dispatchers.Main`；
> - 又如基于线程池的IO调度器、Default调度器；
> 
> 不论何种调度器，最终执行的代码都会被`Runnable`所封装，`DispatchedContinuation`正是实现`Runnable`并重写了run方法。所以在调度的时候，第二个`Runnable`参数才能传入`DispatchedContinuation`本身。而这个最终被执行的代码，就是协程体本身。

不管是否会被调度，最终都会执行`DispatchedContinuation.continuation`的`resumeWith`方法。这个continuation是什么？结合上文的分析，我们知道Kotlin协程使用套娃的方式，一个`Continuation`套另一个`Continuation`。所以，最终各个`Continuation`的调用关系如下，向下调用，向上回溯：

![](wrap_continuation_call_and_back.drawio.png)

这里的continuation就是`BaseContinuationImpl`，跟进它的resumeWith方法（精简了代码）：

```kotlin
abstract class BaseContinuationImpl {
    override fun resumeWith(result: Result<Any?>) {
        var param = result
        ...
        val outcome: Result<Any?> = try {
            val outcome = invokeSuspend(param)
            if (outcome === COROUTINE_SUSPENDED) return
            Result.success(outcome)
        } catch (exception: Throwable) {
            Result.failure(exception)
        }
        ...
        completion.resumeWith(outcome)
    }
}
```

读完简化后的代码后，可以发现关键点就在于`invokeSuspend`方法。invokeSuspend方法由我们传入的suspend lambda表达式提供，是通过编译器生成的synthetic方法：

```java
static final class SuspendLambdaAnonymousClass extends SuspendLambda {
    public final Object invokeSuspend(Object param1Object) {
      ...
      ResultKt.throwOnFailure(param1Object);
      System.out.println("Hello Kotlin Coroutine!");
      return Unit.INSTANCE;
    }
}
```

此时便真正的执行到了协程体，打印出了字符串。

然后调用栈就来到了`AbstractCoroutine`。`AbstractCoroutine`会处理状态扭转以及协调父子协程，保证当前协程的所有子协程完成后，才会去结束当前协程。这里不过多赘述。

此后`Continuation`调用栈层层回溯，直至结束，协程体执行完成，协程扭转为完成的状态。

# 总结
协程的启动并不复杂，只是编译器生成了一些synthetic方法，使得调用栈较难追踪，不过我们可以从反编译出来的代码中寻找到蛛丝马迹，理解Kotlin协程的启动和执行流程。

理解协程的启动与执行的关键之一在于：被suspend关键词修饰的lambda，都是`BaseContinuationImpl`的子类型。`BaseContinuationImpl`有一个重要的方法，就是`resumeWith`方法，是从`Continuation`实现而来的。

但`resumeWith`只管恢复，还没有一个引信来触发协程体的执行。这又引出了`BaseContinuationImpl`的第二个重要方法：`invokeSuspend`。

`invokeSuspend`是abstract的，必须要被重写。从结果来看，我们只是传入了一个suspend lambda参数，并没有重写这个方法；协程体的直接父类`SuspendLambda`（实现了`BaseContinuationImpl`）也没有重写这个方法。我们就能很自然的想到这个方法是编译器偷偷帮我们生成的。

编译器生成的依据就是我们写在协程体内的代码。本文的例子是打印一个字符串，所以编译器也会为我们生成打印字符串的代码。

那如果遇上suspend方法，要怎么生成呢？或者更抽象的，编译器是如何根据协程体内的代码来生成`invokeSupspend`方法的呢？这就涉及到了协程的**非阻塞式挂起**问题。

这篇文章简单的跟踪了Kotlin协程的启动和执行流程，为我接下来理解协程的**非阻塞式挂起**以及启动模式做了铺垫。
