---
title: 浅谈synchronized的实现原理
date: 2018-07-13 20:15:12
header-img: "about_synchronized_cover.jpeg"
tags: 
    - Java
    - 多线程
---

## 前言
Synchronized是Java中的重量级锁，在我刚学Java多线程编程时，我只知道它的实现和monitor有关，但是synchronized和monitor的关系，以及monitor的本质究竟是什么，我并没有尝试理解，而是选择简单的略过。在最近的一段时间，由于实际的需要，我又把这个问题翻出来，Google了很多资料，整个实现的过程总算是弄懂了，为了以防遗忘，便整理成了这篇博客。
在本篇博客中，我将以class文件为突破口，试图解释Synchronized的实现原理。
<!--more-->

## 从java代码的反汇编说起
很容易的想到，可以从*程序的行为*来了解synchronized的实现原理。但是在源代码层面，似乎看不出synchronized的实现原理。锁与不锁的区别，似乎仅仅只是有没有被synchronized修饰。不如把目光放到更加底层的汇编上，看看能不能找到突破口。*javap*是官方提供的\*.class文件分解器，它能帮助我们获取\*.class文件的汇编代码。具体用法可参考[这里](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/javap.html)。 接下来我会使用javap命令对*.class文件进行反汇编。
编写文件Test.java:
{% codeblock lang:Java Test.java%}
public class Test {

    private int i = 0;

    public void addI_1(){
        synchronized (this){
            i++;
        }
    }

    public synchronized  void addI_2(){
        i++;
    }
}
{% endcodeblock %}
生成class文件，并获取对Test.class反汇编的结果:
{% codeblock%}
javac Test.java
javap -v Test.class
{% endcodeblock %}

{% codeblock %}
Classfile /Users/zhangkunwei/Desktop/Test.class
  Last modified Jul 13， 2018; size 453 bytes
  MD5 checksum ada74ec8231c64230d6ae133fee5dd16
  Compiled from "Test.java"
  ... ...
  public void addI_1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3， locals=3， args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_0
         5: dup
         6: getfield      #2                  // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field i:I
        14: aload_1
        15: monitorexit
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return
  ... ...
    public synchronized void addI_2();
    descriptor: ()V
    flags: ACC_PUBLIC， ACC_SYNCHRONIZED
    Code:
      stack=3， locals=1， args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
   ... ...
{% endcodeblock %}
通过反汇编结果，我们可以看到：
* 进入被synchronized修饰的语句块时会执行**monitorenter**，离开时会执行**monitorexit**。
* 相较于被synchronized修饰的语句块，被synchronized修饰的方法中没有指令**monitorenter**和**monitorexit**，且flags中多了ACC_SYNCHRONIZED标志。
**monitorenter**和**monitorexit**指令是做什么的？同步语句块和同步方法的实现原理有何不同？遇事不决查文档，看看官方文档的解释。

### monitorenter
{% blockquote Java Virtual Machine Specification - https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter  monitorenter %}
**Description**
The _objectref_ must be of type *reference*.

Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with _objectref_， as follows:

* If the entry count of the monitor associated with _objectref_ is zero， the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.

* If the thread already owns the monitor associated with _objectref_， it reenters the monitor， incrementing its entry count.

* If another thread already owns the monitor associated with _objectref_， the thread blocks until the monitor's entry count is zero， then tries again to gain ownership.

**Notes**
* A monitorenter instruction may be used with one or more monitorexit instructions ([§monitorexit](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorexit)) to implement a synchronized statement in the Java programming language ([§3.14](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html#jvms-3.14)). The monitorenter and monitorexit instructions are not used in the implementation of synchronized methods， although they can be used to provide equivalent locking semantics. Monitor entry on invocation of a synchronized method， and monitor exit on its return， are handled implicitly by the Java Virtual Machine's method invocation and return instructions， as if monitorenter and monitorexit were used.
{% endblockquote %}
简单翻译一下:
指令**monitorenter**的操作的必须是一个对象的引用，且其类型为引用。每一个对象都会有一个**monitor**与之关联，当且仅当**monitor**被(其他(线程)对象)持有时，**monitor**会被锁上。其执行细节是，当一个线程尝试持有某个对象的**monitor**时：
- 如果该对象的**monitor**中的**entry count**==0，则将**entry count**置1，并令该线程为**monitor**的持有者。
- 如果该线程已经是该对象的**monitor**的持有者，那么重新进入**monitor**，并使得**entry count**自增一次。
- 如果其他线程已经持有该对象的**monitor**，则该线程将会被阻塞，直到**monitor**中的**entry count**==0，然后重新尝试持有。
注意:
**monitorenter**必须与一个以上**monitorexit**配合使用来实现Java中的同步语句块。而同步方法却不是这样的:同步方法不使用**monitorenter**和**monitorexit**来实现。当同步方法被调用时，**Monitor**介入；当同步方法return时，**Monitor**退出。这两个操作，都是被**JVM**隐式的handle的，就好像这两个指令被执行了一样。

### monitorexit
{% blockquote Java Virtual Machine Specification -  https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorexit  monitorexit %}
**Description**
- The _objectref_ must be of type *reference*.

- The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by _objectref_.

- The thread decrements the entry count of the monitor associated with _objectref_. If as a result the value of the entry count is zero， the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.
{% endblockquote %}
简单翻译一下:
指令**monitorenter**的操作的必须是一个对象的引用，且其类型为引用。并且：
- 执行**monitorexit**的线程必须是**monitor**的持有者。
- 执行**monitorexit**的线程让**monitor**的**entry count**自减一次。如果最后**entry count**==0，这个线程就不再是**monitor**的持有者，意味着其他被阻塞线程都能够尝试持有**monitor**

根据以上信息，上面的疑问得到了解释：
1.**monitorenter**和**monitorexit**是做什么的？
**monitorenter**能“锁住”对象。当一个线程获取**monitor**的锁时，其他请求访问共享内存空间的线程无法取得访问权而被阻塞；**monitorexit**能“解锁”对象，唤醒因没有取得共享内存空间访问权而被阻塞的线程。

2.为什么一个**monitorenter**与多个**monitorexit**对应，是一对多，而不是一一对应？
一对多的原因，是为了保证：执行**monitorenter**指令，后面一定会有一个**monitorexit**指令被执行。上面的例子中，程序正常执行，在离开同步语句块时执行第一个**monitorexit**；Runtime期间程序抛出Exception或Error，而后执行第二个**monitorexit**以离开同步语句块。

3.为什么同步语句块和同步方法的反汇编代码略有不同？
同步语句块是使用**monitorenter**和**monitorexit**实现的；而同步方法是**JVM**隐式处理的，效果与**monitorenter**和**monitorexit**一样。并且，同步方法的flags也不一样，多了一个ACC_SYNCHRONIZED标志，这个标志是告诉**JVM**：这个方法是一个同步方法，可以参考[这里](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.6-200-A.1)。


## Monitor
在上一个部分，我们容易得出一个结论：synchronized的实现和**monitor**有关。**monitor**又是什么呢？从文档的描述可以看出，**monitor**类似于操作系统中的**互斥量**这个概念：不同对象对共享内存空间的访问是互斥的。在**JVM**（**Hotspot**）中，**monitor**是由**ObjectMonitor**实现，其主要的数据结构如下:
{% codeblock lang:C ObjectMonitor https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/runtime/objectMonitor.hpp#L140 ObjectMonitor.hpp %}

ObjectMonitor() {
    _header       = NULL;
    _count        = 0;
    _waiters      = 0，
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;   //指向当前monitor的持有者 
    _WaitSet      = NULL;   //持有monitor后，调用的wait()的线程集合
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;  //尝试持有monitor失败后被阻塞的线程集合
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
}
{% endcodeblock %}

可以看出，我们可以
* 通过修改\_owner来指明**monitor**锁的拥有者；
* 通过读取\_EntryList来获取因获取锁失败而被阻塞的线程集合；
* 通过读取\_WaitSet来获取在获得锁后主动放弃锁的线程集合。

到这里，synchronized的实现原理已经基本理清楚了，但是还有一个未解决的疑问：线程是怎么知道**monitor**的地址的？线程只有知道它的地址，才能够访问它，然后才能与以上的分析联系上。答案是**monitor**的地址在Java对象头中。

## Java对象头
在Java中，每一个对象的组成成分中都有一个Java对象头。通过对象头，我们可以获取对象的相关信息。
这是Java对象头的数据结构(32位虚拟机下):
{% asset_img Java对象头数据结构.png Java对象头数据结构 %}
其中的Mark Word，它是一个可变的数据结构，即它的数据结构是依情况而定的。下面是在对应的锁状态下，Mark Word的数据结构(32位虚拟机下)：
{% asset_img Mark_Word数据结构.png Mark Word数据结构%}
synchronized是一个重量级锁，所以对应图中的重量级锁状态。其中有一个字段是：指向重量级锁的指针，共占用25+4+1=30bit，它的内容就是这个对象的引用所关联的**monitor**的地址。
线程可以通过Java对象头中的Mark Word字段，来获取**monitor**的地址，以便获得锁。

## 回到最初的问题
synchronized的实现原理是什么？从上面的分析来看，答案已经显而易见了。当多个线程一起访问共享内存空间时，这些线程可以通过synchronized锁住*对象*的对象头中，根据Mark Word字段来访问该对象所关联的**monitor**，并尝试获取。当一个线程成功获取**monitor**后，其他与之竞争**monitor**持有权的线程将会被阻塞，并进入EntryList。当该线程操作完毕后，释放锁，因争用**monitor**失败而被阻塞的线程就会被唤醒，然后重复以上步骤。

## 写在最后
我发现其实大部分答案都可以从文档中得到，所以以后遇到问题还是要尝试从文档中找到答案。
本人水平有限，如果本文有错误，还望指正，谢谢~
