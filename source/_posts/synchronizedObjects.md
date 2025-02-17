---
title: Synchronized的正确食用方法
date: 2018-07-18 12:27:13
header-img: "synchronized_objects_cover.jpeg"
author:     "spirytusz"
tags: 
    - Java
    - 多线程
---

## 前言
前面的博客[《浅谈synchronized的实现原理》](http://www.zspirytus.com/2018/07/13/aboutSynchronized/)谈到synchronized的实现原理，简单而言就是锁住了对象。但是似乎还比较抽象，本篇博客将解决：
* synchronized锁住的是哪个对象？
* 如何合理使用synchronized？
那么开始吧。
<!--more-->

## 一、synchronized的两种用法
synchronized有两种基本用法说起，分别是同步语句块和同步方法。

### 1.1.同步语句块
同步语句块是指被synchronized修饰的语句块，被synchronized修饰的语句块，被多个线程执行的过程是互斥的。同步语句块的写法如下：
{% codeblock lang:Java %}
Object obj = new Object();
public void synchronizedStatement(){
    synchronized (obj){
        //do Something
    }
}
{% endcodeblock %}
同步语句块锁住的是括号中的对象，上例中锁住的是obj。这个例子是可以编译通过并且正常运行的，说明synchronized能够锁住*任意对象*。我们可以粗略的把*任意对象*分为*本类的实例对象*、*本类的类对象*以及其他普通对象。

#### 1.1.1.本类的实例对象
demo：
{% codeblock lang:Java %}
public void synchronizedStatement(){
    synchronized (this){
        //do Something
    }
}
{% endcodeblock %}
在这里，synchronized锁住的是*本类的实例对象*，需要注意的是：除非本类是单例，否则本类的实例对象在内存中是可以存在有多个的。

#### 1.1.2.本类的类对象
demo：
{% codeblock lang:Java %}
public void synchronizedStatement(){
    synchronized (SomeClass.class){
        // do Something
    }
}
{% endcodeblock %}
在这里，synchronized锁住的是*本类的类对象*，对象SomeClass.class的类型是Class，本类的类对象在内存中*有且仅有一个*，因此可以把它看做是单例的。

#### 1.1.3.其他普通对象
demo：
{% codeblock lang:Java %}
public void synchronizedStatement(){
    synchronized (obj){
        // do Something
    }
}
{% endcodeblock %}
在这里synchronized锁住的是除了以上提到的任意对象。这个对象的类型可以是HashMap、Integer等API自带类型，也可以是自己编写的类的实例对象、类对象等等。

### 1.2.同步方法
同步方法是指被synchronized修饰的方法，同步方法使得多个线程调用该方法的过程是互斥的。同步方法可以分为静态同步方法和非静态同步方法，锁住的对象是不同的。普通的同步方法锁住的对象是*本类的实例对象*；而静态同步方法锁住的对象是*本类的类对象*。以下是两种同步方法的写法：
{% codeblock lang:Java %}
// Static Synchronized Method
public static synchronized void staticSynchronizedMethod(){
    // do something
}
{% endcodeblock %}

{% codeblock lang:Java %}
// Synchronized Method
public synchronized void synchronizedMethod(){
    // do something
}
{% endcodeblock %}

## 二、同步方法和同步语句块的联系
从上面的解释可以看出，同步方法和同步语句块存在一些联系。事实上，从被锁住的对象的角度来看，synchronized(this)和同步方法是等价的；synchronized(SomeClass.class)和静态同步方法是等价的。
{% codeblock lang:Java %}
//下面两种同步是等价的
public synchronized void synchronizedMethod(){
    // do something
}
    
public void add(){
    synchronized (this){
        // do something
    }
}
{% endcodeblock %}
{% codeblock lang:Java %}
//下面两种同步是等价的
public static synchronized void addByMethod(){
    // do something
}
    
public void add(){
    synchronized (SomeClass.class){
        // do something
    }
}
{% endcodeblock %}

## 三、同步与死锁
同步是互斥的，我们可以简单的认为：我在用，其他人不能用；其他人在用，我不能用。一个经典的例子是：~~哲♂学家~~哲学家进餐问题。哲学家问题描述的是经过一系列同步操作后引发死锁的悲剧。因此同步可能会把我们往危险的陷阱——死锁那里带，在同步时我们必须要考虑死锁的情况。

## 四、考虑死锁的情况
### 4.1.synchronized可能发生死锁吗？
synchronized锁是同步锁，它能够使得共享数据被多个线程操作前后保持数据一致性。既然是共享数据，就有可能会发生死锁。死锁发生的必要条件是：
{% blockquote - https://book.douban.com/subject/26079463/ 《计算机操作系统》 %}
* 互斥条件：进程对所分配到的资源进行排他性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其他进程请求该资源，则请求者只能等待，直至占有该资源的进程用毕释放。
* 请求和保持条件：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源又已被其他进程占有，此时请求进程阻塞，但又对自己已获得的其他资源保持不放。
* 不剥夺条件：进程已获得的资源，在未使用完成之前，不能被剥夺，只能在使用完毕由其释放。
* 环路等待条件：在发生死锁时，必然存在一个进程--资源的环形链，即进程集合P={P<sub>1</sub>,P<sub>2</sub>,... ...,P<sub>n</sub>}中的P<sub>1</sub>等待P<sub>2</sub>的资源，P<sub>2</sub>等待P<sub>3</sub>的资源,... ...,P<sub>n</sub>等待P<sub>1</sub>的资源。
{% endblockquote %}
结合synchronized，我们分别考虑这四个条件：
1.对于互斥条件：synchronized是同步锁，多个线程竞争执行同步代码时，有且仅有一个线程获得锁，因此互斥条件是必然满足的。
2.对于请求和保持条件：因为同步方法和同步语句块原理相同，因此在这里考虑同步语句块。请求和保持，意味着每一个线程至少需要两个不同的锁，即同步语句块中嵌入同步语句块，并且锁的对象是不同的，如下：
{% codeblock lang:Java %}
synchronized(object_1){
    synchronized(object_2){
        // do something synchronized or obtain other monitor lock
    }
}
{% endcodeblock%}
在这个例子中，如果线程获得object_1的对象锁，但是因object_2的对象锁被其他线程持有，因此它会被阻塞并保持持有object_1的对象锁。因此，请求和保持条件是可满足的。
3.不剥夺条件：如请求保持条件，直到线程获取全部需要的对象锁并执行完同步语句块，它是不会释放它获得的锁的。synchronized并没有强制剥夺某一个线程拥有锁的机制，因此，在没有人为的情况下，不剥夺条件必然满足。
4.假设请求和保持条件满足，并结合互斥条件和不剥夺条件。对于环路等待条件满足的情况下，情况就会这这样的，线程们都获得了至少一个锁并且都保持，并且任意一个线程，需要其他线程已经获得的锁（请求保持）。于是大家都和和气气地（不剥夺条件）互相等待。然后就成环了。接下来看一段简单的代码，这段代码实现的是线程t<sub>1</sub>和t<sub>2</sub>共同竞争object_1和object_2的对象锁。
{% codeblock lang:Java %}
import java.util.concurrent.TimeUnit;

public class DeadLockTest {

    private static final String THREAD_1 = "Thread 1";
    private static final String THREAD_2 = "Thread 2";

    private Object object_1 = new Object();
    private Object object_2 = new Object();

    public void methodA() throws InterruptedException, IllegalMonitorStateException {
        Logcat("try to get object_1's lock!");

        synchronized (object_1) {
            Logcat("get object_1's lock!");
            Logcat("Sleeping ... ...");
            TimeUnit.SECONDS.sleep(2);
            Logcat("wake up! try to get object_2's lock!");

            synchronized (object_2) {
                Logcat("get object_2's lock!");
            }
        }
    }

    public void methodB() throws InterruptedException, IllegalMonitorStateException {
        Logcat("try to get object_2's lock!");

        synchronized (object_2) {
            Logcat("get object_2's lock!");
            Logcat("try to get object_1's lock!");
            synchronized (object_1) {
                Logcat("get object_1's lock!");
            }
        }
    }

    private static void Logcat(String message) {
        System.out.println(Thread.currentThread().getName() + ": " + message);
    }

    public static void main(String[] args) {
        final DeadLockTest deadLockTest = new DeadLockTest();
        Thread t1 = new Thread(() -> {
            try {
                deadLockTest.methodA();
            } catch (InterruptedException e) {

            } catch (IllegalMonitorStateException e) {
                e.printStackTrace();
            }
        });
        Thread t2 = new Thread(() -> {
            try {
                deadLockTest.methodB();
            } catch (InterruptedException e) {

            } catch (IllegalMonitorStateException e) {
                e.printStackTrace();
            }
        });
        t1.setName(THREAD_1);
        t2.setName(THREAD_2);
        t1.start();
        t2.start();
        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {

        }
        Logcat("Finish Successfully!");
    }
}

// Output:
// Thread 1: try to get object_1's lock!
// Thread 1: get object_1's lock!
// Thread 1: Sleeping ... ...
// Thread 2: try to get object_2's lock!
// Thread 2: get object_2's lock!
// Thread 2: try to get object_1's lock!
// Thread 1: wake up! try to get object_2's lock!
{% endcodeblock %}
这段代码大概率会出现死锁。发生死锁时，输出如62\~68行。代码的执行顺序是不确定的，但是从这个例子的输出来看，要发生死锁，必然存在子顺序：
1.t<sub>1</sub>获得object_1的对象锁
2.t<sub>2</sub>获得object_2的对象锁
3.JVM去执行其他调度，这里就用休眠操作来模拟
4.t<sub>1</sub>尝试获取object_2的对象锁，但已被占用，阻塞。
5.t<sub>2</sub>尝试获取object_1的对象锁，但已被占用，阻塞。
这里就是环路等待的情况了！t<sub>1</sub>等t<sub>2</sub>，t<sub>2</sub>等t<sub>1</sub>，除非人为干预，否则永远持续。
从上面来看，使用synchronized是有可能发生死锁的！因为四个条件的合取并不恒为假。因此在使用时要慎重考虑，因为一旦发生死锁，程序就死掉。

### 4.2.死锁的避免
死锁的发生是有条件的。为了避免发生死锁，我们只需要破坏条件，使其不满足即可。分别考虑四个条件：
1.互斥条件。互斥条件必然满足的，破坏是不可能破坏的。
2.请求等待条件、不剥夺条件和环路等待条件互相配合。在请求等待时，线程至少持有一个对象锁。如果发生死锁时，通过破坏不剥夺条件和环路等待条件，有不多于[线程个数 - 共享资源个数]个线程主动释放当前持有的锁，那么尴尬的局面就可以缓解。为什么是不多于[线程个数 - 共享资源个数]个线程而不是全部线程？以~~哲♂学家~~哲学家进餐问题为例，考虑情况：所有哲学家都饿了，然后他们同时拿到左边的筷子。这个时候他们同时wait()，放下左边的筷子(释放资源)，并等待通知其他哲学家用完的通知(notify())。然而，从上帝视角来看，情况是这样的：所有哲学家先同时拿起左边的筷子，然后发现不对，然后全部放下左边的筷子，然后等，最后饿死。
这里的线程个数为2，共享资源个数为1，因为对于任意两个相邻的哲学家，他们会竞争1根筷子。因此，在这个例子中，当发生死锁时，任意一对相邻的哲学家，只要有[2 - 1]个哲学家放弃筷子，这时候死锁就能解除。因此，上面会发生死锁的代码可以如下改进：
{% codeblock lang:Java %}
import java.util.concurrent.TimeUnit;

public class DeadLockTest {

    private static final String THREAD_1 = "Thread 1";
    private static final String THREAD_2 = "Thread 2";

    private Object object_1 = new Object();
    private Object object_2 = new Object();

    private boolean isObject1Locked = false;

    private boolean isObject2Wait = false;

    public void methodA() throws InterruptedException, IllegalMonitorStateException {
        Logcat("try to get object_1's lock!");

        synchronized (object_1) {
            isObject1Locked = true;
            Logcat("get object_1's lock!");
            Logcat("Sleeping ... ...");
            TimeUnit.SECONDS.sleep(2);
            Logcat("wake up! try to get object_2's lock!");

            synchronized (object_2) {
                Logcat("get object_2's lock!");
                if (isObject2Wait) {
                    Logcat("I will finish, notify!");
                    object_2.notify();
                    isObject2Wait = false;
                }
            }
        }
        isObject1Locked = false;
    }

    public void methodB() throws InterruptedException, IllegalMonitorStateException {
        Logcat("try to get object_2's lock!");

        synchronized (object_2) {
            Logcat("get object_2's lock!");
            Logcat("try to get object_1's lock!");
            if (isObject1Locked) {
                isObject2Wait = true;
                object_2.wait();
                Logcat("Fortunately, object_1 is locked, release object_2's lock and wait!");
            }
            synchronized (object_1) {
                isObject1Locked = true;
                Logcat("get object_1's lock!");
            }
            isObject1Locked = false;
        }
    }

    private static void Logcat(String message) {
        System.out.println(Thread.currentThread().getName() + ": " + message);
    }

    public static void main(String[] args) {
        final DeadLockTest deadLockTest = new DeadLockTest();
        Thread t1 = new Thread(() -> {
            try {
                deadLockTest.methodA();
            } catch (InterruptedException e) {

            } catch (IllegalMonitorStateException e) {
                e.printStackTrace();
            }
        });
        Thread t2 = new Thread(() -> {
            try {
                deadLockTest.methodB();
            } catch (InterruptedException e) {

            } catch (IllegalMonitorStateException e) {
                e.printStackTrace();
            }
        });
        t1.setName(THREAD_1);
        t2.setName(THREAD_2);
        t1.start();
        t2.start();
        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {

        }
        Logcat("Finish Successfully!");
    }
}

// Output:
// Thread 1: try to get object_1's lock!
// Thread 2: try to get object_2's lock!
// Thread 1: get object_1's lock!
// Thread 2: get object_2's lock!
// Thread 2: try to get object_1's lock!
// Thread 1: Sleeping ... ...
// Thread 1: wake up! try to get object_2's lock!
// Thread 1: get object_2's lock!
// Thread 1: I will finish, notify!
// Thread 2: Fortunately, object_1 is locked, release object_2's lock and wait!
// Thread 2: get object_1's lock!
// main: Finish Successfully!
{% endcodeblock %}

这段代码的原理是：
2.1.当线程t<sub>2</sub>准备获取object_1的对象锁时，如果object_1已被锁住则放弃当前持有的object_2锁；
2.2.当线程t<sub>1</sub>将要释放object_2的对象锁时，如果t<sub>1</sub>处于等待状态，则通知它准备。

## 五、题外话：不够用？试试ReetrantLock！
ReetrantLock在Java8 api docs中是被这么描述的：
{% blockquote %}
A reentrant mutual exclusion Lock with the same basic behavior and semantics as the implicit monitor lock accessed using synchronized methods and statements, but with extended capabilities.
{% endblockquote %}
一个与基于monitor锁实现的同步方法或同步语句块有相同的行为和语义的可重入互斥锁，并具有扩展功能。
一个简单的demo：
{% codeblock lang:Java %}
private ReentrantLock lock = new ReentrantLock();
public void add() {
     lock.lock();
    try {
        i++;
    } finally {
        lock.unlock();
    }
}
{% endcodeblock %}
同步前只需要调用ReentrantLock.lock()锁住，使用完毕一定要解锁。为了增加可读性，可使用try...finally结构。
ReentrantLock的功能不仅限于此，它提供尝试获得锁，失败就放弃的功能，demo如下：
{% codeblock lang:Java %}
private ReentrantLock lock = new ReentrantLock();
public void add() {
    try {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
             i++;
        } else {
            System.out.println(Thread.currentThread().getName()+": failed to get lock!");
        }
    } catch (InterruptedException e) {
            e.printStackTrace();
    } finally {
        if(lock.isHeldByCurrentThread()){
            lock.unlock();
        }
    }
}
{% endcodeblock %}
lock.tryLock(1, TimeUnit.SECONDS) 表示尝试获得锁，如果超过一秒还没能获得锁则放弃获得并退出，不会重试。

## 六、总结
* synchronized锁住的是哪个对象？
对于同步语句块，synchronized锁住的是括号中的对象。对于同步方法，静态同步方法中synchronized锁住的是*本类的类对象*，而普通同步方法中的synchronized锁住的是*本类的实例对象*
* 如何合理使用synchronized？
synchronized是同步的，互斥的。在使用的时候必须考虑死锁的情况。通过考察发生死锁的四个必要条件，然后逐一破坏（互斥条件的破坏是不可能的），避免死锁的发生。在复杂的场景，可以考虑使用ReentrantLock。

## 七、感想
在明白原理的前提下，很多问题都能通过原理来得到解答。我第一次接触synchronized的时候，以为它锁的是代码块。然而明白原理后，不仅明白了锁的是对象，而且还明白了锁哪个对象，有什么区别。这是我对于本文的前半部分的感想。重点是死锁的部分，我翻了很多资料，包括大二的操作系统教材。在第一次学同步互斥的时候是一脸懵逼的，感觉难就简单的记一记背一背，没有理解。通过这次写博客的机会，我琢磨了好久，反反复复看死锁那章，终于理解了。相比死记硬背，理解后更能举一反三，融会贯通。所以看这种原理性的书还是要耐心，理解才好。还有，在查资料的过程中发现有很多名词，像多叉树一样互相联系着，感觉自己还有很多不懂，Java这条路还是任重道远。暂时就这么多，就这样吧。水平有限，如果本文有误，还望指正，谢谢~