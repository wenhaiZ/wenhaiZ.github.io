---
layout: post
title: "[译] Dart 异步编程：隔离与事件循环"
date: 2020-01-19 15:20:00 +0800
tags: [Code,Flutter]
subtitle: "快速理解 Dart 的异步机制"
published: true
---
Dart 虽然是一门单线程的编程语言，但是也提供了 future、stream、后台任务等一系列可以用来编写现代化的、基于响应式的异步程序的工具。本文主要讲述 Dart 对异步任务支持的基础：隔离(Isolates)和事件循环(event loops)。  

如果你更喜欢通过听和看的方式学习，本文的所有内容都包含在[下面链接的视频](https://www.youtube.com/watch?v=vl_AaCgudcY)里。这个视频来自[聚焦 Flutter](https://www.youtube.com/playlist?list=PLjxrf2q8roU2HdJQDjJzOeO6J3FoFLWr2) 系列中 Dart 异步编程部分。  

让我们先来看看隔离。

## 隔离（Isolates）
一个隔离就是所有 Dart 代码运行的地方，它就像是机器上一个拥有私有内存块和一个运行事件循环的线程的独立空间。 

![隔离拥有独立的内存空间和一个运行事件循环的线程](/assets/img/post/isolate.png)

在很多其他语言（例如 C++）中，,你可以拥有很多线程，它们共享内存空间，可以分别执行任意代码。但在 Dart 中，每个线程都在它自己的隔离内，有自己的内存空间，这个线程只是处理各种事件（稍后会介绍）。

许多 Dart 应用都在一个隔离内运行它们所有的代码，但是如果需要的话，你可以拥有不止一个隔离区。  
如果你有一个工作量很大的运算需要执行，它在主要隔离区中运行可能导致丢帧，那么你可以使用 [Isolate.spawn()](https://api.dartlang.org/stable/dart-isolate/Isolate/spawn.html) 或者 [Flutter 的 compute()](https://flutter.dev/docs/cookbook/networking/background-parsing#4-move-this-work-to-a-separate-isolate) 这两个函数，这两个函数都会创建一个单独的隔离来执行运算，从而释放主隔离使其可以重建和渲染控件树。    

![](/assets/img/post/two_isolates.png) 

新创建的隔离拥有自己的事件循环和内存，原来的隔离不能访问它们，虽然新的隔离是由原来的隔离创建的——这就是隔离这个名字的来源：这些空间是彼此隔离开来的。

事实上，多个隔离能够一起工作的唯一方式就是互相传递消息。一个隔离区向另一个发送消息，收到消息的一方通过自己的事件循环来处理它。  

缺少共享内存可能显得有点矫枉过正，特别是当你已经熟悉 Java 或者 C++ 时，但是这样对于 Dart 编程者来说拥有重要的优势。

比如，一个隔离内的内存分配和垃圾回收不需要加锁。因为只有一个线程，如果它处于空闲，那么内存状态是不会改变的。这对于 Flutter 应用来说非常有用，因为通常需要短时间内创建和销毁大量的控件（Widget）。

## 事件循环
现在你对隔离已经有了基本的了解，接下来让我们研究真正让异步编程成为可能的东西：事件循环(event loops)。

你可以把应用的生命想象成一个时间线。应用启动、停止，在这期间有很多事件发生——比如磁盘IO、用户点击等等。 

你的应用不能预测这些事件发生的时间以及它们发生的顺序，它只能在一个永远不会阻塞的线程里去处理这些事件。因此，应用会运行一个事件循环，它从事件队列中取出最早的一个事件，处理它，然后再取一个事件，再处理它...直到事件队列为空为止。  

在应用的整个运行期间——你点击屏幕、下载文件、计时器结束——这个事件循环一直运行，然后一个一个的处理这些事件。  

![](/assets/img/post/event_loops.png)  

当事件循环过程中有间断（事件队列为空）时，线程会挂起，然后等待下一个事件到来。它还可以选择触发垃圾回收、喝杯咖啡或者做其他事。  


Dart 为异步编程设计的高层次 API 或者语言特性——future/stream/async/await——都是基于这个简单的事件循环构建的。   

举个例子，假如你有一个按钮用于发起一个网络请求，向下面这样：

```dart
RaisedButton(
  child: Text('Click me'),
  onPressed: () {
    final myFuture = http.get('https://example.com');
    myFuture.then((response) {
      if (response.statusCode == 200) {
        print('Success!');
      }
    });
  },
)
```
当你运行应用时， Flutter 构建这个按钮并显示在屏幕上，然后你的应用就开始等待。应用的事件循环可能有点闲，它在等待下一个事件。在按钮等待用户点击它时，和按钮不相关的事件可能会出现然后被处理。当用户终于点击按钮后，一个点击事件进入了事件队列。  

然后这个事件被取出，Flutter 看着这个事件，渲染系统说：“点击的坐标与按钮相匹配”，因此 Flutter 执行了按钮的 onPressed 函数，在函数中发起了一个网络请求（返回一个 Future 对象），并且通过 then() 方法给 future 注册了一个完成后的处理程序。 

然后就结束了。事件循环完成了对点击事件的处理，然后事件会被清除。  

在这里，onPressed 是 [RaisedButton] 的一个属性，网络请求使用了一个设置给 [future](https://api.dart.dev/stable/dart-async/Future-class.html)的回调，但是这两种方式做的是同一件事，它们都是在告诉 Flutter：“嘿，一会儿你可能会收到某种类型的事件，当你收到它时，请执行这些代码。”  

所以，onPressed 正在等待一个点击，future 正在等待网络事件，但是对于 Dart 来说，它们都只是队列中的事件。  


这就是异步编程在 Dart 中的实现方式。Future 、Stream 、async、await等这些都是在告诉 Dart 的事件循环：“这里有一些代码，请待会儿执行它”。

如果我们回看刚才的例子，你就可以清晰的看出它是怎样根据事件被分解不同的块的，它们是控件构建(1)，点击事件(2)和网络请求响应事件(3)。

```dart
RaisedButton( // (1)
  child: Text('Click me'),
  onPressed: () { // (2)
    final myFuture = http.get('https://example.com');
    myFuture.then((response) { // (3)
      if (response.statusCode == 200) {
        print('Success!');
      }
    });
  },
)
```  

等你习惯使用异步代码后，你就会发现所有的地方都在使用这种模式。理解了事件循环也将会对学习高层次的 API 有所帮助。 

## 总结

我们快速浏览了隔离、事件循环以及Dart异步编程的基础。  

如果想了解更多，可以观看 Dart 异步编程系列的[下一个视频](https://youtu.be/OTS-ap9_aXc)，它讲解了 Future API，你可以用它来轻松进行异步编程。  

感谢 Andrew Brogdon，他创建了这篇文章所基于的视频。








