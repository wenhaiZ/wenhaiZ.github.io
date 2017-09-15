---
layout: post
title: "Handler 的正确使用"
date: 2017-08-12 09:26:00 +0800
tags: [UsingCorrectly,Code]
comments: true
subtitle: "我们的目标是，没有 warning"
published: true
---
在 Android 中，`Handler` 主要用来完成线程间的通信工作。由于 Android 只允许 UI 线程更新 UI，因此通过非 UI 线程更新 UI 的工作就可以通过 `Handler` 来完成，步骤如下：
1. 在 UI 线程创建 `Handler` 对象，并重写 `handleMessage()` 方法
2. 在子线程中通过 `Handler` 发送 `Message`    

UI 线程收到 `Message` 后，会执行 `handleMessage()` 中的操作   

示例代码：
```java
public class MainActivity extends AppCompatActivity {
    private static final int MSG_UPDATE = 0x00;
    private TextView textView;
    //创建 Handler
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case MSG_UPDATE:
                    //更新 UI
                    textView.setText("update from other thread");
                    break;
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = findViewById(R.id.textView);
        //开启一个子线程
        new Thread() {
            @Override
            public void run() {
                super.run();
                try {
                    //模拟耗时
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //发送消息
                handler.sendEmptyMessage(MSG_UPDATE);
            }
        }.start();
    }
}

```
`Handler` 的使用是 Android 开发中很基础的内容，这里要说的是另外一个问题。
## 碍眼的 Warning
如果在 `Android Studio 2.3` 中写上面的代码，你会发现 `new Handler()` 那句出现了 `warning`，我使用的是 `Android Studio 3.0` ，`warning` 更加壮观：  

![handler_warning](/assets/img/handler_warning.jpg)

看一下 warning 信息：  
`This Handler class should be static or leaks might occur.`

编译器警告说 `Handler class` 必须是静态的，否则可能会引发内存泄漏。

为什么 `Handler` 类不声明为静态就可能会引发内存泄漏呢？    

在说明这个问题之前，需要先储备一些知识。

## 内存泄漏与内存溢出
先来看内存泄漏，维基百科解释的十分清楚：
>在计算机科学中，内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。

而内存溢出大家都知道，就是内存不够用了，`Java` 抛出的 `OOM` 异常（Out Of Memory）就是内存溢出的意思。

由于内存泄漏导致系统不能回收已经不再使用的内存，因此如果内存泄漏频繁发生，系统可用内存会越来越小，这样当应用程序申请内存时，就可能会发生内存溢出。

用一句话总结二者关系就是，内存溢出不一定是内存泄漏导致的，但频繁发生的内存泄漏最后一定会导致内存溢出。

关于内存泄漏和内存溢出，你可以看一下[这篇文章](http://www.jianshu.com/p/e97ed5d8a403)。

## 消息分发机制
>这篇文章不会详细介绍消息分发机制的全部内容，只会解释用到的部分。关于消息分发机制更详细的内容，可以参考[这篇博客](http://blog.csdn.net/guolin_blog/article/details/9991569)。

Android 中每个 Looper 线程（创建线程时调用了 `Looper.prepare()` 和 `Looper.loop()`，UI 线程就是这样的线程）都有一个 `MessageQueue` 用于保存 `Message` 对象，`Looper` 通过 `loop()` 方法来获取 `Message` 并对其进行分发。   

看一下 `Looper.loop()` 的源码：
```java
public static void loop() {
    //....
    final MessageQueue queue = me.mQueue;
    for (;;) {
        //获取 Message
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        //.....
        try {
            //分发 Message
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        //....
        msg.recycleUnchecked();
    }
    //......
}
```
`loop()` 方法先通过 `queue.next()` 获取一个 `Message` 对象，然后通过 `msg.target.dispatchMessage(msg)` 进行消息分发，而 `msg.target` 就是发送这个消息的 Handler。
>这一点可以通过 `Handler` 的 `enqueueMessage()` 方法看出，所有 `sendMessageXXX()` 的方法最终将会调用这个方法。
>```java
>private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
>        //为 msg.tatget 赋值
>        msg.target = this;
>        if (mAsynchronous) {
>            msg.setAsynchronous(true);
>        }
>        return queue.enqueueMessage(msg, uptimeMillis);
>}
>```

另外需要注意的是，在调用 `queue.next()` 方法时可能会阻塞，因为 `MessageQueue` 的 `next()` 方法中对于分发时间在当前时间之后的 `Massage` 会进行等待。

再来看 `MessageQueue` 的 `next()` 方法源码：
```java
Message next() {
    //....
    int nextPollTimeoutMillis = 0;
    //循环
    for (;;) {
        //....
        synchronized (this) {
            //获取当前时间
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            //获得下一个 message
            Message msg = mMessages;
            if (msg != null) {
                //当前时间小于 msg 的分发时间
                if (now < msg.when) {
                    //设置 nextPollTimeOutMillis
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    //.....
                } else {
                    // 当前时间到了 msg 的分发时间，循环结束，方法返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            //....
        nextPollTimeoutMillis = 0;
    }
}
```
可以看到对于分发时间晚于当前时间的 `msg`, `next()` 会一直循环，直到当前时间到达 `msg` 的分发时间，才会 `return msg` 。

现在我们知道了，`Looper` 通过 `loop()` 来获取 `Message` 对象，并通过  `msg.target.dispatchMessage()` 来进行消息分发。而 `loop()` 方法又是通过 `MessageQueue` 的 `next()` 方法来获取下一个 `Message` 对象的，该方法对分发时间晚于当前时间的 `msg` 会在循环中等待。  

有了这些知识，就可以解释非静态声明的 `Handler` 为什么可能引发内存泄漏了。

## Handler 引发内存泄漏的原理

先来看一段代码，假如我通过 `Handler` 发送一个这样的消息，然后退出 `Activity`：
```java
 handler.sendMessageDelayed(handler.obtainMessage(MSG_UPDATE), 1000 * 60 * 10);
```
这样会有什么问题？

上面的代码会在 `Looper` 的 `MessageQueue` 中添加一个 `Message` 对象，这个 `msg` 的分发时间是 `10` 分钟后。   

当 `Looper` 通过 `loop()` 获取 `Message` 时，由于 `MessageQueue` 的 `next()` 方法还在阻塞，所以 `Looper` 不会退出。   
`MessageQueue` 中的 `msg` 持有 `Handler` 的引用，而 `Handler` 是通过非静态内部类的方式声明的，它会持有外部类也就是 `Activity` 的引用，而当前 `Activity` 已经退出了，本该由 `GC` 回收，但是由于 `Handler` 的引用，将不会被回收，这样就引发了内存泄漏。

如果这种情况多次发生，那么用户在启动新应用时就可能会面临应用程序崩溃的问题，因为内存溢出了。

## 使用 Handler 的正确姿势 
`Handler` 之所以会引发内存泄漏，归根结底还是因为声明时采用了**非静态内部类**的方式。  
  
在 `Java` 中，非静态内部类实例会持有外部类实例的引用，这样当内部类实例生命周期大于外部类实例时，外部类不会被回收从而引发内存泄漏。

要怎么解决呢？  

其实编译器的警告详情里已经告诉我们处理方式了：
>Since this Handler is declared as an inner class,it may pervent the outer class from being garbage collected.
>If the Handler is using a Looper or MessageQueue for a thread other than the main thread,then there is no issue.
>If the Handler is using the Looper or MessageQueue of the main thread,you need to fix your Handler declaration,as follows:  
> 1. Declare the Handler as a static class;  
> 2. In the outer class,instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler;  
> 3. Make all references to members of the outer class using the WeakReference Object   

它甚至还为我们简述了一下内存泄漏的原因。


处理方式就是采用静态内部类+弱引用，修改后的代码如下：
- `Handler` 声明
```java
//静态内部类
static class MyHandler extends Handler {
    
    private WeakReference<MainActivity> mainActivityWeakReference;

    //构造方法传入 MainActivity 的弱引用
    MyHandler(WeakReference<MainActivity> activityWeakReference) {
        mainActivityWeakReference = activityWeakReference;
    }

    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        //通过弱引用获取 Activity  
        MainActivity activity = mainActivityWeakReference.get();
        //进行判空，因为 Activity 可能已经被回收
        if (activity != null) {
            //handle message
            switch (msg.what) {
                case MSG_UPDATE:
                    activity.textView.setText("update");
            }
        } else {
                //activity has been gc
        }
    }
}
```
- `Handler` 实例化
```java
//在 Activity 中创建 Handler
private Handler handler = new MyHandler(new WeakReference<>(this));
```   

搞定了，不过要注意一下在 `handleMessage()` 中要对 `Activity` 判空，因为此时 `Activity` 可能已经被回收了。

至于为什么要用弱引用，看一下 `Java` 中四种引用类型的回收时机就知道了。     


|引用类型|回收时机|
|:-----------:|:-----------:|
|强引用|不会被回收，内存不足时抛 OOM 异常|
|软引用|内存不足时被回收|
|弱引用|GC 运行时回收|
|虚引用|任何时候都可能被回收|   

>1. 上表的回收时机是指对象只存在当前类型的引用时
>2. 此表只是简单总结，关于四种引用类型的详细内容可以参考《深入理解 Java 虚拟机》或者[这篇博客](http://coderbao.com/2016/05/01/Java-references/)。

使用弱引用，在 `Activity` 退出前，可以通过 `get()` 获取实例；在 `Activity` 退出后，它就只存在 `Handler` 对其的弱引用，那么可以被及时回收。

## 总结
`Handler` 是 Android 开发中很常用的 `API`，但由于 Android 特殊的消息分发机制，使用不当就会引发内存泄漏。通过 **静态内部类** 的方式声明 `Handler`，使用 **弱引用** 访问外部类的实例可以避免内存泄漏的发生。 

其实，如果我们不通过 `Handler` 发送可能会在 `Activity` 退出后才分发的 `Message`，即使使用非静态内部类的形式声明 `Handler`，内存泄漏也不会发生。  
但我们不能对编译器的 `warning` 视而不见，通过处理这些 `warning` ，也能让我们学到一些使用 API 的正确姿势。    

毕竟，姿势很重要。(完)
