---
title: 浅谈Kotlin协程(4)
subtitle: 异常处理
date: 2022-08-29 20:53:34
header-img: "cover.webp"
tags:
	- Kotlin
	- Coroutine
---

# 前言
挂起函数提供了切换调用栈后自动切回来的能力。那么

1. 这对挂起函数异常的捕获有何影响？
2. 当协程发生异常时，协程内如何传递异常？
3. 协程间如何传播异常？

本文是我对《深入理解Kotlin协程》的读书笔记，记录我对协程异常处理的理解。

# 协程内的异常捕获
在实践中我们可以使用try-catch语句包裹需要捕获异常的代码，但如果被包裹的代码涉及到调用栈的切换，则捕获不了：

```kotlin
fun crashAtEventLoop() {
	try {
		handler.post { 1 / 0 }
	} catch(t: Throwable) {
		t.printStackTrace()
	}
}

fun crashAtThreadSwitch() {
	try {
		threadPool.execute { 1 / 0 }
	} catch(t: Throwable) {
		t.printStackTrace()
	}
}
```

我们知道挂起函数内部有可能切换调用栈。如果挂起函数内部切换了调用栈，并且在另一个调用栈发生了异常，会如何表现？

```kotlin
fun testCrash() {
	GlobalScope.launch {
		try {
			suspendFunc()
		} catch (t: Throwable) {
			Log.e(TAG, "testCrash: $t")
		}
	}
}

private suspend fun suspendFunc() = suspendCoroutine<Int> {
	thread {
		it.resume(1/0)
	}
}
```

调用`testCrash`后必会抛出未被捕获的异常而崩溃，崩溃点在第13行。这是合理的，因为崩溃发生在挂起函数切换调用栈后，切回调用栈前，是协程框架所干预不到的地方。

考虑到`Continuation`的`resumeWith`方法接受的是一个`kotlin.Result`参数，我们不妨使用`kotlin.runCatching`包裹住另一个调用栈，发生异常时返回`Result.Failure`：

```kotlin
private suspend fun suspendFunc() = suspendCoroutine<Int> { cont ->
	thread {
		runCatching {
			cont.resumeWith(1/0)
		}.onFailure {
			cont.resumeWithException(it)
		}
	}
}
```

改造完毕后，再执行一遍，发现异常在`testCrash`方法中被捕获了。因此我们可以得出一个结论，在另一个调用栈的异常需要通过`Continuation.resumeWith`方法传递，才能在恢复时挂起点处抛出并捕获。

# 协程内的异常传递

那么原因是什么呢？通过上篇文章[浅谈Kotlin协程(3)-非阻塞式挂起与恢复](../kotlin-coroutine-nonblocking-suspend)，我们知道`suspendCoroutine`所暴露出来的`Continuation`是这么一个结构：

![](safe_continuation_wrap.drawio.png)

`SafeContinuation`的作用是保证`suspendCoroutine`所暴露的`Continuation`只调用一次；

`DispatchedContinuation`的作用是切线程环境；

那么焦点就来到了`BaseContinuationImpl`：


```kotlin
class BaseContinuationImpl {

	override fun resumeWith(result: Result<Any?>) {
		...
		try {
    		val outcome = invokeSuspend(param)
    		if (outcome === COROUTINE_SUSPENDED) return
    		Result.success(outcome)
		} catch (exception: Throwable) {
		    Result.failure(exception)
		}
		...
		completion.resumeWith(outcome)
		...
	}
}

class CoroutineExceptionCatchTest$1 {

	private var label = 0

	override fun invokeSuspend(result: Result<Any?>): Any? {
		...
		if (label == 1) {
			Result.throwOnFailure(result)
			label = 2
		}
		...
	}
}
```

`BaseContinuationImpl.resumeWith`捕获了`invokeSuspend`中主动抛出的异常，将异常传递给了`StandaloneCoroutine`。

查看`StandaloneCoroutine`的代码，发现它只重写了`handleJobException`方法。从这个方法的名字看来，就是它处理`BaseContinuationImpl`传递过来的异常。跟进这个方法：

```kotlin
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(...) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}

fun handleCoroutineException(
	context: CoroutineContext, 
	exception: Throwable
) {
    try {
        context[CoroutineExceptionHandler]?.let {
            it.handleException(context, exception)
            return
        }
    } catch (t: Throwable) {
        handleCoroutineExceptionImpl(context, handlerException(exception, t))
        return
    }
    handleCoroutineExceptionImpl(context, exception)
}

val handlers: List<CoroutineExceptionHandler> = ServiceLoader.load(
	CoroutineExceptionHandler::class.java,
	CoroutineExceptionHandler::class.java.classLoader
).iterator().asSequence().toList()

fun handleCoroutineExceptionImpl(context: CoroutineContext, exception: Throwable) {
    for (handler in handlers) {
    	handler.handleException(context, exception)
    }

    val currentThread = Thread.currentThread()
    currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
}
```

捕获的异常会按顺序传递给以下处理器处理。如果对应的处理器能够处理，则直接返回，否则就会传递给下一个处理器继续处理：

1. 注册在协程上下文中的`CoroutineExceptionHandler`；
2. 注册在`resources/META-INF/services`目录下，通过服务发现机制获取的`CoroutineExceptionHandler`；
3. `Thread.UncaughtExceptionHandler`

由于此时我们并没有注册`CoroutineExceptionHandler`，所以最终异常会交给`Thread.UncaughtExceptionHandler`处理，其结果就是抛出异常。而后在try-catch语句中再次被捕获，并打印出来。

# 协程的结构
## 父子协程关系构建
要想了解异常如何在协程间传播，我们需要了解协程之间的结构关系。协程是一个抽象的概念，在Kotlin中的具象化表达为`Job`，并且Kotlin协程标准库提供了使用`CoroutineScope`为receiver的启动方法。我们可以这么启动子协程：

```kotlin
val job0 = coroutineScope.launch {
	val job1 = launch {
		delay(3000)
	}
	val job2 = launch {
		delay(2000)
	}
	delay(1000)
}
```

从直觉来看，很自然的就能知道三个job之间的关系，job1、job2是job0的子协程:
![](parent_child_job_tree.drawio.png)

事实上在标准库中的实现也确实是这个结构。父子协程关系的建立被标准库抽象成了`Job.attachChild`方法，在启动一个`Job`（即协程）时会与父`Job`建立父子关系。

以`CoroutineScope.launch`启动的协程为例，内部新建了一个协程`AbstractCoroutine`，并在其构造方法中构建父子协程关系：

```kotlin
class AbstractCoroutine : JobSupport(...) {
	init {
        if (initParentJob) initParentJob(parentContext[Job])
    }
}

open class JobSupport : Job, ChildJob, ParentJob {
	var parentHandle: ChildHandle?

    protected fun initParentJob(parent: Job?) {
        if (parent == null) {
            parentHandle = NonDisposableHandle
            return
        }

        ...
        parentHandle = parent.attachChild(this)
        ...
    }
}
```

`attachChild`的真正实现在`JobSupport`内，它返回的是一个双向链表的节点；而这个双向链表的作用，是维护**父Job**下所有子协程的`Job`取消回调和`Job`完成回调。由于此时是子`Job`调用父`Job`的`attachChild`方法，所以我们把视角切换到父`Job`，跟进代码：

```kotlin
open class JobSupport  : Job, ChildJob, ParentJob {
	override fun attachChild(child: ChildJob): ChildHandle {
		return invokeOnCompletion(
			onCancelling = true, 
			handler = ChildHandleNode(child).asHandler
		) as ChildHandle
	}

	override fun invokeOnCompletion(
        onCancelling: Boolean,
        invokeImmediately: Boolean,
        handler: CompletionHandler
    ): DisposableHandle {
    	...
    }
}
```

由于协程标准库使用无锁算法实现双向链表，所以在代码实现中充斥着大量的自旋操作。这里简化了部分代码，只保留关键部分。

首先，`attachChild`方法其实是将传入的`childJob`包装成为了`ChildHandleNode`后，又去调用了`invokeOnComplete`方法。所谓的`ChildHandleNode`，我认为其实是父`Job`与子`Job`建立双向关系的连接点，具体其实现便能明白：

```kotlin
internal class ChildHandleNode(
    @JvmField val childJob: ChildJob
) : JobCancellingNode(), ChildHandle {
    override val parent: Job get() = job
    override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
}
```

其次，在`invokeOnComplete`方法内会构建一个双向链表的节点，然后根据当前`Job`的状态决定是否插入到双向链表中和返回该节点。

```kotlin
open class JobSupport  : Job, ChildJob, ParentJob {
	override fun invokeOnCompletion(
        onCancelling: Boolean,
        invokeImmediately: Boolean,
        handler: CompletionHandler
    ): DisposableHandle {
    	val node: JobNode = makeNode(handler, onCancelling)
    	loopOnState { state ->
    		when (state) {
    			is Empty -> {
    				_state.compareAndSet(state, node)
    				return node
    			}
    			is Incomplete -> {
    				val list = state.list
    				addLastAtomic(state, list, node)
    				return node
    			}
    			else -> {
    				// is complete
    				return NonDisposableHandle
    			}
    		}
    	}
    }

    private fun addLastAtomic(expect: Any, list: NodeList, node: JobNode) =
        list.addLastIf(node) { this.state === expect }
}
```

- 如果当前`Job`处于`Empty`状态（代表未完成，且没有任何回调，链表为空），就以创建的回调节点为链表头，并返回创建的节点；
- 如果当前的`Job`处于`Incomplete`状态（代表未完成，但有回调，链表不为空），就调用`addLastAtomic`将新建的回调节点插入到维护在状态`state`内的双向链表中，并返回新建的节点；
- 否则，当前`Job`已经完成了，则返回一个`NonDisposableHandle`，代表不会再对任何操作做出响应。

此时`Job`父子关系已经构建完毕，其关键在于`Job`内部维护的状态`_state`。状态内部存储着一个双向链表，每个链表节点代表着父子`Job`沟通的桥梁`ChildHandleNode`，内部同时存储了父`Job`和子`Job`。

光阅读代码略显抽象，不妨以上文的`job0`，`job1`和`job2`为例，使用`GlobalScope.launch`启动`job0`时，由于`job0`并没有父`Job`，所以`Job`构建父子关系的过程相对较为简单，其结果为：

![](job0_initial.drawio.png)

由于`job0`的上下文`CoroutineContext`中没有`Job`，因此其`parentHandle`被初始化为`NonDisposableHandle`，然后直接返回。而`_state`则使用默认值`Empty`，代表没有任何的取消、完成回调。

在`job0`运行的时候，启动了`job1`。由于启动`job1`的协程作用域`CoroutineScope`中存在`Job`（即`job0`），所以会执行父子关系构建的逻辑。`parentHandle`被设置为`ChildHandle`，内部指向了双向链表表头`ChildHandleNode`——维护`job0`与`job1`关系的桥梁。

由于`job1`并没有子`Job`，所以其`parentHandle`被初始化为`NonDisposableHandle`，其`_state`为默认值，即`Empty`。

因此，在`job1`启动后，协程间的关系为：

![](job0_with_job1.drawio.png)

`job1`启动完毕后，`job2`的启动过程也类似，我们可以如法炮制。当涉及到双向链表的插入操作时，`job0`与`job2`的桥梁——`ChildHandleNode`会被插入到表头之后。最终，在`job2`启动后，协程间的关系为：

![](job0_parent.drawio.png)

## 小结
协程在启动时，会去检查传入的协程上下文`CoroutineContext`中是否有`Job`的存在。如果存在`Job`，则认为是父`Job`，进而去初始化父子`Job`的关系；反之则跳过；

在初始化父子`Job`关系时，子`Job`会调用父`Job`的`attachChild`，主动让父`Job`来attach自己。在`attachChild`的过程中，父`Job`会构建一个双向链表，每个节点为`ChildHandleNode`，存储父`Job`自己和对应的子`Job`，使得父`Job`和子`Job`能够双向通信。

最终，父`Job`会将这个双向链表包装成`ChildHandle`，保存在`parentHandle`中。

# 异常传播
了解完协程间的结构后，或许我们就能轻松理解异常在协程间传播的逻辑。在一个协程内，异常的传递类似于剥洋葱的方式，在被包了多层`Continuation`的洋葱结构内进行传播，最终被捕获的异常传递到了`AbstractCoroutine`的`resumeWith`方法中：

```kotlin
class AbstractCoroutine {
	override fun resumeWith(result: Result<T>) {
        makeCompletingOnce(result.toState())
        ...
    }
}
```

异常传播的关键全在`makeCompletingOnce`方法中，也是异常传播、协程取消等庞大逻辑的入口函数。其中的`Result.toState`方法，会将成功和失败包装成对应的类，方便后续对结果成功和结果失败的判断。我们进一步追踪`makeCompletingOnce`方法看看：

```kotlin
class JobSupport {

	internal fun makeCompletingOnce(proposedUpdate: Any?): Any? {
		tryMakeCompleting(state, proposedUpdate)
		...
	}

	private fun tryMakeCompleting(state: Any?, proposedUpdate: Any?): Any? {
		if ((state is Empty || state is JobNode) && state !is ChildHandleNode && proposedUpdate !is CompletedExceptionally) {
			tryFinalizeSimpleState(state, proposedUpdate)
			return proposedUpdate
		}
		return tryMakeCompletingSlowPath(state, proposedUpdate)
	}
}
```

贴出的代码省略了各种中间状态，只保留主流程；其调用流程如下：

![](job_make_complete.drawio.png)

可以看到，在`tryMakeCompleting`方法内，会对当前`Job`的状态进行一次判断，以决定走快路径还是走慢路径。所谓快路径和慢路径，区别在于慢路径需要处理一些中间状态，例如多个异常合并为一个异常、挂起等待子`Job`完成等状态，但结果都是殊途同归，和快路径一样，最终会扭转`Job`自身的状态，并通知子`Job`和父`Job`。

先来看看慢路径，慢路径主要处理了多个异常合并为一个异常、挂起等待子`Job`完成等逻辑。总体流程如下：

![](job_make_completion_slow_path.drawio.png)

这里有几个比较关键的方法，`nofityCancelling`、`tryWaitForChild`和`cancelParent`

## nofityCancelling(list, cause)
如其命名以及参数列表，这个方法最主要的职责是将异常通知给当前`Job`的子`Job`。查看代码：

```kotlin
class JobSupprot {
	private fun notifyCancelling(list: NodeList, cause: Throwable) {
		...
        // 取消子Job
        notifyHandlers<JobCancellingNode>(list, cause)
        // 取消当前Job
        cancelParent(cause)
    }
}
```

## tryWaitForChild

## cancelParent















