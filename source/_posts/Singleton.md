---
title: 既然要单例，那就单例到底
date: 2018-09-22 22:36:02
header-img: "Singleton_cover.jpg"
author:     "spirytusz"
tags: 
	- 设计模式
---

## 1. 前言
最近忙着实习和秋招，挤不出时间来写博客。正好现在中秋放假，有时间了，就把我之前想了好久的都写出来吧。这次的主题是有关单例的。

<!--more-->

## 2. 从最简单的单例说起
单例，本质就是内存中只有一个实例（严格来说不是这样的，我们只需保证主内存中只有一个实例）。基于此，单例是好实现的：
{% codeblock lang:Java %}
public class Singleton {

    private static Singleton INSTANCE;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}
{% endcodeblock %}

这段代码不长，需要注意的地方也不多。首先，构造方法是private的，这保证其他类不能随意new出一个对象；其次，本类的实例INSTANCE及其getter是static的，保证内存中只有一个实例的同时，也可以通过类名来获取本类的实例。
这个单例的实现很朴素，在单线程的环境下工作是值得信赖的。

## 3. 多线程下的单例出现的问题
单线程的环境下，上面的单例是能够准确的工作的。那么在多线程环境下，上面的单例能够准确的工作吗？不如实验一下，实验代码：
{% codeblock lang:Java %}
public class TestClass {

    private static final int TEST_CASE_COUNT = 10;

    private void startToGetInstance() {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println(Singleton.getInstance().hashCode());
            }
        };
        for (int i = 0; i < TEST_CASE_COUNT; i++) {
            new Thread(runnable).start();
        }
    }

    public static void main(String[] args) {
        new TestClass().startToGetInstance();
    }
}
{% endcodeblock %}

输出是：
{% asset_img under_multi_threads_result.png under multi threads %}

上面的结果表明，存在线程中创建出来的Singleton对象与其他线程的创建的hashcode不一致的问题，即不满足单例。
问题出在哪里？因为Singleton的getter方法不是原子操作，或者更进一步来说，new出一个对象不是一个原子操作。new出一个对象的一个粗略过程是（顺序可能不对，但这不是重点）：
* 分配内存空间
* 引用指向分配的内存空间
* 执行构造函数
这表明，虽然语言层面上创建一个对象的new操作仅仅只是一个语句，但是“水面”之下却有多个复杂的过程在支撑，new并不是一个原子操作！因此，上面的单例在多线程环境下不再可靠。

## 4. 多线程单例的初步解决方案
简单的单例不可靠，是因为new不是一个原子操作，那不简单，一个解决方案就出来了，同步整个getter：
{% codeblock lang:Java %}
public synchronized static Singleton getInstance() {
    if (INSTANCE == null) {
        INSTANCE = new Singleton();
    }
    return INSTANCE;
}
{% endcodeblock %}

但是，方法级的同步无疑会降低整体性能，我们需要尽可能多的异步，尽可能少的同步。况且，在实际中，getInstance中的操作可能不止有创建实例的行为（不推荐这么做，但是有时不得已而为之），为了实现单例，这个代价可能有点大。所以应该尽可能减少同步语句块的长度（我们称之为减少同步的粒度）。因此，现在就有两种貌似可行的方案出来了：
方案一：
{% codeblock lang:Java %}
public static Singleton getInstance() {
    if (INSTANCE == null) {
        synchronized (Singleton.class) {
            INSTANCE = new Singleton();
        }
    }
    return INSTANCE;
}
{% endcodeblock %}

方案二：
{% codeblock lang:Java %}
public static Singleton getInstance() {
    synchronized (Singleton.class) {
        if (INSTANCE == null) {
            INSTANCE = new Singleton();
        }
    }
    return INSTANCE;
}
{% endcodeblock %}
这两个貌似看似可行，但是在实验环境或者实际环境中它们都出现了不同的问题。

* 对于方案一：
确实，这段代码减少了同步的粒度，同步的部分只是new出一个对象。但是，当第一次创建实例的时候，至少有2个以上线程同时调用Singleton的getter方法，则存在以下情况：
**线程a、线程b都已经在同步代码块的门口，即if分支语句内，synchronized语句块外**
线程a、线程b其中一个被调度了，另一个阻塞了，就令被调度的线程为线程a吧：线程a执行完毕后，线程b就被唤醒继而执行同步语句块，实例被创建两次，自然地两次的实例不是同一个实例，不满足单例的条件。因此，方案一不可行。

* 对于方案二：
这个方案我发现好像没有人提，在我学单例模式的时候自己想到的一个，想了想发现有问题，就顺便提一下。这段代码确实能够保证单例，但问题出在其他地方：我们获取Singleton的实例，一般都是通过：
**Singleton.getInstance()**
来获取。然而，每次获取实例，同步代码块都一定会执行一次，每次获取实例都会请求锁，释放锁，开销是十分大的。因此这段代码的根本问题就是为了单例而付出的代价比较大。所以能不用就不用（事实上有更好的解决方案）。

上面两种方案要么不可行要么不好，因此出现了更好的方案——双检锁方案。

## 5. 双检锁方案
双检锁（Double Check Locking, DCL)，即两检一锁，检是if，锁是synchronized：
{% codeblock lang:Java %}
public static Singleton getInstance() {
    if (INSTANCE == null) {
        synchronized (Singleton.class) {
            if (INSTANCE == null) {
                INSTANCE = new Singleton();
            }
        }
    }
    return INSTANCE;
}
{% endcodeblock %}
个人认为双检锁是上面提到的方案一的改进。既然任意线程进入同步代码块以后，INSTANCE可能不为空，因此就用if再检查一次：若为空则new一个，否则什么也不做。
在相当长的一段时间内，我认为双检锁或许就是单例的封笔了。但是事实果真如此吗？

## 6. 一些必要的知识准备
双检锁方案确实有一点问题，在说明这些问题前需要这些这些知识：
* 原子性、可见性以及有序性
* 指令重排序

如果已经对这些知识有所了解，可以跳过这一小段。

### 6.1. 原子性、可见性以及有序性
个人认为，在多线程环境下，只要保证原子性，以及足够的可见性和有序性，就能够保证线程安全。下面对这三个性质分别解释。
#### 6.1.1. 原子性
如果一个操作满足原子性，这表明这个操作是不可以被打断的：要么不执行，要么就一次性执行完毕。这个好理解，我们常常接触的synchronized关键词能够保证代码块、方法执行的原子性：所有线程对同步代码块和同步方法的访问是互斥的，即被synchronized所修饰的代码块或方法的执行时不可以被打断的。
#### 6.1.2. 可见性
所谓可见性，就是在多线程环境下，一个线程的操作对其他所有的线程是可见的。何为可见？即线程A修改了变量i的值，在其他线程被唤醒时，他们看（读取）到的i是线程A修改后的值。但是这些字我都认识，但是拼起来怎么就不懂了？或者觉得定义可见性是多此一举的操作？实则不然，其实只要清楚Java内存模型(Java Memory Model, JMM)，就能明白定义可见性的必要性：
{% asset_img working_memory_and_main_memory.png Java内存模型 %}
这是多线程环境下线程的工作模型。在多线程环境下，线程们并不是如我所想的那样直接从内存中取数据的，而是从他们的工作存储中取数据。而工作存储中的数据来源，是从主内存中拷贝来的。因此，JMM描述的工作过程是这样的：某个线程启动后从主内存中拷贝共享数据到它的私有空间，然后对这个私有空间读写一顿操作后，再写回主内存。所谓的“对所有线程可见”，就是其他线程在读取主内存前，被某一个线程修改的变量已经被写回主内存了，从而读取的值是已被修改的。
#### 6.1.3. 有序性
有序性，就是CPU能够按照程序员编写的代码顺序来执行。但是如果发生指令重排序，有序性就不能完全保证。


### 6.2. 指令重排序
指令重排序是指编译器或硬件不按照原有代码的顺序执行，而是对其重新排序后再执行，以达到提高效率的目的。这里只讨论编译器级别的重排序，硬件级别的重排序不讨论。
编译器对代码进行重新排序是为了提高执行效率，比如下面的代码：
{% codeblock lang:Java %}
int a;
boolean b;
a = 1;
b = false;
{% endcodeblock %}
重排序后可能是：
{% codeblock lang:Java %}
int a = 1;
boolean b = false;
{% endcodeblock %}
或者是其他不影响结果的形式。
何为“不影响结果”？就是在单线程环境下，不进行重排序与进行重排序执行完毕后，所有的变量值是一致的。
譬如，对于这段代码：
{% codeblock lang:Java %}
int a;
a = 2333; // ①
a = a + 1; // ②
int b = a * a; // ③
{% endcodeblock %}
语句③的结果依赖于语句②，语句②的结果依赖于语句①，即依赖关系：语句① <-- 语句② <-- 语句③。
如果对这段代码进行重排序，其结果必然与不进行重排序的不一致。因此，编译器不会对这段代码进行重排序。

## 双检锁方案存在的问题及其解决方法









