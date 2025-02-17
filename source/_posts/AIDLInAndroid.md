---
title: 记一次AIDL的学习与实践
date: 2018-11-25 14:36:12
header-img: "aidlInAndroid_cover.jpg"
author:     "spirytusz"
tags: 
	- Android
	- 多进程
---

## 1. 前言
终于考完试了hhh，终于有时间更新博客了。正好之前自学跨进程编程，写个博文记录一下。

<!--more-->

## 2. Android中的多进程
在Android中，默认情况下，应用中的组件是运行在同一个进程的。我们可以在AndroidManifest.xml中使用`android:process`属性来为四大组件指定运行的进程。比如对于Service：
{% codeblock lang:xml %}
<service ...
    android:process=":playMusicProcess"
    ...>
</service>
{% endcodeblock %}
当然也可以通过JNI，在C/C++代码中调用fork()来开启进程，这里不讨论这种情况。

然而，好端端的为什么要使用多进程？因为多进程相较于单进程是有好处的。
首先，多进程意味着更多内存空间，OOM的概率更小。DVM会给每个进程分配固定的空间。多进程意味着会被分配更多空间，因此合理利用多进程，能够减少OOM发生的概率。
其次，像一些功能，相对独立，只需要在后台运行，那我们可以让它运行在另一个进程中的Service，即使UI进程被杀了，这个模块也能正常运行。

接下来看看多进程的实现，我使用AIDL来进行进程间的通信。

## 3. 准备知识：序列化
在跨进程通信中，从一个进程发送消息到另一个进程时，会先把消息进行序列化，通过Binder传给另一个进程，另一个进程再通过反序列化以获得消息。所以在实现跨进程通信之前，我们需要对序列化有一定的了解。
在Android开发中我们有两种序列化方案：`Serializable接口`和`Parcelable接口`。前者是Java自带的，后者是为Android系统量身打造的。两者各有优劣：
`Serializable接口`使用方便，但是性能开销较大，涉及大量I/O操作。
`Parcelable接口`性能开销小，但使用较为复杂。
基于性能优先的原则，我优先使用的是`Parcelable接口`。

对于一个实体类Music，我们让它实现Parcelable接口：
{% codeblock lang:Java %}
public class Music implements Parcelable {
	
	private String mMusicName;
	private long mMusicDuration;

	public Music(String musicName, long musicDuration) {
		mMusicName = musicName;
		mMusicDuration = musicDuration;
	}

	private Music(Parcel source) {
        mMusicName = source.readString();
        mMusicDuration = source.readLong();
    }

	public String getMusicName() {
        return musicName;
    }

    public long getMusicDuration() {
        return musicDuration;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(mMusicName);
        dest.writeLong(mMusicDuration);
    }

    public static final Parcelable.Creator<Music> CREATOR = new Creator<Music>() {
        @Override
        public Music createFromParcel(Parcel source) {
            return new Music(source);
        }

        @Override
        public Music[] newArray(int size) {
            return new Music[size];
        }
    };
}
{% endcodeblock %}

实现Parcelable接口，需要定义一个CREATOR变量，并实现方法`describeContents()`和`writeToParcel()`。
对于变量`CREATOR`，它是Parcelable.Creator<T\>接口一个实现的对象。其中，`createFromParcel()`是用来反序列化出一个对象，`newArray()`是用来反序列化出一个对象的数组。这里只需要返回一个空数组即可。
对于方法`describeContents()`，它返回对象内容的描述，如果内容中有文件描述符(File Descriptor, fd)就返回1，否则返回0。这里返回0。
对于方法`writeToParcel()`，它在序列化的时候被调用。在这里面序列化需要序列化的变量。
{% quote %}
踩坑点： 在`writeToParcel`按ABCD的顺序序列化，在反序列化的时候也一定要按照ABCD的顺序反序列化。否则会反序列化出一个不正确的对象。
{% endquote %}

序列化的一些注意点就基本这样。

## 4. 关于AIDL
AIDL是Android接口定义语言(Android Interface Definition Language, AIDL)，用于规定进程间通信的方式(这里我理解为进程间的通信协议)。
它和Java的接口是有些许不同的：
1. AIDL不支持定义静态变量，仅支持定义方法。
2. AIDL仅支持有限的数据类型：
	* 基本数据类型：int, long, char, boolean, double等
	* String和CharSequence
	* List，仅支持ArrayList，且里面的数据必须是AIDL支持的数据类型
	* Map，仅支持HashMap，不支持带泛型参数的Map([参考](http://wing-linux.sourceforge.net/guide/developing/tools/aidl.html#aidlsyntax))
	* Parcelable，支持所有实现了Parcelable接口的对象
	* AIDL，支持AIDL接口对象

{% quote %}
踩坑点： 在方法的参数列表中，除了基本类型和AIDL接口外，都需要标注方向in, out, inout
in代表数据只能从客户端流向服务端，即客户端 -> 服务端
out代表数据只能从服务端流向客户端，即客户端 <- 服务端
inout代表数据可以在客户端和服务端之间双向流动，即客户端 <-> 服务端
{% endquote %}

另外，Parcelable对象需要在aidl中显式的声明和import
如果实体Music.java位于java文件夹中的包com.example.exampleproject.entity包下，那就需要在aidl文件夹中的包com.example.exampleproject.entity下声明Music.aidl，内容为：
{% codeblock %}
// Music.aidl
package com.example.exampleproject.entity;

parcelable Music;
{% endcodeblock %}
然后在需要用到Music的AIDL文件中显式import出来，无论同不同包：
{% codeblock %}
import com.example.exampleproject.entity.Music;
{% endcodeblock %}

{% quote %}
如果你用Android Studio来编写AIDL文件，那你就暂时把AS当成文本编辑器吧，不要指望它会帮你代码补全，自动import相关包...
{% endquote %}

所以，一个正确的AIDL文件是这样的：
{% codeblock %}
// ITest.aidl
package com.example.exampleproject;

import java.util.List;
import java.util.Map;

import com.example.exampleproject.entity.Music;

interface ITest{
	
	void setMusic(in Music music);
	void setMap(in Map map);
	void setList(in List<Music> musicList);
}
{% endcodeblock %}

## 5. 使用AIDL开启多进程模式
好了，我们已经明白了AIDL的正确书写方法，接下来使用AIDL来实现进程间的通信。这里以双向通信为例子：Activity传一个int型的值给Service，Service处理后返回给Activity。
这里的Activity，我们取名为MainActivity，Service我们取名为MyService。
ok，既然是处理后返回，所以MainActivity中需要有一个回调接口来给MyService回调，但是AIDL不支持普通接口，但支持AIDL接口，因此，我们就用AIDL来声明MainActivity中的回调接口：
{% codeblock ICallback.aidl %}
package com.zspirytus.simpleaidltest;

interface ICallback {
    void callback(int a);
}
{% endcodeblock %}

Activity和Service之间的通信是通过Binder来实现的，而AIDL接口的内部类`Stub`的实现也是一个Binder，因为它继承了`android.os.Binder`接口，因此我们可以在AIDL接口中规定MainActivity和MyService的通信方式。由于我们需要从MainActivity中传值给MyService，并且要给MyService设置MainActivity的回调接口。因此就有：
{% codeblock IAIDLTest.aidl %}
package com.zspirytus.simpleaidltest;

import com.zspirytus.simpleaidltest.ICallback;

interface IAIDLTest {
    void testMethod(int a);
    void setCallback(ICallback callback);
}
{% endcodeblock %}
`testMethod()`是MainActivity向MyService传值的方法
`setCallback()`是MainActivity为MyService设置回调接口的方法。

MainActivity和MyService通信的桥梁已经规定好了，接下来就是实现它。我们在MyService中定义一个内部类MyBinder来实现它：
{% codeblock lang:Java %}
public class MyService extends Service {

	private ICallback mCallback;

	... ...

	private class MyBinder extends IAIDLTest.Stub {
        @Override
        public void testMethod(int a) throws RemoteException {
            Log.e(this.getClass().getSimpleName(), "MyService Receive Msg: " + a + "\t at Thread: " + Thread.currentThread().getName());
            if (mCallback != null)
                mCallback.callback(a);
        }

        @Override
        public void setCallback(ICallback callback) throws RemoteException {
            mCallback = callback;
        }
    }
    ... ...
}
{% endcodeblock %}

简单解释一下，MainActivity通过`setCallback()`为MyService设置回调接口，当MainActivity通过`testMethod()`传值给MyService时，MyService会处理传过来的值(这里以打印日志代替)，然后检查一下回调接口是否为空。如果不为空则调用它的回调方法，进而通知MainActivity。
{% quote %}
如果提示找不到IAIDLTest，可以尝试Clean一下Project尝试解决，如果Gradle报错，请检查AIDL文件是否正确。
{% endquote %}

然后补充一下MyService的生命周期，服务端就大功告成了：
{% codeblock lang:Java %}
public class MyService extends Service {

    private MyBinder mBinder;
    private ICallback mCallback;

    @Override
    public IBinder onBind(Intent intent) {
        if (mBinder == null)
            mBinder = new MyBinder();
        return mBinder;
    }

    private class MyBinder extends IAIDLTest.Stub {
        @Override
        public void testMethod(int a) throws RemoteException {
            Log.e(this.getClass().getSimpleName(), "MyService Receive Msg: " + a + "\t at Thread: " + Thread.currentThread().getName());
            if (mCallback != null)
                mCallback.callback(a);
        }

        @Override
        public void setCallback(ICallback callback) throws RemoteException {
            mCallback = callback;
        }
    }
}
{% endcodeblock %}

服务端已经解决了，现在来看客户端MainActivity。
Activity和Service之间的通信可以通过Activity绑定Service的形式来实现，多进程模式下也不例外，但有些许不同。
首先声明一个Binder来构建MainActivity与MyService通信的桥梁：
{% codeblock lang:Java %}
private IAIDLTest mBinder;
{% endcodeblock %}
然后以内部类的形式实现MyService回调MainActivity的回调接口：
{% codeblock lang:Java %}
private class ICallbackImpl extends ICallback.Stub {
    @Override
    public void callback(final int a) throws RemoteException {
        switch (a) {
            case 1:
                a1();
                break;
            case 2:
                a2();
                break;
        }
    }
}

private void a1() {
    mRemoteMsg.setText("Receive From Remote Service: a = 1");
}

private void a2() {
    mRemoteMsg.setText("Receive From Remote Service: a = 2");
}
{% endcodeblock %}
其次创建一个`ServiceConnection`:
{% codeblock lang:Java %}
conn = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        mBinder = IAIDLTest.Stub.asInterface(iBinder);
        try {
            mBinder.setCallback(mCallback);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        mBinder = null;
    }
};
{% endcodeblock %}

最后在适当生命周期绑定和解绑Service:
{% codeblock lang:Java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    ... ...
    bindService();
    ... ...
}

@Override
protected void onDestroy() {
    super.onDestroy();
    ... ...
    unbindService();
    ... ...
}

private void bindService() {
    Intent startServiceIntent = new Intent(MainActivity.this, MyService.class);
    bindService(startServiceIntent, conn, BIND_AUTO_CREATE);
}

private void unbindService() {
    unbindService(conn);
}
{% endcodeblock %}

然后在适当的时候发送请求就可以了，比如单击按钮发送请求：
{% codeblock lang:Java %}
private void initView() {
	findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
            start();
        }
    });
}

private void start() {
    try {
        mBinder.testMethod(A);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
{% endcodeblock %}

但是，毕竟是跨进程通信，当前进程下发起远程请求的线程是会被挂起的。如果发起的请求是一个耗时操作，并且是UI线程发起的，那就会有ANR的风险。因此，安全起见，在发起跨进程请求时，推荐切换到另一个线程后再发起。所以上面的程序可以这么改进：
{% codeblock lang:Java %}
private void start() {
    Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                mBinder.testMethod(A);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("myThread");
    thread.start();
}
{% endcodeblock %}

然后呢，等到服务端把请求执行完毕后，客户端发起请求的线程会被唤醒，然后继续执行。所以我们不能在回调方法中执行UI更新操作，因为回调方法并不在UI线程中执行(前面在非UI线程中发起请求)。如果需要更新UI，需要切换到UI线程后再更新UI。所以需要如下改进：
{% codeblock lang:Java %}
private class ICallbackImpl extends ICallback.Stub {
    @Override
    public void callback(final int a) throws RemoteException {
        mRemoteMsg.post(new Runnable() {
            @Override
            public void run() {
                switch (a) {
                    case 1:
                        a1();
                        break;
                    case 2:
                        a2();
                        break;
                }
            }
        });
    }
}
{% endcodeblock %}

因此，即使服务端执行耗时操作，界面也不会卡死：
{% codeblock lang:Java %}
@Override
public void testMethod(int a) throws RemoteException {
    SystemClock.sleep(5000);
    Log.e(this.getClass().getSimpleName(), "MyService Receive Msg: " + a + "\t at Thread: " + Thread.currentThread().getName());
    if (mCallback != null)
	mCallback.callback(a);
}
{% endcodeblock %}

{% quote %}
服务端的方法会在Binder线程池中的线程运行，因此不必担心主线程会被阻塞
{% endquote %}

还有一个小问题，如果已经绑定的Service意外死亡怎么办（比如说被系统杀死了）？我们只需要在`ServiceConnect#onServiceDisconnected()`中重新连接服务即可。
{% codeblock lang:Java %}
@Override
public void onServiceDisconnected(ComponentName componentName) {
    bindService();
}
{% endcodeblock %}
{% quote %}
MyService被杀死也没关系，`MainActivity#bindService()`中在绑定MyService时，flag已经被设置为BIND_AUTO_CREATE，会重新启动MyService的。
{% endquote %}
由于代码太长，我没有全放，只放一些比较关键的，完整代码可以看[这里](https://github.com/zkw012300/SampleCodeRepo/tree/master/simpleaidltest)

好了，简单进程间通信的实现就完成了，总结一下，基本步骤：
1. 创建AIDL文件搭建客户端和服务端的沟通桥梁
2. 服务端实现AIDL接口
3. 客户端通过ServiceConnection#onServiceConnected()获取Binder对象
4. 在ServiceConnection#onServiceDisconnected()中重新连接，防止Service意外死亡
5. 在客户端使用Binder对象在适当的线程向服务端发送请求
6. 在客户端的适当线程中回调

## 6. 实践 - 简单的音乐播放器实现
明白了进程间通讯的流程，接下来我们来实现一个简单的音乐播放功能吧。
我们需要完成：
1. Activity和Service在不同进程
2. 默认进程中的Activity控制音乐播放暂停，接收来自另一个进程的Service发送的播放进度并显示
3. 另一个进程的Service负责播放音乐，发送播放进度给Activity。

这个音乐播放器，至少要能播放、暂停、显示播放进度。我们可以把播放暂停和显示播放进度这两个功能分离，分别管理。
对于播放暂停，只需要简单的定义两个方法`play()`和`pause()`即可：
{% codeblock IPlayControl.aidl%}
// IPlayControl.aidl
package com.zspirytus.simplemusicplayer;

import com.zspirytus.simplemusicplayer.entity.Music;

interface IPlayControl {
    void play(in Music music);
    void pause();
}
{% endcodeblock %}
对于获取播放进度的问题，我们采取订阅 - 发布的模式，需要获取播放进度的对象，传一个回调接口给被观察者，即服务端。
以下分别是订阅取消订阅和回调接口的AIDL文件：
{% codeblock IPlayProgressRegister.aidl%}
// IPlayProgressRegister.aidl
package com.zspirytus.simplemusicplayer;

import com.zspirytus.simplemusicplayer.IOnProgressChange;

interface IPlayProgressRegister {
    void registerProgressObserver(IOnProgressChange observer);
    void unregisterProgressObserver(IOnProgressChange observer);
}
{% endcodeblock %}

{% codeblock IOnProgressChange.aidl %}
// IOnProgressChange.aidl
package com.zspirytus.simplemusicplayer;

interface IOnProgressChange {
    void onProgressChange(int seconds);
}
{% endcodeblock %}

接下来，就要实现服务端了。但是，前面音乐播放控制功能和进度获取功能已经被我分离了，换言之，就是有两个Binder。但是一个Service只能有一个Binder和外界沟通啊？怎么解决？不如我们定义一个Binder连接池，根据适当的请求码binderCode来返回对应的Binder。这个Binder连接池能被客户端拿到，所以它应该由AIDL实现：
{% codeblock %}
// IBinderPool.aidl
package com.zspirytus.simplemusicplayer;

interface IBinderPool {
    IBinder getBinder(int binderCode);
}
{% endcodeblock %}

Binder连接池已经定义好了，接下来就是实现服务端了，同样的，我还是用内部类的方式实现：
{% codeblock lang:Java %}
public class PlayMusicService extends Service {
	
	... ...

	private static final int PLAY_CONTROL_BINDER = 1;
    private static final int PLAY_PROGRESS_REGISTER_BINDER = 2;

	private BinderPool mBinder;
    private PlayControl mPlayControl;
    private PlayProgressRegister mPlayProgressRegister;

    private RemoteCallbackList<IOnProgressChange> mIOnProgressChangeList = new RemoteCallbackList<>();

    ... ...

	public class BinderPool extends IBinderPool.Stub {
        @Override
        public IBinder getBinder(int binderCode) throws RemoteException {
            switch (binderCode) {
                case PLAY_CONTROL_BINDER:
                    if (mPlayControl == null)
                        mPlayControl = new PlayControl();
                    return mPlayControl;
                case PLAY_PROGRESS_REGISTER_BINDER:
                    if (mPlayProgressRegister == null)
                        mPlayProgressRegister = new PlayProgressRegister();
                    return mPlayProgressRegister;
                default:
                    return null;
            }
        }
    }

    private class PlayControl extends IPlayControl.Stub {
        @Override
        public void play(Music music) throws RemoteException {
            MyMediaPlayer.getInstance().play(music);
        }

        @Override
        public void pause() throws RemoteException {
            MyMediaPlayer.getInstance().pause();
        }
    }

    private class PlayProgressRegister extends IPlayProgressRegister.Stub {
        @Override
        public void registerProgressObserver(IOnProgressChange observer) throws RemoteException {
            mIOnProgressChangeList.register(observer);
        }

        @Override
        public void unregisterProgressObserver(IOnProgressChange observer) throws RemoteException {
            mIOnProgressChangeList.unregister(observer);
        }
    }
    ... ...
}
{% endcodeblock %}
在IBinderPool.Stub的实现BinderPool中，会根据传递过来的binderCode来返回对应的已被实例化的Binder。
`MyMediaPlayer`是我对MediaPlayer的一个很简单的封装，拿来当例子够了。
接下来补上Service的生命周期就ok了，代码比较长，我就不贴了。

接下来看客户端的实现：
首先定义从Service端获得的Binder对象，然后在ServiceConnection中获取Binder和重连Service。
{% codeblock lang:Java %}
private IBinderPool mBinder;
private ServiceConnection conn;

conn = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        mBinder = IBinderPool.Stub.asInterface(iBinder);
        register();
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        bindService();
    }
};
{% endcodeblock %}

接着，实现回调接口并传给Service，以供其传递播放进度给Activity（上一个步骤的`register()`方法已经把回调接口传给了Service）：
{% codeblock lang:Java %}
private class IOnProgressChangeImpl extends IOnProgressChange.Stub {
    @Override
    public void onProgressChange(final int currentMilliseconds) throws RemoteException {
        mProgressText.post(new Runnable() {
            @Override
            public void run() {
                mProgressText.setText(DateUtil.getMinutesSeconds(currentMilliseconds));
            }
        });
    }
}
{% endcodeblock %}

最后就是请求部分，在客户端中，我们获得是Binder连接池的对象，我们可以传binderCode来获得对应的Binder:
{% codeblock lang:Java %}
private static final int PLAY_CONTROL_BINDER = 1;
private static final int PLAY_PROGRESS_REGISTER_BINDER = 2;

private void play() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            Music sampleMusic = MusicFactory.getSampleMusic();
            try {
                if (mPlayControl == null) {
                    IBinder iBinder = mBinder.getBinder(PLAY_CONTROL_BINDER);
                    mPlayControl = IPlayControl.Stub.asInterface(iBinder);
                }
                mPlayControl.play(sampleMusic);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }).start();
}

private void pause() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                if (mPlayControl == null) {
                    IBinder iBinder = mBinder.getBinder(PLAY_CONTROL_BINDER);
                    mPlayControl = IPlayControl.Stub.asInterface(iBinder);
                }
                mPlayControl.pause();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }).start();
}
{% endcodeblock %}

最后在AndroidManifest.xml中为Service指定进程，就大功告成了。
{% codeblock lang:xml %}
<service
    android:name="com.zspirytus.simplemusicplayer.PlayMusicService"
    android:enabled="true"
    android:exported="true"
    android:process=":playMusicService"></service>
{% endcodeblock %}

由于代码太长，我没有全放，只放一些比较关键的，完整代码可以看[这里](https://github.com/zkw012300/SampleCodeRepo/tree/master/simplemusicplayer)。

## 7. 总结
我目前的理解是，在Android跨进程通讯中，一般是序列化+接口+线程切换。只要注意这三点一般没大问题。