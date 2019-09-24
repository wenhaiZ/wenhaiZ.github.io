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