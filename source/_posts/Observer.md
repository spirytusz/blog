---
title: 设计模式——观察者模式
date: 2018-07-26 23:54:55
header-img: "observer_cover.jpeg"
author:     "spirytusz"
tags: 
	- 设计模式
---

## 前言
嗯，观察者模式在Android开发中还是挺常用的，比如说Adapter数据更新后RecyclerView的刷新，以及四大组件的Broadcast等等。接下来本文将简单谈谈设计模式中的观察和模式。
为了有更好的食用体验，本文会结合一个简单的例子，让你更好的理解观察者模式。
<!--more-->

## 一、观察者模式
先来看看定义：定义对象之间的一对多依赖关系，当一个对象改变状态时，它的所有依赖对象都会自动获得通知。
emmm...一对多依赖关系？？？状态改变依赖对象会被通知？？？定义有点拗口，不如直接看看一个例子。
小时候各位都有做过眼保健操吧，当广播响起眼保健操的音乐时，我们就开始做眼保健操。此时，广播和学生构成了一对多的依赖：当广播播放眼保健操的音乐时，即广播的状态发生改变时，我们获得要开始做眼保健操的通知。


## 二、观察者模式中的角色
观察者模式，就像它的名字一样，这个设计模式中有一个角色叫『观察者』，同样的也有一个角色叫『被观察者』。它们都是抽象出来的概念，因此它们的具体的实现是『具体观察者』和『具体被观察者』。
在上面的例子中，『广播』扮演的的是『被观察者』的角色；『发送眼保健操音乐的广播』扮演的是『具体被观察者』的角色；『学生』扮演的是『具体观察者』的角色；而『观察者』的角色并不明显，它扮演的是接口的角色。如下例子：
定义观察者：
{% codeblock lang:Java Observer.java %}
public interface Observer {

    void doSomething(BroadCast broadCast);
}
{% endcodeblock %}

定义具体观察者——Student
{% codeblock lang:Java Student.java %}
public class Student implements Observer {

    private static final String TAG = Student.class.getSimpleName();

    @Override
    public void doSomething(BroadCast broadCast) {
        if ("send Eye Exercises BroadCast! ".equals(broadCast.getBroadCast())) {
            System.out.println(TAG + ": I will do eye exercises!");
        } else {
            System.out.println(TAG + ":I will be back home!");
        }
    }

}
{% endcodeblock %}

定义被观察者——BroadCast：
{% codeblock lang:Java BroadCast.java%}
public abstract class BroadCast {

    private static final List<Observer> observers = new ArrayList<>();

    public void register(Observer observer) {
        if (!observers.contains(observer)) {
            synchronized (BroadCast.class) {
                if (!observers.contains(observer)) {
                    observers.add(observer);
                }
            }
        }
    }

    public void unRegister(Observer observer) {
        if (!observers.contains(observer)) {
            synchronized (BroadCast.class) {
                if (!observers.contains(observer)) {
                    observers.remove(observer);
                }
            }
        }
    }

    protected void notifyAllObserver(){
        Iterator<Observer> iterator = observers.iterator();
        while(iterator.hasNext()){
            Observer observer = iterator.next();
            observer.doSomething(this);
        }
    }

    public abstract String getBroadCast();

    public abstract void sendBroadCast();

}
{% endcodeblock %}

定义具体的被观察者——眼保健操广播和放学铃声广播：
{% codeblock lang:Java EyeExercisesBroadCast.java%}
public class EyeExercisesBroadCast extends BroadCast {

    private static final EyeExercisesBroadCast INSTANCE = new EyeExercisesBroadCast();

    private String broadCastMessage;

    private EyeExercisesBroadCast(){

    }

    public static EyeExercisesBroadCast getInstance(){
        return INSTANCE;
    }

    @Override
    public String getBroadCast() {
        return broadCastMessage;
    }

    @Override
    public void sendBroadCast() {
        broadCastMessage = "send Eye Exercises BroadCast! ";
        notifyAllObserver();
    }
}
{% endcodeblock %}

{% codeblock lang:Java FinishClassBroadCast.java%}
public class FinishClassBroadCast extends BroadCast {

    private static final FinishClassBroadCast INSTANCE = new FinishClassBroadCast();

    private String broadCastMessage;

    private FinishClassBroadCast(){

    }

    public static FinishClassBroadCast getInstance(){
        return INSTANCE;
    }

    @Override
    public String getBroadCast() {
        return broadCastMessage;
    }

    @Override
    public void sendBroadCast() {
        broadCastMessage = "send finish class BroadCast! ";
        notifyAllObserver();
    }
}
{% endcodeblock %}

另外在定义两个辅助类Main和Teacher，Main负责执行代码，而Teacher是广播的管理者，可以管理广播的发送
{% codeblock lang:Java Main.java %}
public class Main {

    public static void main(String[] args) {
        Student student = new Student();
        BroadCast broadCast1 = EyeExercisesBroadCast.getInstance();
        BroadCast broadCast2 = FinishClassBroadCast.getInstance();
        broadCast1.register(student);
        broadCast2.register(student);
        Teacher teacher = new Teacher();
    }
}
{% endcodeblock %}

{% codeblock lang:Java Teacher.java %}
public class Teacher {

    private static final String TAG = Teacher.class.getSimpleName();

    private BroadCast[] broadCasts;

    public Teacher() {
        init();
        System.out.println(TAG+": I send eye exercises broadcast! ");
        sendEyeExercisesBroadCast();
        try {
            TimeUnit.SECONDS.sleep((int) (Math.random() * 6));
        } catch (InterruptedException e) {

        }
        System.out.println(TAG+": I send finish class broadcast! ");
        sendFinishClassBroadCast();
    }

    private void init() {
        broadCasts = new BroadCast[2];
        broadCasts[0] = EyeExercisesBroadCast.getInstance();
        broadCasts[1] = FinishClassBroadCast.getInstance();
    }

    private void sendEyeExercisesBroadCast() {
        broadCasts[0].sendBroadCast();
    }

    private void sendFinishClassBroadCast(){
        broadCasts[1].sendBroadCast();
    }
}
{% endcodeblock %}
此时运行程序，当广播播放眼保健操的音乐时，学生便收到通知，开始做眼保健操；当广播播放放学铃声时，学生便收到通知，开始离校。运行结果如下：
{% asset_img result.png 运行结果 %}

## 三、观察者模式的工作原理
在观察者模式中，Observer观察Observable。这个过程有几个关键点：
* 与大部分现实不同的是，Observer不是主动观察，而是被动的接收来自Observable的通知
* Observable要知道谁在观察它，才能在状态改变的时候通知，因此它需要维护一个List<Observer\>
* Observer中要有一个传入Observable参数的回调方法，以便Observer在收到通知时能够获得Observable的状态

首先，Observer是被动的接收来自Observable的通知，因为我们引入观察者模式的目的之一就是为了避免轮询消耗CPU资源，所以比较合适的方法应该是当Observable状态发生改变时再去通知Observer。就像上面的例子，脑补一下画面：学生不必不停的问老师：放学了吗？放学了吗？放学了吗？...老师回答：没有。没有。没有。...放学了。这样很喜感，也太消耗体力(CPU资源)。

其次，正是因为Observable需要通知Observer，所以Observable需要知道谁是Observer，即需要存储所有Observer对象，因此它需要维护一个List<Observer\>，至于List<Observer\>的数据结构如何选用，就要看看实际情况了。就像上面的例子：因为广播(即Observable)知道它要通知的是学生，所以才会被安装在教室和走廊里(不然装在闹市区突然响起眼保健操的音乐，路人一脸黑人问号)。

最后，Observer需要知道Observable的通知内容，因此Observable要在回调函数(的参数)中放入通知内容，Observer可以在回调函数中(的参数)获得通知内容。在上面的例子中，学生是可以通过广播获得放学的信息的。

另外，如果存在优先级的问题，可以List<Observer\>把按优先级排序以下，再进行通知。如果在通知的时候(for循环里面)加一点逻辑，可以实现类似于Android系统中有序广播的功能。

## 四、回调函数的参数列表
在第三部分提到，Observer可以通过回调函数获取通知内容。但是这个通知内容是什么？是直接传一个Observable参数好，还是直接传message参数的参数好？还是两个一起传？不如分别讨论。

### 4.1.只传Observable参数
只传Observable参数，简单，不会有一大堆重载函数。但是可能会有意想不到的安全问题：Student类实现了Observer接口，可以通过doSomething(BroadCast)获取BroadCast对象，然而BroadCast的notifyAllObserver方法是public的！即使我们相信学生不会为了恶作剧而重复发广播，但是这个问题还是有可能发生的。从编程的角度，在代码运行时，如果观察者接收到信息后调用BroadCast.notifyAllObserver()方法后，是会爆栈的。比如说，修改Student.doSomething(BroadCast)方法：
{% codeblock lang:Java Student.doSomething(BroadCast) %}
@Override
public void doSomething(BroadCast broadCast) {
    if("send Eye Exercises BroadCast! ".equals(broadCast.getBroadCast())){
        System.out.println(this.getClass().getName()+": I will do eye exercises!");
    } else {
        System.out.println(this.getClass().getName()+":I will be back home!");
    }
    broadCast.notifyAllObserver();
}
{% endcodeblock %}
这里增加了一行broadCast.notifyAllObserver()，再重新执行main方法，结果如下：

{% asset_img StackBomb.png StackBomb %}

很不幸爆栈了。因此我们要把notifyAllObserver方法屏蔽掉，只有BroadCast及其它的子类能碰。因此notifyAllObserver方法需要被protected关键词修饰，并且其BroadCast的子类需要和BroadCast同包。修改后包结构如下：

{% asset_img PackageStructure.png Package Structure %}

通过修改notifyAllObserver()的访问权限，就能解决安全问题。

### 4.2.只传message参数
只传message参数？似乎看起来并没有什么问题。但是message可能不止一个！而且Observable不知道Observer需要什么样的message。假设多个Observer要观察Observable中的n个message中任意多个，它们需要的message可能各不相同。所以回调函数就必须要有2<sup>n</sup>个，这样才可以满足所有Observer的需求。显然这样是不可取的，因为重载函数的增加降低了程序的可维护性。同时，当Observer观察多个Observable时，当某一个Observable通知它时，Observer无法得知是哪个Observable更新了，所以只传message参数是有一定局限的。

### 4.3.同时传Observable和message
4.2中已经说明，当message有多个时，需要对应指数级个的回调函数，所以在这种多个message的情况下，还是尽量不要传message参数为好。

所以，个人认为回调函数因只需要传入一个Observable参数。

## 五、总结
一个类可以实现Observer接口来获得『观察』的功能；一个类可以实现Observable接口/抽象类来获得『可观察』的属性。当观察者需要观察被观察者时，它需要实例化出一个被观察者对象，然后调用被观察者的注册方法来注册自己，以便被观察者状态改变时，能够通知自己。
个人认为，观察者模式本质是维护一个回调函数集合，在被观察者发生改变时，被观察者便逐个调用这些回调函数，这时观察者就被通知了。
另外，基于观察者模式的特性，个人认为观察者模式特别适用于一些异步操作，比如IO操作和网络请求(下载图片并显示在ImageView上等)。

## 六、感想
事实上观察者模式不是我第一个接触的设计模式，我第一个接触的设计模式是『构建者模式』，当时是在学Android开发中的自定义View，没有资料来源所以只能硬啃别人的自定义View的代码，在里面View有必选属性和可选属性，多个构造方法重载和JavaBean模式都不理想，因此构建者模式才被设计出来，结合了这个场景，我也就理解了构建者模式的设计初衷和原理。回到观察者模式，在学Android开发的时候，使用到了RecyclerView，当时不理解为什么它的数据更新的机制，后来看了这本书[《Android开发进阶从小工到专家》](https://book.douban.com/subject/26744163/)，原来RecyclerView的数据更新是适配器模式+观察者模式！结合这本书里面的部分源码，我才理解了RecyclerView的数据更新机制，也就顺手理解了观察者模式的设计初衷。
所以设计模式，还是要结合实际场景来学，才能更容易理解然后运用。








