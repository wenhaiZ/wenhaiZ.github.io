---
layout: post
title: "[译]在 Flutter 中如何设计重叠布局?"
date: 2019-10-03 19:50:00 +0800
tags: [Code,Flutter]
subtitle: "Stack 及其辅助控件"
published: true
---
>**原文作者：Burhanuddin Rashid，Google 认证 Android 开发者**

> 原文链接：[Flutter For Android Developers : How to design FrameLayout in Flutter.](https://medium.com/flutter-community/flutter-for-android-developers-how-to-design-framelayout-in-flutter-93a19fc7e7a6)

-----------------
这篇博客是为那些想把现有的知识应用到 Flutter 的 Android 开发者写的。在这篇文章里，我们将会探索在 Flutter 中与 `FrameLayout` 对应的是什么。 

## 博客系列
- [在 Flutter 中如何设计 Activity 界面](https://wenhaiz.github.io/how-to-design-activity-ui-in-flutter)
- [在 Flutter 中如何设计 LinearLayout](https://wenhaiz.github.io/how-to-design-linear-layout-in-flutter)
- 在 Flutter 中如何设计 FrameLayout (本篇)


## 先决条件
这篇文章假设你已经在电脑上配置好了 `Flutter` 的运行环境并且能成功运行一个 Hello World 项目。如果你还没有安装 Flutter，你可以从[这里开始](https://flutter.dev/docs/get-started/install)。

`Dart` 是一门基于面向对象的语言，对一个 Android 开发者来说是很容易掌握的。

## 让我们开始吧

`FrameLayout` 是 Android 开发中比较常用的布局。我们声明一个`FrameLayout`然后在其中添加一个或多个子控件，
它们会以栈的顺序绘制（最后添加的出现在最上面）。  

下图展示了我们在 Android 中的做法：  

![framelayout_stack](/assets/img/post/frame_layout_stack.jpeg)   
>https://stackoverflow.com/questions/25679369/what-does-framelayout-do   

在 Android 开发中，`FrameLayout`主要用于两种情况：   
1. 绘制一个在其他控件之上的控件，比如以 **栈(Stack)** 的形式去承载控件（最后添加的在最上面）  
2. 用作加载 `Fragment` 的容器    

第二个原因适用于 Android，但是在 Flutter 中，**一切都是部件(Everything is a widget)**，所以并没有类似`Fragment`的概念，我们使用部件来代替。   

第一种情况在设计中是很常见的，所以 Flutter 提供了一个与`FrameLayout`行为一致的部件。   
是的，它就是`Stack`,我在第一种情况中已经用粗体标明了栈这个词。   


## Stack  
[Stack](https://docs.flutter.io/flutter/widgets/Stack-class.html) 基于自身边缘来定位子控件，它和`FrameLayout`相同。当你想简单的把一系列控件交叠时它会非常好用，比如有一些文字和一张图片，然后上面覆盖渐变色，底部有个按钮。   

我们可以像下面代码这样声明 `Stack`:
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

正如我们知道的那样，`FrameLayout`的子控件以栈的形式绘制，它们出现的顺序取决于代码中定义的顺序。  
第一个声明的控件将会出现在最下面，而最后声明的控件将会出现在最上层。   
  

Stack 也是如此。`children:<Widget>[]`中声明的第一个子控件将会出现在最下层，最后一个声明的子控件将会出现在最顶层。   

### 1. android:gravity  

由于`FrameLayout`并没有这个属性，因此需要给每个子控件声明`layout_gravity`属性以达到和`android:gravity`同样的行为。    

相对而言，Stack 就更方便了一步，它内置了`Stack.alignment`属性以达到`android:gravity`的行为。  
`Stack.alignment`属性接受值类型为 `AlignmentDirectional`枚举，例如`AlignmentDirectional.topStart`和`AlignmentDirectional.center`等。    

由于`android:gravity`是在父布局中声明的，因此我们也在父布局中声明`Stack.alignment`这个属性，也就是在`Stack`中。   

你可以像下面这样声明：  

```dart
child: Stack(
  alignment: AlignmentDirectional.center,
  children: [
   ...//all your child widgets
  ],
)
```   

如果没有声明`alignment`属性，那么它的默认值将会是`lignmentDirectional.topStart`。   
你可以参考上面的截图，我们没有声明`alignment`属性时，Stack 里面的内容出现在了左上角。   

Flutter 的一大好处就是，你通过类的命名就知道它的含义。通过`alignment`属性值的名字，你就知道即将会发生什么。 

比如`AlignmentDirectional.topStart`就是把子控件都放在布局的左上角。  
下面展示了`AlignmentDirectional`其他值得情况。    

![](/assets/img/post/alignment_top.png)
![](/assets/img/post/alignment_center.png)
![](/assets/img/post/alignment_bottom.png)  


####  注意：
1. 如果我们没有为Stack 指定大小，那么它将具有与父布局同样的大小(`match_parent`)。在上面的例子中，我们通过`BoxConstraints.expand()`把
`Container`设置成可扩展的，因此它将占据所有的可用空间（在我们的例子中就是整个屏幕，我们可以通过蓝绿色来辨认）。
由于没有指定大小，因此`Stack.alignment`属性将依照`Container`的大小来生效。 
2. 如果Stack 的父布局没有指定大小，那么 Stack 将和最大子控件的大小一致(`wrap_content`)。下图展示了移除`Container`的 `BoxConstraints.expand()`之后的效果。  
![](/assets/img/post/alignment_wrap.png)      


### 2. android:layout_gravity   
`android:layout_gravity`是控件的外部重力，表明了控件应该挨着父布局的那个边，在`FrameLayout`我们经常这么使用。 

为了在`Stack`中达到同样的效果，我们要用`Align`控件将真正的子控件包裹起来，如下面代码所示：
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
            alignment: AlignmentDirectional.center,
            children: [
              Align(
                child: Container(
                  height: 200.0,
                  width: 200.0,
                  color: Colors.red,
                ),
                alignment: AlignmentDirectional.topStart,
              ),
              Align(
                child:Container(
                  height: 150.0,
                  width: 150.0,
                  color: Colors.blue,
                ),
                alignment: AlignmentDirectional.topEnd,
              ),
              Align(
                child: Container(
                  height: 100.0,
                  width: 100.0,
                  color: Colors.green,
                ),
                  alignment: AlignmentDirectional.bottomStart
              ),
              Align(
                child:Container(
                  height: 50.0,
                  width: 50.0,
                  color: Colors.yellow,
                ),
                  alignment: AlignmentDirectional.bottomEnd
              )
            ],
          ),
        ),
      ),
    );

  }
}

```  

![](/assets/img/post/stack_layout_gravity.png)   

在上面的代码中，我们使用`Align.alignment`属性将子控件对齐到了四个角上。
如果使用`Align`时我们没有指定`Stack`的大小，那么 Stack 将会占用所有的可用空间并在此空间中对齐子控件
（在我们的例子中是整个屏幕），这就是为什么没有指定`BoxConstraints.expand()`依然可以看到全屏的浅绿色。   

让我们在绿色盒子上试验一下`Align.alignment`不同取值的情况：   

![](/assets/img/post/align_top.png)   

![](/assets/img/post/align_center.png)  

![](/assets/img/post/align_bottom.png)   


### 3.Positioned  
`Positioned`是一个只能在 Stack 中使用的额外控件，现在`FrameLayout`中并没有定位行为。  
[Positioned](https://docs.flutter.io/flutter/widgets/Positioned-class.html) 就是一个控制 Stack 子控件如何定位的控件。  

想定义 Stack 子控件的定位，我们需要用 `Positioned`来包裹这个子控件，然后根据需要声明`top`、`bottom`、`left`和 `right`属性。

下面是示例代码：
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
            alignment: AlignmentDirectional.center,
            children: [
              Positioned(
                child: Container(
                  height: 200.0,
                  width: 200.0,
                  color: Colors.red,
                ),
                top: 10.0,
                left: 10.0,
              ),
              Positioned(
                child: Container(
                  height: 150.0,
                  width: 150.0,
                  color: Colors.blue,
                ),
                top: 30.0,
                right: 50.0,
              ),
              Positioned(
                child: Container(
                  height: 100.0,
                  width: 100.0,
                  color: Colors.green,
                ),
                bottom: 100.0,
                left: 30.0,
              ),
              Positioned(
                child: Container(
                  height: 50.0,
                  width: 50.0,
                  color: Colors.yellow,
                ),
                bottom: 50.0,
                right: 100.0,
              )
            ],
          ),
        ),
      ),
    );

  }
}

```
![](/assets/img/post/stack_positioned.png)  

从上面的图可以看到控件被定位到了指定位置，你还可以为`Positioned`设置宽度和高度。  


### 总结  

`FrameLayout`在 Android 中是很常用的，在 Flutter 中使用 `Stack`以及`Positioned`等控件可以很容易的实现同样的效果。   


希望可以在接下来的博客中讨论更多内容，我创建了一个样例 app 来对照`FrameLayout`演示 `Stack` 的属性,
你可以点击工具栏查看`Positioned`控件的输出结果。  
你可以在[这里](https://github.com/burhanrashid52/FlutterForAndroidExample)查看这个样例。     

![](/assets/img/post/stack_demo.gif)

    


感谢阅读。





