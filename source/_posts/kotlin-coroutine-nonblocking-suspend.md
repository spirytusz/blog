---
title: 浅谈Kotlin协程(3)
subtitle: 非阻塞式挂起与恢复
date: 2022-09-04 22:24:41
header-img: "cover.webp"
tags:
	- Kotlin
	- Coroutine
---

# 前言
在学习Kotlin协程的过程中，**非阻塞式挂起**这个概念一直困扰着我。挂起是什么？又是怎么个非阻塞式法？挂起后又如何恢复？这是《深入理解Kotlin协程》的读书笔记，记录我对**非阻塞式挂起**和**恢复**的一些理解。

# 挂起函数
上文[浅谈Kotlin协程(2)——协程的启动和执行](../kotlin-coroutine-launch)中，我们已经知道协程体代码都被编译器编译在了`invokeSuspend`方法中。如果涉及到普通函数的调用，编译出来的代码与源码并无二致。那如果调用的是挂起函数呢？

```kotlin
fun test() {
    GlobalScope.launch {
        suspendableFun()
    }
}

private suspend fun suspendableFun() {
    println("Hello Kotlin Coroutine")
}
```

反编译看看：
```java
public final Object invokeSuspend(@NotNull Object $result) {
	// ①
    Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
    switch(this.label) {
    case 0:
        ResultKt.throwOnFailure($result);
        // ②
        CoroutineSuspendableTest var10000 = CoroutineSuspendableTest.this;
        this.label = 1;
        // ③
        if (var10000.suspendableFun(this) == var2) {
            return var2;
        }
        break;
    case 1:
        ResultKt.throwOnFailure($result);
        break;
    default:
        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }
    // ④
    return Unit.INSTANCE;
}

private final Object suspendableFun(Continuation $completion) {
    String var2 = "Hello Kotlin Coroutine";
    System.out.println(var2);
    return Unit.INSTANCE;
}
```

阅读代码发现了以下几个疑点：

1. 调用流程是①->②->③->④，是同步调用，没有涉及到任何与挂起有关的逻辑；
2. 挂起函数`suspendableFun`的方法签名从`() -> Unit`变成了`(Continuation) -> Any?`；
3. `invokeSuspend`多了`switch-case`结构；

我们一起来分析一下。

## 挂起函数不一定会挂起

事实上`suspendableFun`永远不会被挂起，这点IDE已经有了提示：

![](redundant_suspend.png)

挂起的充要条件是**调用栈改变**。来一个挂起函数一定会被挂起的例子：

```kotlin
suspend fun willSuspendFunc(): String = suspendCoroutine<String> { 
	Thread {
		it.resume("Hello Kotlin Coroutine!")
	}
}
```

这个方法有两个关键：

1. 调用了`suspendCoroutine`方法；
2. 返回值是String，但是并没有看到显式返回String的代码；

跟进`suspendCoroutine`的代码：

```kotlin
suspend inline fun <T> suspendCoroutine(
	crossinline block: (Continuation<T>) -> Unit
): T = suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
    val safe = SafeContinuation(c.intercepted())
    block(safe)
    safe.getOrThrow()
}
```

`suspendCoroutineUninterceptedOrReturn`的作用就是将编译器生成的`Continuation $completion`参数通过lambda参数暴露给给我们，所以这里的`c: Continuation<T>`事实上就是被`DispatchedContinuation`包装了的`BaseContinuationImpl`。我们在`willSuspendFunc`方法拿到的continuation，它的结构如图所示：
![](safe_continuation_wrap.drawio.png)

执行完传入`suspendCoroutine`的lambda表达式后，重点在于`SafeContinuation`的`getOrThrow`方法：

```kotlin
enum class CoroutineSingletons { COROUTINE_SUSPENDED, UNDECIDED, RESUMED }

class SafeContinuation<in T> : Continuation<T> {
    
    private var RESULT = AtomicReference<Any?>(UNDECIDED)

    fun getOrThrow(): Any? {
        var result = this.RESULT // atomic read
        if (result === UNDECIDED) {
            if (RESULT.compareAndSet(this, UNDECIDED, COROUTINE_SUSPENDED)) {
            	return COROUTINE_SUSPENDED
            }
            result = this.result
        }
        ...
    }

    override fun resumeWith(result: Result<T>) {
        val cur = this.RESULT.get()
        when {
            cur === COROUTINE_SUSPENDED -> if (RESULT.compareAndSet(this, COROUTINE_SUSPENDED, RESUMED)) {
                delegate.resumeWith(result)
                return
            }
            ...
        }
    }
}
```

`RESULT`的类型是`Any?`，用于存放当前协程体的状态和挂起函数的执行结果，当存放状态时，会有三种情况：

| 状态 | 含义 | 如何扭转至此状态 |
| --- | --- | --- |
| UNDECIDED | 初始状态 | 创建`SafeContinuation` |
| COROUTINE_SUSPENDED | 挂起状态 | 执行lambda表达式的调用栈下，没有调用`resumeWith` |
| RESUMED | 恢复状态 | 执行`resumeWith` |

由于在执行lambda表达式的调用栈下，并没有执行`resumeWith`方法，所以此时的`RESULT`会从`UNDECIDED`扭转为`COROUTINE_SUSPENDED`。显然这里`getOrThrow`方法的返回值是`COROUTINE_SUSPENDED`，接下来如何执行，源码并没有告诉我们答案，考虑到编译器会帮我们生成一些代码，我们不妨反编译成java代码试试：
```java
private final Object willSuspendFuc(
	Continuation<? super String> paramContinuation
) {
    SafeContinuation safeContinuation = new SafeContinuation(
    	IntrinsicsKt.intercepted(paramContinuation)
    );
    new Thread(
    	new CoroutineSuspendableTest$suspendableFun$2$1(
    		(Continuation<? super String>)safeContinuation
    	)
    );
    // ②
    Object object = safeContinuation.getOrThrow();
    return object;
}

public final Object invokeSuspend(Object param1Object) {
    ...
    // ①
    param1Object = param1Object.willSuspendFuc(continuation);
    // ③
    if (param1Object == IntrinsicsKt.getCOROUTINE_SUSPENDED())
        return IntrinsicsKt.getCOROUTINE_SUSPENDED(); 
    return Unit.INSTANCE;
}
```

①处调用了`willSuspendFuc`方法，并在②返回了`COROUTINE_SUSPEND`，于是乎在③处便也返回了`COROUTINE_SUSPEND`。③处在源码中的表现，相当于代码的执行停在了挂起方法`willSuspendFunc`处。只有在另一个线程起来，resume了Continuation，代码才会继续执行。

当另一个线程起来的时候就会调用`Continuation`的`resumeWith`方法。此时的`Continuation`结构如图所示（与上图一模一样）：
![](safe_continuation_wrap.drawio.png)

其调用链如同剥洋葱一样，从外到内按顺序调用`Continuation`的`resumeWith`方法。先来看看`SafeContinuation`：

```kotlin
class SafeContinuation<in T> : Continuation<T> {
    
    private var RESULT = AtomicReference<Any?>(UNDECIDED)

    override fun resumeWith(result: Result<T>) {
        val cur = this.RESULT.get()
        when {
            cur === COROUTINE_SUSPENDED -> if (RESULT.compareAndSet(this, COROUTINE_SUSPENDED, RESUMED)) {
                delegate.resumeWith(result)
                return
            }
            ...
        }
    }
}
```

由于当前的状态已经是`COROUTINE_SUSPENDED`，所以会直接调用内层`Continuation`的`resumeWith`方法。内层是`DispatchedContinuation`，会帮我们切换到调度器指定的线程中，而后在这个线程内，调用`BaseContinuationImpl`的`resumeWith`方法。

上篇文章[浅谈Kotlin协程(2)——协程的启动和执行](../kotlin-coroutine-launch)中，我们已经知道`BaseContinuationImpl`的`resumeWith`方法会调用其`invokeSuspend`方法：

```java
public final Object invokeSuspend(Object param1Object) {
    switch (this.label) {
    	case 0:
            break;
        case 1:
        	// ②
            ResultKt.throwOnFailure(param1Object);
            System.out.println((String)param1Object);
            return Unit.INSTANCE;
        default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    } 
    ResultKt.throwOnFailure(param1Object);
    param1Object = CoroutineSuspendableTest.this;
    Continuation continuation = (Continuation)this;
    // ①
    this.label = 1;
    param1Object = param1Object.suspendableFun(continuation);
    if (param1Object == IntrinsicsKt.getCOROUTINE_SUSPENDED())
        return IntrinsicsKt.getCOROUTINE_SUSPENDED(); 
    return Unit.INSTANCE;
}
```

因为在挂起前label被置为了1（①处代码），所以协程恢复时会走case 1，打印出字符串。自从挂起函数`withSuspendFunc`执行完毕，走完**执行-挂起-恢复**完整的流程。

完整的**执行-挂起-恢复**如图所示：
![](suspend_func_exec_suspend_resume.drawio.png)

经过分析和代码跟进后，我们可以发现，一个挂起函数的**执行-挂起-恢复**流程事实上是开发者、Kotlin协程标准库以及kotlin编译器三者打配合共同完成的。

- 对于开发者

  从源码的角度，对于开发者只负责①和⑫，对开发者而言这是一个同步过程；

- 对于Kotlin协程标准库

  负责兜住挂起状态，不将挂起状态暴露给开发者，图中的③~⑩；

- 对于kotlin编译器

  负责`invokeSuspend`的生成，即处理挂起状态和协程状态机的状态扭转，图中的②~③、⑦和⑪；

## 小结
挂起函数虽然叫做挂起函数，但是它不一定会被挂起。如上两个例子，两者最根本的区别就是前者是同步调用——在挂起方法内同步返回，后者是异步调用，**调用栈**改变了。因此，挂起函数是否会被挂起，取决于调用栈是否改变。

另外，虽然从源码的角度来看，调用挂起函数如同同步调用一般，但经过编译器的处理后，本质上还是异步回调。异步转同步的桥梁即为`kotlin-coroutine-core`标准库的逻辑，它帮我们兜住挂起的状态，通过`Continuation`保存了挂起前的现场，在协程恢复时能够恢复现场，从而继续执行。这也解释了为何挂起函数在经过编译后，方法签名改变了：

- 增加`Continuation`参数是为了能和协程体关联起来，以便在挂起恢复后能够再次调用`invokeSuspend`方法；
- 返回值变成`Any?`是为了支持返回挂起状态，以便能够挂起。

# 协程状态机
我们可以在协程体内同步调用多个挂起方法。这就涉及到的多个挂起函数状态维护的问题，kotlin协程的设计者设计出了一个协程状态机，巧妙的解决了这个问题。


阅读经过编译器生成的`invokeSuspend`方法：

```java
private int label = 0

@Nullable
public Object invokeSuspend(Object param) {
	switch (label) {
		case 0: {
			label = 1;
			Object ret = suspendFunc1(this);
			if (ret == COROUTINE_SUSPENDED) {
				return COROUTINE_SUSPENDED;
			}
			param = ret;
		}
		case 1: {
			System.out.println((String)param);

			label = 2;
			Object ret = suspendFunc2(this);
			if (ret == COROUTINE_SUSPENDED) {
				return COROUTINE_SUSPENDED;
			}
			param = ret;
		}
		case 2: {
			System.out.println((String)param);

			label = 3;
			Object ret = suspendFunc3(this);
			if (ret == COROUTINE_SUSPENDED) {
				return COROUTINE_SUSPENDED;
			}
			param = ret;
		}
		case 3: {
			System.out.println((String)param);

			label = 4;
			Object ret = suspendFunc4(this);
			if (ret == COROUTINE_SUSPENDED) {
				return COROUTINE_SUSPENDED;
			}
			param = ret;
		}
		case 4: {
			System.out.println((String)param);
			return Unit.INSTANCE;
		}
		default: throw IllegalStateException("")
	}
}
```

上面代码执行的顺序如图：
![](suspend_func_sequence_exec.drawio.png)

1. 每一次调用`invokeSuspend`都会扭转一次状态，即label++；
2. 对于每个状态（case），都会调用挂起函数；
3. 如果挂起函数返回了`COROUTINE_SUSPENDED`，`invokeSuspend`则会直接返回，当前的调用栈结束，让出当前线程，协程体的执行流程停留于此，直至恢复时再次调用`invokeSuspend`；
4. 如果协程体的某一个挂起函数直接返回结果，则会接着执行下一个case，然后回到3处，直至协程体结束。

简而言之，Kotlin协程的设计者把每个挂起函数都拆分成了一个状态：

如果这个挂起函数没有被挂起，则利用switch-case的特性，执行下一个挂起函数；

如果这个挂起函数被挂起了，就让出当前线程，在恢复的时候，再次调用`invokeSuspend`，就自然而然的走到了下一个状态，去执行下一个挂起函数。

Kotlin协程状态机，本质就是switch-case + label。

# 非阻塞式挂起与恢复
经过上文分析，如何非阻塞式挂起与恢复的答案便呼之欲出了。

所谓挂起，本质就是`invokeSuspend`返回`COROUTINE_SUSPENDED`，效果上就是调用栈结束，协程体的执行停留在了挂起点，让出当前线程；因为当前线程空闲了，没有停在那里一直等协程体恢复，所以挂起才是非阻塞式的。

既然挂起时让出了当前线程，必然需要有一个对象能够保存挂起点的现场（当前的调用栈走到哪里），以便后续恢复的时候能够继续执行协程体。而这个对象就是`Continuation`：

```kotlin
public interface Continuation<in T> {

    public val context: CoroutineContext

    public fun resumeWith(result: Result<T>)
}
```

协程恢复的发起方调用`resumeWith`后，开始剥洋葱式的层层执行`Continuation`，最终走到了`BaseContinuationImpl`的`resumeWith`方法，而`BaseContinuationImpl`的`resumeWith`又会调用`invokeSuspend`方法，继续执行挂起点之后的代码，直至下一个挂起点或协程体结束。

那么问题就是这个`Continuation`从哪来了。这个`Continuation`来源于挂起函数的参数列表，挂起函数参数列表的`Continuation`就是`BaseContinuationImpl`，或者是被包装过的`BaseContinuationImpl`。

千言万言，一个协程体的启动和结束，可以简化成下面的代码，：
```kotlin
GlobalScope.launch(object: BaseContinuationImpl() {

	private var label = 0

	init {
		invokeSuspend(Unit)
	}

	internal override fun invokeSuspend(param: Any): Any {
		if (label == 0) {
			label = 1
			val ret = suspendFunc1(this)
			if (ret == 挂起) {
				return 挂起
			}
			param = ret
		}

		if (label == 1) {
			println(param)
			label = 2
			val ret = suspendFunc2(this)
			if (ret == 挂起) {
				return 挂起
			}
			param = ret
		}

		println(param)
		return Unit
	}

	override fun resumeWith(result: Result<Any>) {
		invokeSuspend(result)
	}
})
```

# 协程的本质
分析到这里，我对Kotlin协程的本质有了一个大致认识。

本质上来说，Kotlin协程给我们构造了这么一个世界：在这个世界（协程体）中，普通函数都会被同步调用，挂起函数同样也会被同步调用。然而，与普通函数不同的是，挂起函数会被编译器插入一个`Continuation`，它代表一个回调，允许挂起函数切换调用栈后在另一个调用栈结束时，通过回调再给我们**切回来**，以达到继续执行协程体的目的。

所谓调用栈切换，既可以是基于事件循环，如Android平台的`Handler.post`，以及Swing平台的`invokeLater`，也可以是基于线程池的`execute`或`submit`。切到其他调用栈后，当前调用栈结束，及时让出当前线程，才能够更加充分的榨取CPU资源。

# 总结
所谓挂起，本质就是调用栈的切换。调用栈切换后，当前调用栈结束，线程转为空闲状态，而不是忙等协程恢复，所以才谓之为非阻塞式。

恢复的本质即为回调，通过编译期插入一个回调`Continuation`给挂起函数，使得挂起函数才切换调用栈后，在另个调用栈调用该回调，从而回到挂起点，以继续执行代码。











