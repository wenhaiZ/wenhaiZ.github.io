---
layout: post
title: "Android 消息机制原理解析"
date: 2017-09-21 22:29:00 +0800
tags: [Code,Android]
comments: true
subtitle: "Message/MessageQueue/Looper/Handler"
code-link: "/assets/code/170921.md"
---
>本文先从整体上对 Android 消息机制进行简述，然后通过分析这四个类的核心源码来介绍其实现原理。

提到 Android 中的消息机制，最容易想到就是 [Handler](https://developer.android.google.cn/reference/android/os/Handler.html) 和 [Message](https://developer.android.google.cn/reference/android/os/Message.html)，如果你创建过 Handler 线程，可能还会想到 [Looper](https://developer.android.google.cn/reference/android/os/Looper.html)。 

Handler 和 Message 是开发过程中经常用到的 API，它是 Android 消息机制的上层调用接口。  

由于 Android 不允许子线程更新 UI，所以需要通过 Handler 来实现子线程与 UI 线程的交互，然而 Handler 不仅仅可以用来更新 UI，抽象一点的说，Handler 的作用**是将任务转移到创建 Handler 的线程中继续执行。**  

Android 消息机制除了涉及上面提到的 Handler、Message 和 Looper 外，还有一个很重要的类——[MessageQueue](https://developer.android.google.cn/reference/android/os/MessageQueue.html)，这四个类均位于`android.os` 包下。   

如果你之前了解过消息机制，应该对这四个类有所了解，如果不了解也没有关系，你现在只要能分清这四个类就可以了。   
   
# 概述  
先来简要介绍这四个类的作用：
- Message：包含消息的基本信息，可携带数据，每个 Message 都持有对发送它的 Handler 的引用
- Handler：用于发送和处理 Message
- MessageQueue：根据 Message 的分发时间以 **链表** 的形式存储所有未处理的 Message 
- Looper：不断从 MessageQueue 中取出下一个 Message 并通知其对应的 Handler 进行处理   

可以用下图解释他们是怎样互相协作来完成线程间通信的了：  
![Handler机制](/assets/img/post/handler.png)   

概括成一句话就是：**Message 由 Handler 发送，保存在 MessageQueue 中，然后 Looper 不断从 MessageQueue 中取出下一条待处理消息，并通知发送它的 Handler 进行处理。**  

需要注意的是，Handler 发送消息是在 thread2 中完成的，而 Handler 的创建和对 Message 的处理都是在 Thread1 中的，换句话说，**Handler 发送消息是在其他线程(当然也可以在创建 Handler 的线程)中完成的，对 Message 的处理过程是在其被创建的线程中进行的。**   

下面从源码的角度来分析它们是如何完成各自工作并进行配合的，先从最简单的 Message 开始。
## Message 
Message 定义了消息的基本信息并且可以携带数据，它实现了 Parcelable 接口，是可序列化的。  

Message 类包含下面几个重要字段：
- `int what` ：用于标识消息类型，以便 Handler 根据消息类型进行相应的处理
- `int arg1`，`int arg2`：两个整型变量，如果 Message 只需要携带1到2个整型数据，应优先通过这两个变量存储
- `Object obj`：携带的一个任意类型的数据
- `long when`：Message 的分发时间（**不是发送时间**）
- `Bundle data`： 携带的一组数据
- `Handler target`：发送该 Message 的 Handler
- `Runnable callback` : 携带的可执行任务
- `Message next`：用于形成链表结构
- `static Message sPool`：用于缓存并复用已使用过的 Message
- `static int sPoolSize = 0`：缓存的 Message 数量  

它还有几个常用的方法：
### Message.Obtain()
![code01](/assets/img/post/code/170921_01.png)  

这个方法的作用就是从 Message 缓存链表中取出第一个 Message，如果缓存池为空，那么就新建一个 Message。 

> 1. 相比 new 一个 Message 对象，通过 obtain() 方法获取 Message 实现了 Message 的复用，成本较低，因此我们应该尽量使用这种方式来获取 Message  
> 2. 这里构造的缓存链表使用了 Message 的 next 字段 
> 3. 还有一种获取 Message 的方法是通过 Handler 实例的 obtainMessage 方法（后面会涉及），这两种都是获取 Message 的最佳做法   

obtain 方法还有几个重载方法，就是将 Message 对应的参数初始化，这里就不再赘述了，Message 还有下面两个用于设置数据的方法：
### Message#setData(Bundle data) 
![code02](/assets/img/post/code/170921_02.png) 

### Message#peekData()
![code03](/assets/img/post/code/170921_03.png) 

对于 Message ，了解这些就足够了，接下来看看 MessageQueue。
## MessageQueue
顾名思义，MessageQueue 就是指消息队列，但它并不是严格的队列，因为它不符合 FIFO 原则。这主要是因为 MessageQueue 中的 Message 是按照**分发时间的早晚进行排序的，分发越早，位置越靠前。** MessageQueue 采用单链表的形式来保存所有未处理的 Message。   

我们只需要关注它的一个字段：  
![code04](/assets/img/post/code/170921_04.png)

前面提到 Message 时说了，Message 有一个 Message 类型的 next 字段，可以用来构成单链表，这个 mMessages 就是链表的头指针。

MessageQueue 有一个构造方法：
### MessageQueue(boolean quitAllowed)
![code05](/assets/img/post/code/170921_05.png)

其中 quiteAllowed 表示该 MessageQueue 是否允许通过 quit 方法直接终止。
> MessageQueue 有一个 quit(boolean safe) 方法用于终止消息的接收，由参数 safe 指定是否安全退出。非安全退出直接移除消息队列中所有消息并终止，安全退出只移除分发时间在 quit() 调用之后的所有消息，等处理完剩余消息后再退出，具体源码就不展示了。

对于 MessageQueue，需要关注的方法只有两个：enqueueMessage 和 next.   

next 用于获取消息队列中的下一个 Message，**可能会阻塞**；enqueueMessage 用来将 Message 插入到 MessageQueue 的适当位置，先来看 enqueueMessage。
### MessageQueue#enqueueMessage
![code06](/assets/img/post/code/170921_06.png)

enqueueMessage 方法就是将 msg 按照分发时间升序插入到消息队列中。
> 这里我只保留了核心代码，如果有兴趣可以自行研究其他代码。  

### MessageQueue#next()
![code07](/assets/img/post/code/170921_07.png)

next 方法的逻辑就是获取 MessageQueue 中下一条消息，**如果该消息的分发时间(对应 Message 的 when 字段)比当前时间（对应方法中的 now）晚，就设定一个等待时间（这个等待时间是消息分发时间与当前时间之差），然后阻塞线程，直到当前时间消息分发时间到了之后，再取出消息并返回。**

> 1. 这里省略了一些 native 层的方法调用以及日志信息  
> 2. 方法中出现的 nativePollOnce 是一个 native 方法，用于使线程等待一段时间，关于这个方法以及 next() 中其他 native 层逻辑更深入的探讨可以参考[这篇博客](http://blog.csdn.net/luoshengyang/article/details/6817933) 

下面要分析的是 Looper。  
## Looper  
对于主线程之外的其他线程，是不能直接创建 Handler 的，而是要像下面这样先通过 Looper.prepare() 为该线程创建 Looper：  
![code08](/assets/img/post/code/170921_08.png)

否则，程序就会抛出如下异常： 

![code09](/assets/img/post/code/170921_09.png)   

至于为什么会有这个异常，等一下就清楚了，现在先来了解一下 Looper。  

Looper 类有几个重要的字段：
- `final MessageQueue mQueue`：保存所有 Message 
- `final Thread mThread`：创建 Looper 的线程
- `static final ThreadLocal<Looper> sThreadLocal`：通过 ThreadLocal 来实现线程隔离的 Looper   
> 如果不熟悉 ThreadLocal 可以参考我的[这篇文章](/inside-thread-local)    

上面代码涉及了 prepare 和 loop 两个方法，先看下 prepare 方法：
### Looper.prepare()
![code10](/assets/img/post/code/170921_10.png) 

方法内调用了下面这个私有静态方法：
### Looper.prepare(boolean quitAllowed)
![code11](/assets/img/post/code/170921_11.png)   

prepare 方法就是通过 ThreadLocal 的 set 方法给当前线程设置 Looper。 

>通过异常信息也可以知道，一个线程只能拥有一个 Looper，相应的也就只有一个 MessageQueue.  

来看一下 Looper 的构造方法：
### Looper(boolean quitAllowed)
![code12](/assets/img/post/code/170921_12.png)  


在构造方法中对 MessageQueue 进行了初始化，接下来看看 loop 方法：
### Looper.loop() 
![code13](/assets/img/post/code/170921_13.png)

省略了一些非核心代码，loop 的逻辑是在循环内通过 MessageQueue 的 next 方法获取下一条消息（通过上面对 MessageQueue 的分析我们知道，这时可能会阻塞），获取消息后，调用 msg.target.dispatchMessage(msg) 来处理消息。   

刚才分析 Message 源码时提到了它有一个 Handler 类型的 target 字段，这里调用的就是发送该消息的 Handler 的 dispatchMessage 方法。  

刚才还涉及到了 Looper 的另一个方法：
### Looper.myLooper()
![code14](/assets/img/post/code/170921_14.png)

就是通过 ThreadLocal 的 get 方法来获取当前线程对应的 Looper。  

终于轮到要 Handler 了。   
## Handler
之所以把 Handler 放到最后才介绍，是因为有了前面的介绍，Handler 的原理理解起来将会非常容易。    

我们对 Handler 最熟悉了，经常通过它的一系列 sendXxx 方法来发送消息，并重写它的 handleMessage 方法来进行消息处理，下面就来看看调用这些方法后到底分发了什么。   

Handler 的字段很少，我们用到的只有下面几个：
- `final Looper mLooper`：对应线程的 Looper
- `final MessageQueue mQueue`：对应线程的 MessageQueue
- `final Callback mCallback`：Callback 是一个接口，只有一个 handleMessage 方法，Handler 有一个含有 Callback 参数的构造方法，这样可以不声明 Handler 子类来创建 Handler ，比如像下面这样：
![code15](/assets/img/post/code/170921_15.png)

> Handler 还有一个重要的字段是 Messenger，主要用于跨进程通信，这里就不介绍了。  

Handler 有两种构造方法，一种含 Looper 参数，一种不含的，分别来看一个。   
### Handler(Looper looper, Callback callback, boolean async)
![code16](/assets/img/post/code/170921_16.png)

带 Looper 参数的构造方法只是给字段赋值。  
### Handler(Callback callback, boolean async)
![code17](/assets/img/post/code/170921_17.png)
 
这个构造方法中通过 Looper.myLooper() 获取当前线程的 Looper 对象，然后初始化其他字段，注意 mQueue 是使用 looper.mQueue 进行初始化的。   

注意，如果 mLooper 为 null，就会抛出异常，这个异常就是我们刚才看到的那个异常，原因就是没有在线程内调用 Looper.prepare()，从而 mLooper 为 null。 

> 值得注意的是方法前半部分对 Handler 的声明做了检查，如果声明方式是非静态内部类，那么 Android Studio 会警告我们可能分发内存泄漏，原因及处置方式可以参考我的[这篇博客](/use-handler-correctly)。  

Handler 的其他方法也不是很多，这里详细介绍下。   

首先是一系列 send 方法：
- sendEmptyMessage(int what)
- sendEmptyMessageDelayed(int what, long delayMillis)
- sendMessage(Message msg)
- sendMessageDelayed(Message msg, long delayMillis)
- sendEmptyMessageAtTime(int what, long uptimeMillis)
- sendMessageAtTime(Message msg, long uptimeMillis)
- sendMessageAtFrontOfQueue(Message msg)   

上面的方法通过名称和参数就可以知道它们的作用，而且又是开发中常用的方法，就不一一分析了。   

前五个方法最终都会调用 sendMessageAtTime(Message msg, long uptimeMillis)，所以只需要看下它的源码就可以了。 
### Handler#sendMessageAtTime(Message msg, long uptimeMillis) 
![code18](/assets/img/post/code/170921_18.png)

可以看到当 MessageQueue 不为空时，调用了 enqueueMessage，其中内容如下：
### Handler#enqueueMessage 
![code19](/assets/img/post/code/170921_19.png)

方法中将当前的 Handler 赋值给 msg 的 target 字段，然后调用了 MessageQueue 的 enqueueMessage() 将 Message 插入到 MessageQueue 的适当位置。  

上面还有一个 sendMessageAtFrontOfQueue 方法。
### Handler#sendMessageAtFrontOfQueue
![code20](/assets/img/post/code/170921_20.png)

这个方法同样调用了 enqueueMessage ,只是第三个参数为0，通过刚才分析 enqueueMessage 的源码，我们现在知道它的作用是将 Message 放到 MessageQueue 的队头。  

除了 send 系列，Handler 还有 post 系列方法：
- post(Runnable r)
- postAtFrontOfQueue(Runnable r)
- postAtTime(Runnable r, long uptimeMillis)
- postAtTime(Runnable r, Object token, long uptimeMillis)
- postDelayed(Runnable r, long delayMillis)   

类比 send 系列方法，就知道这些方法是怎么一回事了，不过还是来看一个。
### Handler#postAtTime
![code21](/assets/img/post/code/170921_21.png)

可以看到，post 还是通过 send 来发送消息的，来看一眼 getPostMessage.
### Handler.getPostMessage
![code22](/assets/img/post/code/170921_22.png)

通过 obtain 方法获取一个 Message 对象，然后为它的 callback 字段赋值。

除此之外，Handler 还有一系列 obtain 方法，用于复用已经处理过的 Message 对象：
- obtainMessage()
- obtainMessage(int what)
- obtainMessage(int what, int arg1, int arg2)
- obtainMessage(int what, int arg1, int arg2, Object obj)
- obtainMessage(int what, Object obj)
这些方法都是通过刚才介绍的 Message 的 obtain 方法来完成的，比如： 

### Handler#obtainMessage(int what, Object obj)
![code23](/assets/img/post/code/170921_23.png)

其他类似。

**接下来是一个非常重要的方法：**
### Handler#dispatchMessage(Message msg) 

![code24](/assets/img/post/code/170921_24.png)

这就是在 Looper 的 loop 方法中，通过 msg.targe.dispatchMessage 调用的方法，是 Handler 对 Message 进行处理的逻辑。其中，handleMessage 以及 callback 的 HandleMessage 是我们创建 Handler 时重写的方法，所以只需要看一下 handleCallback 方法.
### Handler#handleCallback(Message message)

![code25](/assets/img/post/code/170921_25.png)

很简单，直接 run。   

上面对消息处理的过程可以表示为下面这个流程图：  

![Handle-Message-Flow](/assets/img/post/handle-message-flow.png)

到此，Android 消息机制涉及到的核心源码就分析完成了，不妨结合最开始的流程图再来梳理一下。
## 回顾 
Android 的消息机制主要涉及四个类：Message、Looper、MessageQueue 和 Handler。要启动消息机制，需要在线程内通过 Looper.prepare() 初始化当前线程的 Looper 和 MessageQueue，然后创建 Handler ，最后通过 Looper.loop() 启动消息机制。   

消息机制主要涉及下面几个过程：   

- Handler 通过 send/post 等方法间接调用 MessageQueue 的 enqueueMessage 来发送 Message
- Looper 不断调用 MessageQueue 的 next 方法从 MessageQueue 中读取下一条消息（在消息分发时间比当前时间晚时，会分发阻塞），并通过 msg.target.dispatchMessage 方法通知对应的 Handler 进行消息处理；
- Handler 在 dispatchMessage 方法中对消息进行处理  

根据概述里的流程图以及涉及的方法，可以将消息机制整理成下图：
![handler-with-method](/assets/img/post/handler-with-method.png)

>1. 图中创建 Looper、创建 Handler 以及获取消息的线程就是所谓的 Looper 线程，发送消息既可以在 Looper 线程里，也可以在其他线程里 
>2. 一个 Looper 线程只能有一个 Looper，对应一个 MessageQueue，但是可以有多个 Handler    

## 主线程的消息循环
刚才介绍 Looper 的时候提到，若想在线程中使用 Handler，必须先通过通过 Looper.prepare() 初始化当前线程的 Looper 和 MessageQueue，然后创建 Handler ，最后通过 Looper.loop() 启动消息机制，否则会出现异常。  

但我们平时在主线程中就是直接创建 Handler，并没有调用 Looper.prepare() 和 Looper.loop()，但是并没有出现异常。  

原因很简单，一定是系统帮我们调用了这些方法，如果想知道这些方法是在哪被调用的，就要来看 `ActivityThread` 的源码了。
### AcivityThread.main(String[] args)

![code26](/assets/img/post/code/170921_26.png)

可以看到在 main 方法中，也就是主线程的入口，调用了 Looper.prepareMainLooper()，最后调用了 Looper.loop() 方法开始消息循环。  

再来看下之前没提到的 Looper.prepareMainLooper() 方法：
### Looper.prepareMainLooper()
![code27](/assets/img/post/code/170921_27.png)

可以看到同样是调用了 prepare(boolean quitAllowed) 方法来为线程初始化 Looper，与 prepare() 不同的是，这是传入的参数值是 false，通过上面对 Looper 和 MessageQueue 分析可以得知，主线程的 Looper 是不允许开发者调用 quit 方法直接终止的。
## 总结
这应该是目前我的所有文章里写的最长的了。   

虽然感觉对消息机制的介绍还是不太全面，但已经涵盖了消息机制的大部分内容了，native 层的还没接触，所以暂时先放一放吧。  

通过这次整理，我对 Android 消息机制的认识更加深刻和全面了，希望读到这篇文章的你也是如此。
## Refs
- 《[Android 开发艺术探索](https://item.jd.com/11760209.html)》 第十章