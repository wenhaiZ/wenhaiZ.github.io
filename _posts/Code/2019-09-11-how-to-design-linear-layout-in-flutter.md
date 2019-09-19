---
layout: post
title: "[译]在 Flutter 中如何设计线性布局?"
date: 2019-09-06 10:51:00 +0800
tags: [Code,Flutter]
subtitle: "答案是... Scaffold"
published: false
---
**作者：Burhanuddin Rashid，Google 认证 Android 开发者**

> 原文链接：[Flutter For Android Developers : How to design LinearLayout in Flutter.](https://proandroiddev.com/flutter-for-android-developers-how-to-design-linearlayout-in-flutter-5d819c0ddf1a)

-----------------
这篇博客是为那些想把现有的开发知识应用 Flutter 的 Android 开发者写的。在这篇文章里，我们将会探索在 Flutter 里与 `LinearLayout` 对应的是什么。 

## 博客系列
- [在 Flutter 中如何设计 Activity 界面](https://wenhaiz.github.io/how-to-design-activity-ui-in-flutter)
- 在 Flutter 中如何设计 LinearLayout (本篇)
- [在 Flutter 中如何设计 FrameLayout](https://medium.com/@burhanrashid52/flutter-for-android-developers-how-to-design-framelayout-in-flutter-93a19fc7e7a6)


## 先决条件
这篇文章假设你已经在电脑上配置好了 Flutter 的运行环境并且能成功运行一个 Hello World 项目。如果你还没有安装 Flutter，你可以从[这里开始](https://flutter.dev/docs/get-started/install)。

`Dart` 是一门基于面向对象的语言，对一个 Android 开发者来说是很容易掌握的。

## 让我们开始吧 
如果你是一个 Android 开发者，我想你应该会在设计布局时重度使用`LinearLayout`。对于不熟悉`LinearLayout`的人，我先从[官方定义](https://developer.android.com/reference/android/widget/LinearLayout)开始吧。
>线性布局就是一个把其他视图组件水平排成一行或垂直排成一列的布局。

![线性布局](/assets/img/post/linear_layout.jpeg)

有了上述定义和图片演示，你就可以知道在 Flutter 中与 LinearLayout 相同的部件。没错，它们就是`Row`和`Column`。这两个部件的行为几乎和 Android 原生的 LinearLayout 一模一样。Row 和 Column 在 Flutter 中也是被重度使用的部件。  

>注意：Row/Column 不会滚动。如果你有一些线性排列的控件并且希望在空间不够时它们可以滚动，可以考虑使用[ListView](https://api.flutter.dev/flutter/widgets/ListView-class.html)。   

现在我们将会讨论 LinearLayout 的一些主要属性，这些属性在 Flutter 中有对应的部件属性。

## 1.方向（Orientation）
在 LinearLayout 中你可以通过`android:orientation=”horizontal”`这个属性来指定子控件的排列方向。这个属性接收`horizontal/vertical`两个值，对应 Flutter 中的 Row/Column。   

在 Android 中，LinearLayout 是一个`ViewGroup`，它可以将其他的控件作为子控件。你可以在 `<LinearLayout> </LinearLayout>`标签内声明它所有的子控件。  

为了声明`Row/Column`的子控件，我们需要用到 `Row/Column`的`children`属性，这个属性接收一个控件列表(`List<Widget>`)。如下面代码所示：     


```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatefulWidget {
  @override
  _MyAppState createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  Widget build(BuildContext context) {
    return  MaterialApp(
      home:  Scaffold(
        appBar:  AppBar(
          title:  Text("LinearLayout Example"),
        ),
        body:  Container(
          color: Colors.yellowAccent,
          child:  Row(
            children: [
               Icon(
                Icons.access_time,
                size: 50.0,
              ),
               Icon(
                Icons.pie_chart,
                size: 100.0,
              ),
               Icon(
                Icons.email,
                size: 50.0,
              )
            ],
          ),
        ),
      ),
    );
  }
}

```
在这个例子中，我们使用了`Row`这个控件，相当于设置了`android:orientation=”horizontal”`的 LinearLayout。如果是垂直的情况我们就用`Column`。如果你想知道`Scaffold`在这里的用处，你可以阅读我之前的文章[在 Flutter 中如何设计 Activity 界面?](https://wenhaiz.github.io/how-to-design-activity-ui-in-flutter)。下图是上面代码的输出结果：  

![](/assets/img/post/row_column_result.jpeg)   

|LinearLayout(属性)|值|Flutter 部件|     
|:---:|:----:|:----:|  
|android:orientation|horizontal|Row| 
|android:orientation|vertical|Column|   

## 2."match_parent" 和 "wrap_content"

- `MATCH_PARENT`:即控件想和父控件一样大，如果你的控件是顶级根控件，那么它将和屏幕一样大。
- `WRAP_CONTENT`:即控件的大小刚好够包围它的内容。  

为了获得与`match_parent`和`wrap_content`一样的行为，我们需要用到 Row和 Column 的`mainAxisSize`属性，这个属性接收`MainAxisSize`枚举。`MainAxisSize`有两个值，`MainAxisSize.min`即`wrap_content`,`MainAxisSize.max` 即`match_parent`。  

在上面的例子中，我们并没有给`Row`指定`mainAxisSize`属性，所以它将默认设为`MainAxisSize.max`，即`match_parent`。黄色背景展示了容器的剩余空间是如何被覆盖的。下面代码演示了在上面例子如何指定`mainAxisSize`属性，不同值对应的输出结果如下图所示。
```dart
....
body: Container(
  color: Colors.yellowAccent,
  child: Row(
    mainAxisSize: MainAxisSize.min,
    children: [...],
  ),
)
...
```

![](/assets/img/post/main_axis_size.png)  

这样我们就可以从视觉上区分 `mainAxisSize` 的不同值是如何应用在 Row 和 Column上的。

## 3.重力(Gravity)
重力指定了子控件如何在自身范围内定位其内容，我们在LinearLayout中使用`android:gravity=”center”`属性指定重力，这个属性接收多个标识如何对齐的值。在Row和Column中，我们可以使用`MainAxisAlignment`和`CrossAxisAlignment`达到同样的效果。

### 1.[MainAxisAlignment](https://docs.flutter.io/flutter/rendering/MainAxisAlignment-class.html):  
这个属性指定了子控件在主轴方向上如何被放置。为了让这个属性生效，在Row和Column中必须有剩余空间。如果你将`mainAxisSize`属性设为`MainAxisSize.min`,那么设置`MainAxisAlignment`属性将会没有效果，因为没有剩余空间可用。我们可以像下面这样指定`MainAxisAlignment`属性：
```dart
....
body: Container(
  color: Colors.yellowAccent,
  child: Row(
    mainAxisSize: MainAxisSize.max,
    mainAxisAlignment: MainAxisAlignment.start,
    children: [...],
  ),
)
...
```
>一图胜千言，与其用语言描述每个属性，不如用图片来展示更直观。

下面的输出比较了 `LinearLayout` 的属性和 `Row` 的`MainAxisAlignment`属性。  

![](/assets/img/post/linearlayout_row.png)  

然后我们来比较一下`Column`部件：   

![](/assets/img/post/linearlayout_column.png)   

>**练习**：你可以试下其他的枚举：`spaceEvenly`,`spaceAround`,`spaceBetween`,它们与`ConstraintLayout`中用到了水平/垂直链（chain）行为一致。 



### 2.[CrossAxisAlignment](https://docs.flutter.io/flutter/rendering/CrossAxisAlignment-class.html):
这个属性指定了子控件在交叉轴方向如何放置。即如果我们使用`Row`部件，那么子控件的重力则基于垂直线；如果我们 使用`Column`部件，那么子控件的重力则基于水平线。   

这听起来有点让人费解，不过随着继续阅读你将会理解它。  

为了便于理解 ，我们将`mainAxisSize`设为`MainAxisSize.min`。你可以像下面代码一样声明`CrossAxisAlignment`属性，如果没有设置，那么默认值为`CrossAxisAlignment. start`。  

```dart
....
body: Container(
  color: Colors.yellowAccent,
  child: Row(
    mainAxisSize: MainAxisSize.min,
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [...],
  ),
)
...
```  

下图比较了LinearLayout 的属性和`Row`的`CrossAxisAlignment`属性：  

![](/assets/img/post/linearlayout_row_cross.png)   

下面再来看看`Column`属性：
![](/assets/img/post/linearlayout_column_cross.png) 


`stretch`表现的有点不一样，它把控件拉伸以占满交叉轴剩余的空间（`match_parent`）。     

## 4.布局比重(Layout Weight))


