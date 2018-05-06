---
title: "动态添加带有样式的控件"
layout: post
date: 2017-07-31 20:44:00 +0800
tags: [Code,Android]
comments: true
subtitle: ""
code-link: "assets/code/170731.md"
---
在 `Android` 中通过代码动态添加 `View` 并不困难，只不过比起在 `xml` 布局文件中声明布局参数，通过 `Java` 代码显得没有那么直观和方便，所以一般在布局明确的情况下，都会首选在 `xml` 中添加控件并设置参数。  
## 问题从需求开始
今天在工作中遇到了一个需求：由于在一个 `Dialog` 布局中 `Button` 的数目不确定（不会超过三个，三个 `Button` 的 `Dialog` 已经是一种很失败的设计了，但需求就是这样），需要通过代码动态添加并设置监听事件。   
我封装了一个 `Dialog`，然后提供了`setTitle()`、`setContent()`来设置标题和内容，还有一个 `setButtons(JsonArray buttonList)` 方法用于给 `Dialog` 设置按钮，`buttonList` 包含服务器返回的按钮信息，包括按钮文字和点击后的动作。
![code01](/assets/img/post/code/170731_01.png)

然后在初始化 `Dialog` 布局时动态添加 `Button` 
![code02](/assets/img/post/code/170731_02.png)

写完运行一看，发现没什么大问题，就是样式有点怪。看了一眼之前写在 `xml` 中的参数，原来为了让按钮不显示边界，还设置了一个 `style`: 
![code03](/assets/img/post/code/170731_03.png)

为了保留这个 `style`， 我首先想到的是看 `Button` 有没有一个 set 开头的参数用于设置 `style`，很不幸没有，
那就只能 `Google` 了。  

## 一个巧妙的解决方法
这个方法的思想就是：先写一个布局文件，这个布局文件中除了父布局外，只有我想要的那个 `Button`；然后在初始化 `Dialog` 布局时，通过 `LayoutInflater` 加载这个布局，并通过 `findViewById()` 拿到这个 `Button`，然后再把它添加到 `Dialog` 布局中去。

我首先写了一个 `button_for_dialog.xml` 文件，内容如下：  
![code04](/assets/img/post/code/170731_04.png)

除了父布局 `LinearLayout` 之外，只有我想要的那个 `Button`。

然后在 `Dialog` 初始化时进行加载并设置： 
![code05](/assets/img/post/code/170731_05.png)

这种方法操作简单，而且可以不使用 `Java` 代码设置参数。   

## 日常踩坑
运行之后发现，程序崩掉了，异常信息为： 
![code06](/assets/img/post/code/170731_06.png)

原来在 `Android` 中，一个 `View` 只能有一个 **直接父布局** 。   

在这里一开始我的 `Button` 的父布局是名为 `buttonView` 的 `LinearLayout`，在代码中我不加处理的想把它添加到了 `Dialog` 的布局中，这时它就有两个父布局了。  
`Android` 不允许这么做，当然这么做不符合逻辑也没有道理。   

异常信息也告诉了我们解决问题的方式：先通过 `removeView()` 把 `Button` 从原来的父布局中移除掉，然后这个 `Button` 就自由了。
## 出坑在即
修改后的代码如下：
![code07](/assets/img/post/code/170731_07.png)
 

运行一下，没有问题，这样就实现了动态添加带有样式的控件。
## 一个疑问
在初始化 `Dialog` 时多次加载同一个布局，有没有可能获取到的 `Button` 是同一个呢？如果是的话，又该如何解决呢？

先来解决第一个问题。  
我把数据设置成三个按钮，然后通过打印每个 `Button` 的 `hashCode` 进行对比，发现三个 `button` 的值完全不同：
  
![code08](/assets/img/post/code/170731_08.png)

这说明这个三个 `Button` 对象是三个不同的对象，这样就没有第二个问题了，看来是我多虑了。

其实仔细想想，我们在给 `RecyclerView` 设置 `Adapter`，创建 `ViewHolder` 时其实也是重复加载同一个布局文件，如果多次加载布局生成的是同一个对象，那岂不是要为每个 `item` 都写一个布局，而且还都是一模一样的？

当然，在这里我把加载布局的代码写在了循环里面，所以每次生成的对象不同。  
如果写在循环外面，这里的 `Button` 就会是同一个了，这样就会发生比较尴尬的事，就不演示了。
## 参考
这篇文章参考了 [smilesea1988的博客](http://blog.csdn.net/smilesea1988/article/details/8672099)，虽然我们的面对的问题不同，但他的方法也解决了我的问题。  
感谢。
