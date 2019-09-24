---
layout: post
title: "[译]在 Flutter 中如何设计重叠布局?"
date: 2019-09-30 11:17:00 +0800
tags: [Code,Flutter]
subtitle: "Stack"
published: false
---
>**原文作者：Burhanuddin Rashid，Google 认证 Android 开发者**

> 原文链接：[Flutter For Android Developers : How to design FrameLayout in Flutter.](https://medium.com/flutter-community/flutter-for-android-developers-how-to-design-framelayout-in-flutter-93a19fc7e7a6)

-----------------
这篇博客是为那些想把现有的开发知识应用到 Flutter 的 Android 开发者写的。在这篇文章里，我们将会探索在 Flutter 中与 `FrameLayout` 对应的是什么。 

## 博客系列
- [在 Flutter 中如何设计 Activity 界面](https://wenhaiz.github.io/how-to-design-activity-ui-in-flutter)
- 在 Flutter 中如何设计 LinearLayout (本篇)
- [在 Flutter 中如何设计 FrameLayout](https://medium.com/@burhanrashid52/flutter-for-android-developers-how-to-design-framelayout-in-flutter-93a19fc7e7a6)


## 先决条件
这篇文章假设你已经在电脑上配置好了 Flutter 的运行环境并且能成功运行一个 Hello World 项目。如果你还没有安装 Flutter，你可以从[这里开始](https://flutter.dev/docs/get-started/install)。

`Dart` 是一门基于面向对象的语言，对一个 Android 开发者来说是很容易掌握的。

## 让我们开始吧

`FrameLayout`是 Android 开发中比较常用的布局。我们声明一个`FrameLayout`然后在其中添加一个或多个子控件，它们会以栈的顺序绘制（最后添加的在最上面）。  

下图展示了我们再 Android 中的做法：
![framelayout_stack](/assets/img/post/frame_layout_stack.jpeg)   
>https://stackoverflow.com/questions/25679369/what-does-framelayout-do   

在 Android 开发中，`FrameLayout`主要用于两种情况：   
1. 绘制一个在其他控件之上的控件，比如以**栈(Stack)**的形式去负载控件（最后添加的在最上面）  
2. 用作加载 `Fragment` 的容器    

第二个原因适用于 Android，但是在 Flutter 中，**一切都是部件(Everything is a widget)**，所以并没有类似`Fragment`的概念，我们使用部件来代替。   

第一种情况在设计中是很常见的，所以Flutter 提供了一个与`FrameLayout`行为一致的部件。是的，它就是`Stack`,我在第一种情况中已经用粗体标明了栈这个词。   


## Stack  
[Stack](https://docs.flutter.io/flutter/widgets/Stack-class.html) 基于自身边缘来定位子控件，它和`FrameLayout`相同。当你想简单的把一系列控件交叠时它会非常好用，比如有一些文字和一张图片，然后上面覆盖渐变色，底部有个按钮。   

我们可以像下面代码这样声明`Stack`:
```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatefulWidget {
  
  @override
  State<StatefulWidget> createState() {
    return _MyAppState();
  }
}


class _MyAppState extends State<MyApp> {

  @override
  Widget build(BuildContext context) {

    return MaterialApp(

      home: Scaffold(
        appBar: AppBar(
          title: Text("FrameLayout"),
        ),
        body: Container(
          constraints: BoxConstraints.expand(),
          color: Colors.tealAccent,
          child: Stack(
            children: [
              Container(
                height: 200.0,
                width: 200.0,
                color: Colors.red,
              ),
              Container(
                height: 150.0,
                width: 150.0,
                color: Colors.blue,
              ),
              Container(
                height: 100.0,
                width: 100.0,
                color: Colors.green,
              ),
              Container(
                height: 50.0,
                width: 50.0,
                color: Colors.yellow,
              ),
            ],
          ),
        ),
      ),
    );

  }
}

```   

下图是上面代码的输出：
![](/assets/img/post/stack_output1.png)  

正如我们知道的那样，`FrameLayout`的子控件以栈的形式绘制，它们出现的顺序取决于定义的顺序。  
第一个声明的控件将会出现在最下面，而最后声明的控件将会出现在最上层。   
  

Stack 也是如此。`children:<Widget>[]`中声明的第一个子控件将会出现在最下层，最后一个声明的子控件将会出现在最顶层。   

### 1. android:gravity





