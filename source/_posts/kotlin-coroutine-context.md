---
title: 浅谈Kotlin协程(1)
subtitle: Kotlin协程上下文是什么
date: 2022-08-15 20:54:40
header-img: "cover.webp"
tags:
	- Kotlin
    - Coroutine
---

# 前言
我所理解的上下文是一种具有承上启下作用的对象，例如：对外获取资源，对内管理、分配资源。那么协程上下文的职责又是什么呢？本文是《深入理解Kotlin协程》的读书笔记之一，是我对Kotlin协程上下文的一些理解。

# CoroutineContext是什么
协程上下文定义在`kotlinx-coroutine-core`中：

```kotlin
public interface CoroutineContext {

    public operator fun <E : Element> get(key: Key<E>): E?

    public fun <R> fold(initial: R, operation: (R, Element) -> R): R

    public operator fun plus(context: CoroutineContext): CoroutineContext = ...

    public fun minusKey(key: Key<*>): CoroutineContext

    ...
}
```

从`CoroutineContext`暴露的接口来看:

- `CoroutineContext`重写了`[]`和`+`这两个操作符;
- 支持根据给定的`Key`减去某个`CoroutineContext`;
- 支持fold操作，将当前的`CoroutineContext`‘折叠’成类型为R的结果;

这么看来似乎`CoroutineContext`更像一个容器类型，并且还引入了`Key`和`Element`这两个未知的概念。我们不妨继续看`CoroutineContext`还定义了什么：

```kotlin
public interface CoroutineContext {

    ...

    public interface Key<E : Element>

    public interface Element : CoroutineContext {

        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            if (this.key == key) this as E else null

        public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)

        public override fun minusKey(key: Key<*>): CoroutineContext =
            if (this.key == key) EmptyCoroutineContext else this
    }
}
```

从上面的代码可以看出：

- 一个`Key`对应一个`Element`；
- `Element`又实现了`CoroutineContext`；

结合上面的信息，我对`CoroutineContext`就有了一个大致的印象：它是一个以`Key`为主键，以`Element`为值的集合。
但是有一点比较奇怪，哪有对象即是元素本身，也是一个集合的，这不是矛盾了吗？阅读了相关代码后，发现这并不矛盾，分情况讨论：

## CoroutineContext是一个元素
如果`CoroutineContext`是集合的一个元素，那这个`CoroutineContext`必然实现了`CoroutineContext.Element`接口。

例如协程调度器`CoroutineDispatcher`、协程拦截器`ContinuationInterceptor`和协程名`CoroutineName`等；以协程名为例：

```kotlin
public data class CoroutineName(
    val name: String
) : AbstractCoroutineContextElement(CoroutineName) {

    public companion object Key : CoroutineContext.Key<CoroutineName>
}

public abstract class AbstractCoroutineContextElement(
	public override val key: Key<*>
) : Element {

	/* 使用默认实现
	 *
	public override operator fun <E : Element> get(key: Key<E>): E? =
        if (this.key == key) this as E else null

    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    public override fun minusKey(key: Key<*>): CoroutineContext =
        if (this.key == key) EmptyCoroutineContext else this
	 *
	 */

}
```

除了key以外，其他均使用了`Element`的默认实现。这很符合常识，毕竟此时的`CoroutineContext`是一个元素，只需要比较key，以决定进一步的操作。

## CoroutineContext也是一个集合
如果`CoroutineContext`是一个集合，那它的实现必定是`CombinedContext`：

```kotlin
internal class CombinedContext(
    private val left: CoroutineContext,
    private val element: Element
): CoroutineContext {
	...
}
```

从定义来看，`CoroutineContext`类似于集合的递归定义，大致的结构如图所示：

![](combined_context.drawio.svg)

既然是集合，必定重新实现了`CoroutineContext`关于集合修改的方法：

### get方法
既然是一个集合，那很自然的就想到了线性查找。查看实现，确实如此，从右至左查找：
```kotlin
internal class CombinedContext(...) {
	override fun <E : Element> get(key: Key<E>): E? {
    	var cur = this
    	while (true) {
        	cur.element[key]?.let { return it }
        	val next = cur.left
        	if (next is CombinedContext) {
            	cur = next
        	} else {
            	return next[key]
        	}
    	}
	}
	...
}
```
![](combined_context_get.drawio.svg)

### minusKey方法
minusKey方法，顾名思义就是根据给定的`Key`删除集合中的对应元素。看代码会首先判断最右边的元素，如果key相同，则保留左支；否则在左支线性搜索。处理好做减法后集合不变和集合变空这两种边界情况，剩下的就是一般的情况——返回一个新的`CombinedContext`。

```kotlin
internal class CombinedContext(...) {
	public override fun minusKey(key: Key<*>): CoroutineContext {
        element[key]?.let { return left }
        val newLeft = left.minusKey(key)
        return when {
            newLeft === left -> this
            newLeft === EmptyCoroutineContext -> element
            else -> CombinedContext(newLeft, element)
        }
    }
}
```
![](CombinedContext_minusKey.drawio.svg)

### plus方法
`CombinedContext`并没有重写plus方法，而是使用了`CoroutineContext`的默认实现。因为要保证拦截器处于上下文的最右边以方便快速取到拦截器，所以默认实现稍显复杂。

虽然plus方法较为复杂，但其本质还是重写了`+`操作符。`+`操作符的左右边都是`CorouinteContext`，可以是`CoroutineContext.Element`和`CombinedContext`。因为加法是从左至右的，所以分为三种情况（使用`[]`代表`CombinedContext`，最右边即为element字段）：

#### 存在空集合EmptyCoroutineContext
![](CombinedContext_plus_desc.drawio.svg)

#### Element + Element
![](CombinedContext_plus_desc_2.drawio.svg)

因为一个上下文中拦截器有且仅有一个，所以默认实现是后来的覆盖前面的（情况③），且拦截器必定在上下文的最右边。

#### CombinedContext + Element
![](CombinedContext_plus_desc_3.drawio.svg)

拦截器的覆盖如情况⑤，结果也必定是拦截器在最右边。

## CoroutineContext结构既是List也是Map
`CoroutineContext`既像List一样具有线性接口，也能像Map一样根据key来获取对应的value。为什么这么设计？

- 如果`CoroutineContext`只是一个List，我们需要获取对应的类型的上下文时，需要通过index线性查找，最后再通过泛型进行强转；
  ```kotlin
  val elements: List<Element>

  inline fun <reified E: Element> get(): E? {
      return elements.find { it is E } as? E
  }
  ```
- 如果`CoroutineContext`只是一个Map，我们需要在获取对应类型的上下文后，手动再进行强转；
  ```kotlin
  val elementMap: Map<Key<out Element>, Element>

  fun get(key: Key<out Element>): Element? {
  	  return elementMap[key]
  }

  // 这么调用
  val coroutineDispatcher = 
      coroutineContext[CoroutineDispatcher] as? CoroutineDispatcher
  ```

但如果结合List和Map的特点，上面两个问题便迎刃而解。


## 小结
`CoroutineContext`既可以是集合的元素，也可以是集合本身，其表现形式分别是`CoroutineContext.Element`和`CombinedContext`。类图如下：

![](CoroutineContext_UML.drawio.svg)

同时，为了方便搜索，`CoroutineContext`被设计成了Indexed Sets的数据结构——既有List的线性结构，也有Map的键值对结构。

# CoroutineContext的元素
阅读完上文，我们知道原来`CoroutineContext`是一个线性键值对集合结构，那这个集合必定有多种元素。主要的元素与`CoroutineContext`的关系如下：
![](CoroutineContext_Elements.drawio.svg)

## ContinuationInterceptor
`CoroutineContext`的元素有协程拦截器：`ContinuationInterceptor`。顾名思义，协程拦截器是是用于拦截协程的。如果你对协程有大致的了解，就会知道Kotlin协程本质还是回调，其表现形式就是`Continuation`:

```kotlin
public interface Continuation<in T> {

    public val context: CoroutineContext

    public fun resumeWith(result: Result<T>)
}
```
这里的resumeWith就是成功接口和失败接口的合体，当协程从挂起转向恢复状态的时候会调用这个方法。而协程拦截器的本质就是Wrapper模式：将旧有的`Continuation`包装成新的`Continuation`:

```kotlin
public interface ContinuationInterceptor : CoroutineContext.Element {

    public fun <T> interceptContinuation(
    	continuation: Continuation<T>
    ): Continuation<T>

    ...
}
```

这种Wrapper模式可以帮我们做很多事，比如在协程恢复的时候切换线程：

```kotlin
class ContinuationInterceptorImpl<T>(
	private val delegate: Continuation<T>
) : ContinuationInterceptor {

	override fun <T> interceptContinuation(
    	continuation: Continuation<T>
    ): Continuation<T> {
    	return object: Continuation<T> {
    		override val context: CoroutineContext = delegate.context

    		override fun resumeWith(result: Result<T>) {
    			thread { 
    				delegate.resumeWith(result)
    			}
    		}
    	}
    }

}
```

Kotlin的协程已经帮我们实现好了——协程调度器`CoroutineDispatcher`

## CoroutineDispatcher
`CoroutineDispatcher`是协程调度器，它会在协程恢复时决定要不要切换线程和切换到哪个线程执行。其原理如上所描述，在协程恢复时决定是否帮我们切线程和切哪个线程：

```kotlin
public abstract class CoroutineDispatcher : ..., ContinuationInterceptor {

    public final override fun <T> interceptContinuation(
        continuation: Continuation<T>
    ): Continuation<T> = DispatchedContinuation(this, continuation)

    public abstract fun dispatch(
        context: CoroutineContext, 
        block: Runnable
    )

}

internal class DispatchedContinuation<in T>(
    val dispatcher: CoroutineDispatcher,
    val continuation: Continuation<T>
): ..., Continuation<T> {

    override fun resumeWith(result: Result<T>) {
        val context = continuation.context
        if (dispatcher.isDispatchNeeded(context)) {
            dispatcher.dispatch(context, this)
        } else {
            continuation.resumeWith(result)
        }
    }

}
```

可以看到`CoroutineDispatcher`实现了协程拦截器，在拦截协程时，`interceptContinuation`返回了一个`DispatchedContinuation`，本质还是Wrapper模式：

在恢复的时候，检查是否需要切线程，即`CoroutineDispatcher.isDispatchNeeded`，如果需要就调用`CoroutineDispatcher.dispatch`切换，否则就直接恢复协程。

Kotlin协程已经内置好了多个协程调度器，供开发者使用：

```kotlin
public actual object Dispatchers {
    
    @JvmStatic
    public actual val Default: CoroutineDispatcher = DefaultScheduler

    @JvmStatic
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

    @JvmStatic
    public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

    @JvmStatic
    public val IO: CoroutineDispatcher = DefaultIoScheduler
}
```

| 调度器 | 含义 |
|---|---|
|Dispatchers.Main| 主线程调度器，协程恢复时切换到主线程 | 
|Dispatchers.Default| 默认调度器，协程恢复时切换到默认线程池的线程 | 
|Dispatchers.IO| IO线程调度器，协程恢复时切换到IO线程池的线程 | 
|Dispatchers.Unconfined| 协程恢复发起方在哪个线程，协程恢复时就在哪个线程执行 | 

## Job
`Job`也是协程上下文之一，用于描述协程执行状态、层级关系的一类上下文：

```kotlin
public interface Job : CoroutineContext.Element {

    public val isActive: Boolean

    public val isCompleted: Boolean

    public val isCancelled: Boolean

    public fun start(): Boolean

    public fun cancel(cause: CancellationException? = null)

    public val children: Sequence<Job>
}
```

- start和cancel用于扭转协程的状态；
- isActive，isCompleted，isCancelled用于描述协程的状态；
- children用于描述层级关系，即协程的父子关系；

# CoroutineContext的本质
我所理解的`CoroutineContext`，其本质就是“全局”的公共集合，作为协程执行过程中一些必要元素的存取容器：

- 协程（挂起后）恢复时，用`CoroutineDispatcher`把线程环境切回来；
- 协程完成、取消或异常了，用`Job`来扭转协程的状态，并根据父子协程的关系传递事件或者异常；
- 协程执行的过程中出现异常了，使用`CoroutineExceptionHanlder`来处理；
- 为了方便追踪，用`CoroutineName`给协程起个名字；

而这个“全局”，即为作用域：

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

如果脱离作用域来启动一个协程，相当于这个协程没有了上下文，也就丧失了上面所说的几种能力。

所以标准库中定义的启动协程函数，是以`CoroutineScope`为receiver的：

```kotlin
fun CoroutineScope.launch(...): Job { ... }

fun <T> CoroutineScope.async(...): Deferred<T> { ... }
```

所以，`CoroutineContext`的职责为：在作用域下提供协程执行过程中存取必要元素的能力。

# 总结
`CoroutineContext`是一个Indexed Set的数据结构，既有List的线性结构，也有Map的键值对结构。

每个协程作用域都有一个`CoroutineContext`，它为协程执行的过程中提供一个存取必要元素的容器。

这些元素服务于协程的执行——有负责在挂起后恢复时切换线程环境的`CoroutineDispatcher`，也有描述协程执行状态与层级结构的`Job`，也有负责处理协程执行过程中出现的异常的`CoroutineExceptionHandler`，等等。

其本质就是作用域下公共元素的存储集合。