---
layout: post
title: "[译]在 Flutter 中如何设计 Activity 界面?"
date: 2019-09-06 10:51:00 +0800
tags: [Code,Flutter]
subtitle: "答案是... Scaffold"
---
**作者：Burhanuddin Rashid，Google 认证 Android 开发者**

> 原文链接：[Flutter For Android Developers : How to design Activity UI in Flutter.](https://blog.usejournal.com/flutter-for-android-developers-how-to-design-activity-ui-in-flutter-4bf7b0de1e48)

-----------------

这篇博客是为那些想把现有的开发知识应用到 Flutter 的 Android 开发者写的。在这篇文章里，我们将会探索在 Flutter 里与 Activity 对应的是什么。 

## 博客系列
- 在 Flutter 中如何设计 Activity 界面 (本篇)
- [在 Flutter 中如何设计 LinearLayout](https://medium.com/@burhanrashid52/flutter-for-android-developers-how-to-design-linearlayout-in-flutter-5d819c0ddf1a)
- [在 Flutter 中如何设计 FrameLayout](https://medium.com/@burhanrashid52/flutter-for-android-developers-how-to-design-framelayout-in-flutter-93a19fc7e7a6)


## 先决条件
这篇文章假设你已经在电脑上配置好了 Flutter 的运行环境并且能成功运行一个 Hello World 项目。如果你还没有安装 Flutter，你可以从[这里开始](https://flutter.dev/docs/get-started/install)。

`Dart` 是一门基于面向对象的语言，对一个 Android 开发者来说是很容易掌握的。

## 目标
在这篇文章结束时，我们将使用 Flutter 创建一个如下图所示的 Activity 布局。   

![Goal ui](/assets/img/post/goal_ui.png)  

技术上来说，如果你研究过 Flutter 生成的 Android 项目并且查看过它的 `AndroidMenifest.xml` 文件，你会发现它只运行了一个 Activity，比如 `FlutterActivity`，但这篇文章要探讨的问题是，在 Flutter 里如何设计一个 Activity 的界面？   
答案是...**Scaffold**。

## Scaffold
`Scaffold` 是一个表示 Activity 界面的小部件。作为 Android 开发者，我们使用 Activity 来表示一屏内容，它可以包括顶部工具栏(Toolbar)、菜单(Menus)、侧滑菜单(Drawer)、底部导航栏(BottomNavigationBar)、底部提示(SnackBar)、悬浮按钮(FloatActionButton)等，我们还会用一个`FrameLayout`作为`Fragment`的容器。   
Scaffold 以部件`(Widgets)`的形式包含了上述所有内容。   

>记住，在 Flutter 里，一切皆部件(Widget)。


下面这张图片展示了 Scaffold 的内容组成：它提供了用于展示左右两侧布局的 API，即`DrawerLayout`;BottomBar 就是 Android 中的 `BottomNavigationView`，App bar 就是 `Toolbar`，我们可以把 Content area 当做上面说的`FrameLayout`容器。

![Scaffold](/assets/img/post/scaffold_struct.jpeg)   


由于 Scaffold 是材料部件(Material Widgets)的一部分，因此它需要一个材料App(MaterialApp)作为父容器。   
我们将会在接下来的文章探讨关于 `MaterialApp` 的细节，现在我们先来看看如何创建一个 Scaffold 部件。

```dart
import 'package:flutter/material.dart';

void main() => runApp(MaterialApp(
      home: Scaffold(
      ),
    ));
```

运行上面的代码，你将会看到一个白色的界面，因为我们没有在 Scaffold 里面添加任何东西，让我们通过`backgroundColor`属性来设置一个黄色的背景：

```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        //设置背景色为黄色  
        backgroundColor: Colors.yellowAccent,
      ),
    ));
```
运行代码后你就能看到屏幕上出现黄色的背景了。   
>你可以试着设置其他属性并通过热重载（Hot Reload）来查看运行结果，你也可以在[官方文档](https://api.flutter.dev/flutter/material/Scaffold-class.html)查看Scaffold的所有属性。    

现在你知道如何创建一个 Scaffold 了，我们接下来将会一个一个的探索它的主要属性。

### 1.Appbar(Toolbar)

Appbar 展示的部件和我们在 Activity 中使用的 `Toolbar` 是相同的，下面的图展示了在所用的语言方向是从左到右(例如英语)时，Appbar 的每个属性出现在工具栏的对应位置。   

![Appbar](/assets/img/post/appbar.png)   

- [leading](https://docs.flutter.io/flutter/material/AppBar/leading.html):展示在标题之前的控件，这个控件通常用来展示图标或者后退按钮。
- [title](https://docs.flutter.io/flutter/material/AppBar/title.html):用一个 `Text` 控件来包裹 `Toolbar` 的标题。
- [actions](https://docs.flutter.io/flutter/material/AppBar/actions.html):这和我们使用 `menu.xml` 通过定义`<item/>`来展示菜单是一样的，actions 属性接收一个控件列表来在 Appbar 上展示菜单，这些控件通常是 [IconButton](https://docs.flutter.io/flutter/material/IconButton-class.html)。
- [bottom](https://docs.flutter.io/flutter/material/AppBar/bottom.html):bottom 通常用于在 Appbar 下面展示一个 `TabBar`。
- [flexibleSpace](https://docs.flutter.io/flutter/material/AppBar/flexibleSpace.html): 这个部件通常用于配合 Appbar 创建 `CollapsingToolbarLayout` 效果。  

你可以像下面这样创建一个包含leading、title 和 menus 的 Appbar：  

```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        backgroundColor: Colors.yellowAccent,
        appBar: AppBar(
          leading: Icon(Icons.menu),
          title: Text('My Title'),
          actions: <Widget>[
            IconButton(
              color: Colors.white,
              icon: Icon(
                Icons.shopping_cart,
                color: Colors.white,
              ),
              onPressed: null,
            ),
            IconButton(
              icon: Icon(
                Icons.monetization_on,
                color: Colors.white,
              ),
              onPressed: null,
            )
          ],
        ),
      ),
    ));
```   

下图就是上面代码的运行效果，它看起来和 Activity 的 `Toolbar`非常像。

![ui-appbar](/assets/img/post/appbar_ui.png)
> 你可以试着增删控件，或者为控件提供一个样式或颜色，也可以把探索 Appbar 的其他属性作为一个练习。  

### 2.Body (其他 View 的容器) 
这是 Scaffold 的主要内容区域，可以充当 Fragment 的容器。它接收一个控件并把它展示出来，这就是我们向用户展示主要内容的地方。   
为了简单起见，在这个例子中我们只为 body 添加一个红色背景。在实际使用中，可不止一个背景颜色这么简单，还可以添加 `ListView`、`Row`、`Column`、`Stack`等。  

```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        backgroundColor: Colors.yellowAccent,
        appBar: AppBar(
          leading: Icon(Icons.menu),
          title: Text('My Title'),
          actions: <Widget>[
            IconButton(
              color: Colors.white,
              icon: Icon(
                Icons.shopping_cart,
                color: Colors.white,
              ),
              onPressed: null,
            ),
            IconButton(
              icon: Icon(
                Icons.monetization_on,
                color: Colors.white,
              ),
              onPressed: null,
            )
          ],
        ),
        body: Container(
          //设置红色背景  
          color: Colors.red,
        ),
      ),
    ));
```

![ui-red](/assets/img/post/ui_red_bg.png)

> Body 的属性展示在 Appbar 下方，并且在 floatingActionButton 和 drawer 后面。虽然我们之前给 Scaffold 定义了一个黄色的背景，但是红色背景会覆盖它。

### 3. Drawer (DrawerLayout)
这个控件代表 Android 中的 `DrawerLayout`,它可以从 Activity 边缘水平滑入，来展示应用的导航。 

![drawer](/assets/img/post/drawer.png)

Drawer 通常与 [Scaffold.drawer](https://docs.flutter.io/flutter/material/Scaffold/drawer.html)属性一起使用。就像在 Android 里我们用`NavigationView` 填充 `DrawerLayout`一样，下面的表格展示了在 Android 和 Flutter 中可以用于 Drawer 的等价控件。    

|NavigationView(Android)|Drawer(Flutter)|
|:----|:-------|
|app:headerLayout="@layout/nav_header"|DrawerHeader|
|app:menu="@menu/drawer_view"|ListTile|
|android:layout_gravity="start"|Use Scaffold.drawer property|
|android:layout_gravity="end"|Use Scaffold.endDrawer property|    


Drawer 的子控件通常是一个[ListView](https://docs.flutter.io/flutter/widgets/ListView-class.html),它的第一个子控件是[DrawerHeader](https://docs.flutter.io/flutter/material/DrawerHeader-class.html)，用来展示当前用户的状态信息。Drawer的其他子控件通常使用[ListTile](https://docs.flutter.io/flutter/material/ListTile-class.html)来构造。    

下面的代码展示了如何创建一个Drawer：
```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        backgroundColor: Colors.yellowAccent,
        appBar: AppBar(
          // leading: Icon(Icons.menu),  
          title: Text('My Title'),
          actions: <Widget>[
            IconButton(
              color: Colors.white,
              icon: Icon(
                Icons.shopping_cart,
                color: Colors.white,
              ),
              onPressed: null,
            ),
            IconButton(
              icon: Icon(
                Icons.monetization_on,
                color: Colors.white,
              ),
              onPressed: null,
            )
          ],
        ),
        body: Container(
          color: Colors.red,
        ),
        drawer: Drawer(
          child: ListView(
            children: <Widget>[
              DrawerHeader(
                child: Text("Drawer header"),
                decoration: BoxDecoration(
                  color: Colors.blue,
                ),
              ),
              Text("Item 1"),
              Text("Item 2"),
              Text("Item 3"),
              Text("Item 4"),
              Text("Item 5"),
              Text("Item 6"),
            ],
          ),
        ),
      ),
    ));
```

上面代码的运行结果如下图所示：

![ui-drawer](/assets/img/post/ui_drawer.gif)  

> 需要注意一点，上面的代码中我们去掉了 appBar 的 leading 图标，当我们给 Scaffold 添加 drawer 时，它会自动在 appbar 的 leading 上添加一个汉堡图标。  

关于 drawer 的更多细节可以查看下面的链接：
- [为屏幕添加 Drawer](https://flutter.io/cookbook/design/drawer/)
- [设置一个导航 Drawer](https://medium.com/@kashifmin/flutter-setting-up-a-navigation-drawer-with-multiple-fragments-widgets-1914fda3c8a8)   

### 3.BottomNavigationBar (BottomNavigationView)  
BottomNavigationBar 是一个展示在应用底部，用于选择不同视图（通常3-5个）的材料控件。底部导航栏包括很多项，可以是浮在材料布局上的文本、图标或者两者都有。   

底部导航栏通常和 Scaffold 一起使用，可以通过 `Scaffold.bottomNavigationBar` 属性提供。   

在 Android 里，你通过`app:menu=”@menu/my_navigation_items”`来定义 `BottomNavigationView`的菜单项。`my_navigation_items` 包括所有菜单项的`<item/>`标签列表。   

在 Flutter 里，我们使用 `items` 属性，它接收一个 `BottomNavigationBarItem` 列表。`BottomNavigationBarItem` 包含菜单的图标、标题和背景色。  

```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        backgroundColor: Colors.yellowAccent,
        appBar: ....,
        body:...,
        drawer:...,
        bottomNavigationBar: BottomNavigationBar(
            items: [
              BottomNavigationBarItem(
                icon: Icon(Icons.home),
                title: Text("Home"),
              ),
              BottomNavigationBarItem(
                icon: Icon(Icons.search),
                title: Text("Search"),
              )
            ]
        ),
      ),
    ));
```

![](/assets/img/post/ui_bottom_nav.png)   

如你所见，底部有一个包含两个菜单的 `BottomNavigationBar`。  

>处理点击事件和改变 Scaffold 的 body 属性需要一个带状态的部件(Stateful widget)和一些额外的工作，已经超出了这篇文章的范畴，你可以在[官方文档](https://docs.flutter.io/flutter/material/BottomNavigationBar-class.html)里了解更多。   

此外，我还给 Scaffold 添加了一个悬浮按钮。   

下面是使用 Scaffold 展示我们Activity 界面的完整代码。 

```dart
import 'package:flutter/material.dart';

void main() =>
    runApp(MaterialApp(
      home: Scaffold(
        backgroundColor: Colors.yellowAccent,
        appBar: AppBar(
          leading: Icon(Icons.menu),
          title: Text('My Title'),
          actions: <Widget>[
            IconButton(
              color: Colors.white,
              icon: Icon(
                Icons.shopping_cart,
                color: Colors.white,
              ),
              onPressed: null,
            ),
            IconButton(
              icon: Icon(
                Icons.monetization_on,
                color: Colors.white,
              ),
              onPressed: null,
            )
          ],
        ),
        body: Container(
          color: Colors.red,
        ),
        drawer: Drawer(
          child: ListView(
            children: <Widget>[
              DrawerHeader(
                child: Text("Drawer header"),
                decoration: BoxDecoration(
                  color: Colors.blue,
                ),
              ),
              Text("Item 1"),
              Text("Item 2"),
              Text("Item 3"),
              Text("Item 4"),
              Text("Item 5"),
              Text("Item 6"),
            ],
          ),
        ),
        bottomNavigationBar: BottomNavigationBar(
            items: [
              BottomNavigationBarItem(
                icon: Icon(Icons.home),
                title: Text("Home"),
              ),
              BottomNavigationBarItem(
                icon: Icon(Icons.search),
                title: Text("Search"),
              )
            ]
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: () {},
          child: Icon(Icons.add),
        ),
      ),
    ));
```
  

![floating_aciton_button](/assets/img/post/ui_floating_button.png)


>如果悬浮按钮的`onPressed`回调是`null`,那么按钮将会不可用并且不响应触摸事件。所以为了有触摸效果，你需要处理`onPressed`回调——保持空函数或者执行其他操作。  

最终，我们完成了在文章开始要构建的界面。   

## 总结 
Flutter 是一个可以快速构建高质量、美观界面的强大工具，它有很多部件可以用于构建具有很棒动画的灵活界面，Scaffold 是其中的一个并且它只是冰山一角。  
我希望在接下来的博客中能探讨关于它们的更多话题。

感谢！  




















