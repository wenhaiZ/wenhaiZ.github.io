---
layout: post
title: "关于 Cursor 的一件小事"
date: 2017-09-24 14:01:00 +0800
tags: [Code,Android]
comments: true
subtitle: "记录一个小 bug"
code-link: "assets/code/170924.md"
---  
昨天在整理去年的一个日记应用 [Dear Diary](https://github.com/wenhaiz/DearDiary) 时，发现定时提醒总是报错，通过断点调试发现是按天查询日记后获取日记内容时抛了一个异常  
![code01](/assets/img/post/code/170924_01.png)

抛出异常的方法代码如下：
![code02](/assets/img/post/code/170924_02.png)

根据异常栈信息，是第一句通过调用 Cursor#getInt() 时抛出的。   
我最初以为是 getColumnIndex 方法有问题，后来发现并不是它的问题。  

通过断点调试时找到执行的方法是在 AbstractWindowedCursor 中的 getInt 中，这个方法的代码如下： 
![code03](/assets/img/post/code/170924_03.png)

在 checkPositon 方法中，调用了父类的方法，AbstarctWindowedCursor 的父类是 AbstractCursor，代码如下： 
![code04](/assets/img/post/code/170924_04.png)

可以看出，当 mPos 为 -1 或者等于记录数量时，就抛出了上面的异常。   

经过研究，这个 mPos 就是数据在 Cursor 中对应的位置，相当于数组的下标，初始值为 -1。

了解这些后，我看了一下我的查询方法：  
![code05](/assets/img/post/code/170924_05.png)

也就是说，在查到记录后，我直接把 Cursor 传给 getDiaryFromCursor，这时 Cursor 的 mPos 还是 -1，所以就会抛出异常。   

解决的办法很简单，就是在调用 getDiaryFromCursor 前加一句： 
![code06](/assets/img/post/code/170924_06.png)

通过看源码就知道 moveToFirst 做了什么： 
![code07](/assets/img/post/code/170924_07.png)

而 moveToPositon 的逻辑就是安全的给 mPos 赋值: 
![code08](/assets/img/post/code/170924_08.png)

所有的 moveXxx 方法最终都会调用 moveToPosition 来完成对 mPos 的修改，修改成功就返回 true，出现意外就返回 false 。

另外，正是由于 mPos 初始值为-1，所以才可以像下面这样通过 moveToNext 方法来完成对 Cursor 的遍历： 
![code09](/assets/img/post/code/170924_09.png)

因为 moveToNext() 代码如下： 

![code10](/assets/img/post/code/170924_10.png)

第一次调用 moveToNext() 后，mPos 被修改为 0，这样就指向了第一条记录。  

因此，上面的 moveToFirst 也可以改为 moveToNext。 

虽然数据库操作有很多开源框架使用，但是了解一下 Cursor 原理也没什么坏处。   

这篇文章总结起来就是一句话：Cursor 中的位置指针默认为 -1，在使用 Cursor 前，记得通过 moveToXxx 方法把指针移动到你想要的位置。