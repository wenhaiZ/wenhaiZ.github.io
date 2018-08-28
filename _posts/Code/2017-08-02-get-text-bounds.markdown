---
layout: post
title: "迷之 paint.getTextBounds()"
date: 2017-08-02 21:01:00 +0800
tags: [Code,Android]
comments: true
subtitle: "一个字符引发的喜剧"
---
最近在跟着 [HenCoder](http://hencoder.com/) （一个很贴心的技术博客，作者之前是 `Flipboard` 的 `Android` 工程师，现在全职做这个博客）学习 Android 中自定义 View 绘制的内容，这篇文章说的是我在练习过程中的一个小问题。

在练习作者提供的 [ Demo](https://github.com/hencoder/PracticeDraw3)  时，有一项练习的要求是这样的：通过 `paint.getTextBounds()` 获取文字的边界信息，然后以此设置文字的绘制位置，使每个字符都要在矩形内垂直居中，如图所示：   

![default_draw](/assets/img/post/default_draw.jpg)  
上半部分是完成后的效果，下半部分是需要自己完成的。    

在描述我遇到的问题之前，先简单介绍一下 `drawText()` 方法。
## canvas.drawText(String text, float x, float y, Paint paint)
`drawText` 方法用于绘制文字。参数中的 `text` 是要绘制的文字，`paint` 是绘制时使用的画笔，`x` 和 `y`是文字的位置坐标，但这两个坐标里是有学问的。  

为了使不同文字放在一起时看起来更美观协调，Android 在绘制文字时采用了`基线（baseline）对齐`的方式，可以用下图很直观的解释这种方式。   
![baseline](/assets/img/post/baseline.jpg)  
图中的虚线就是文字的基线。   
 `drawText()` 方法中传入的 `x`、`y` 就是 **基线起点** 的坐标（以 View 的左上角为坐标原点，下同）。同时，通过上图还可以看到，文字下边缘是可以超过基线的。

当 `x` 和 `y` 都为 `0` 的时候的绘制情况如下图：  
![start_baseline](/assets/img/post/start_baseline.jpg)
可以看到**基线起点并没有紧贴着文字边缘而是留有一定的空隙**，这是为了让文字之间不那么拥挤。

更详细的知识就不多介绍了，现在只需明确两点：   
1. `drawText()` 中传入的坐标是基线起点的坐标   
2. 文字的下边缘是可以超过基线的  

有了这两点，就可以开始讲一下我遇到的问题了。
>以上关于 `drawText()` 的内容和图片均来自 `HenCoder` 的 [这篇博客](http://hencoder.com/ui-1-3/)，关于 `drawText()` 以及其他文字绘制的内容可以点击链接查看。

## 看似正确的方法
回到刚才 Demo 中的的问题，使文字垂直居中的方法肯定是将文字绘制位置往下移，问题就是移多少了。   

先来看下当前文字是怎么被绘制的：
```java
//chars ：所有要绘制的字符
//middle ：View 中矩形中心线的 y 轴坐标
for (int i = 0; i < chars.length; i++) {
    //绘制每一个字符
    canvas.drawText(chars, i, 1, 70 + 100 * i, middle, paint2);
}
``` 

>这里使用方法的是 `canvas.drawText(char[] text, int index, int count, float x, float y,Paint paint)`。参数 `index` 是要绘制文字在`text`数组的起始位置，`count` 是要绘制的文字个数。 

通过代码可以知道，当前文字的基线位置就是矩形中心线。我迷之自信的认为：向下偏移量应该是文字高度的一半，于是我就写下了如下代码：   

```java
for (int i = 0; i < chars.length; i++) {
    //获取文字边界信息,textBound 是用于接收文字边界信息的 Rect 
    paint2.getTextBounds(chars, i, 1, textBound);
    //y 轴偏移量设置为文字高度一半
    int yOffset = textBound.height() / 2;
    //绘制每一个字符
    canvas.drawText(chars, i, 1, 70 + 100 * i, middle + yOffset, paint2);
}
``` 

运行之后效果如下：
![wrong_draw](/assets/img/post/wrong_draw.jpg)  
这样看上去，除了那个 `j` 以外，别的字符看起来符合要求。这就有点奇怪了，如果是方法有问题，那为什么别的字符都正常呢？   

我用 Log 把 `textBound` 的 `top`、`bottom` 都打了出来：  
```
D/Practice13GetTextBounds: correctDraw: A top=-114,bottom=0
D/Practice13GetTextBounds: correctDraw: a top=-87,bottom=2
D/Practice13GetTextBounds: correctDraw: J top=-114,bottom=2
D/Practice13GetTextBounds: correctDraw: j top=-116,bottom=35
D/Practice13GetTextBounds: correctDraw: Â top=-144,bottom=0
D/Practice13GetTextBounds: correctDraw: â top=-120,bottom=2
``` 

可以看到 `top` 是负值，而 `bottom` **却因字符的不同有时为正有时为负**。

打开 `getTextBounds()` 的源码，方法介绍中有这么一句：
>Return in bounds (allocated by the caller) the smallest rectangle that encloses all of the characters, with an implied origin at (0,0).      

也就是说，在返回包围文字的最小矩形时默认以（0，0）为坐标原点的，`getTextBounds()` 又调用了一个 native 方法，我暂时没有办法看到源码。   

方法介绍中说以（0，0）为坐标原点，那这个矩形是怎么摆放的呢？换句话说，（0，0）是矩形哪个点的坐标呢？    
   
在 `Android` 中，`x` 轴正方向为右，`y` 轴正方向为下，结合 `top` 为负值，是不是说（0，0）就是矩形左下角的坐标呢？但如果是这样的话，那 `bottom` 就应该一直为 `0` ，但由上面的 Log 可以看出，并不是这个样子，**所以说，（0，0）肯定不是矩形的左下角坐标。**  

## 正确的画法
暂时没了头绪，突然想到看一下「答案」，也就是作者的源码，发现作者是这样写的：    
```java
for (int i = 0; i < chars.length; i++) {
    paint2.getTextBounds(chars, i, 1, textBound);
    int textMiddle = (textBound.top + textBound.bottom) / 2;
    int yOffset = - textMiddle;
    canvas.drawText(chars, i, 1, 70 + 100 * i, middle + yOffset, paint2);
}
```  

照着敲了一遍，果然那个 `j` 正常了，我却更迷惑了。

由基础的数学知识可以知道，代码中 `textMiddle` 是文字垂直中心线的 y 坐标值，是负的，`yOffset` 对其取了相反值，然后绘制的时候文字位置向下偏移了 `yOffset`。

这个操作我一时还真没看懂。   
  
那就先顺着作者的思路把图画一下。   
这里只看 y 轴坐标，以字符 `A` 为例，通过 Log 可以看到，`A` 的 `bottom = 0`，简单画个草图：
![A](/assets/img/post/A.jpg)   

图中左上角的 `A` 是 `getTextBounds()` 时的坐标位置，可以看出 `textMiddle` 就是文字垂直中心线相对于 x 轴 ( y=0) 的坐标，为负值；而 `yOffset` 是取 `textMiddle` 的相反值，也就是**文字中心线距离 x 轴的距离**，是正的。

中间的 `A`是文字绘制的原始位置 ，`middle` 是传入的 y 坐标。右边的 `A` 绘制时传入的 y 坐标是 `y + yOffset`，也就是向下移动 `yOffset` 的距离，因此它是严格的垂直居中的。

按照作者的思路捋了一遍，只是证明了作者是对的。    
  

但现在问题是为什么要计算这个 `textMiddle` ，以及为什么要以它（准确的说，是它的绝对值）作为 y 轴偏移量。
## 一个假设
目前可以明确的有：

- `textMiddle` 是获取 `top` 和 `bottom` 的中心位置，这是基础的数学知识。   
 
- `yOffset` 就是取了 `textMiddle` 与 x 轴的距离，而使用的时候，又是以 `middle` 作为起点向下偏移了 `yOffset` 。    

- `middle` 是第一次绘制文字时传入的 y 值，也就是说 `middle` 是第一次绘制时文字的基线起点的 y 坐标值。

- `yOffset` 是文字垂直中心线与 x 轴的距离，而基线下移 `yOffset` 正好可以使文字垂直居中。

想到这里，我有了一个假设：**x 轴就是左上角 A 的基线位置，而 `yOffset` 的实际意义就是文字水平中心线与文字基线的间距**。  

因为只有这样，才能解释为什么以`middle`为基线的中间的 `A`向下移动 `yOffset` 的距离，文字刚好可以垂直居中（此时文字的垂直中心线的位置正好与粉色矩形的垂直中心线——也就是`middle`所在的位置——重合）。   

如果上面这个假设成立，那么`getTextBounds()`的 **效果相当于** 先在 `drawText()` 中传入（0,0）绘制文字，然后再用一个尽可能小的矩形的去包围绘制好的文字，返回的边界信息就是这个矩形的四个边的坐标。  
## 验证假设  
那么该怎么验证这个假设呢？很好办。   

由于假设中 `getTextBounds()` 绘制文字是以（0，0）为基线，导致我们看不见文字或者只能看到文字底部的一小部分，想看到全部文字只需要把它们移下来就可以了。具体一点就是：   
1. 通过 `getTextBounds()` 获得文字边界矩形 `textBound`
2. 以一个能看到的基线位置来绘制文字，也就相当于将文字的基线从（0，0）的位置平移，这里我选择的坐标为（300，300）
3. 让矩形平移相同的距离，也就是把矩形四个边的坐标都加上 300
4. 看这个矩形是不是正好把文字包围起来。如果是，那么 `getTextBounds()` 的效果确实如假设那样，是以（0，0）为文字的基线。

那就以那个特殊的 j 为例吧，代码如下：  
```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    //chars = {'j'};
    //1.获得包围文字边界的矩形
    paint2.getTextBounds(chars, 0, 1, textBound);
    //2.文字基线位置向正方向偏移 300 后绘制文字
    int offset = 300;
    canvas.drawText(chars, 0, 1, 0 + offset, 0 + offset, paint2);
    //3.平移矩形（给矩形边界加上偏移量）
    textBound.top += offset;
    textBound.bottom += offset;
    textBound.left += offset;
    textBound.right += offset;
    //4.绘制矩形
    canvas.drawRect(textBound, paint3);
}
```
  
接下来就是见证奇迹的时刻：
![j](/assets/img/post/prove.jpg)
可以看到，矩形完美的包围了字符 j 。这说明假设是成立的，之前计算的 `yOffset` 确实就是文字垂直中心线与文字基线的距离。   
现在就可以解释为什么文字可以在粉色矩形内垂直居中：原来的基线位置向下偏移 `yOffset` ，文字垂直中心线自然就与原来的基线在同一直线上了，而原来的基线就是粉色矩形的垂直中心线。   
此外，这还解释了 `textBound` 的 `bottom` 为什么可以大于 0 （字符的下边缘可以超过基线）。 
>尽管如此，由于没有源码为证，关于 getTextBounds() 的效果还只能算是我的猜测。

其实到这里问题已经解决了，不过还是来看看两次绘制时的 j 到底为什么不一样吧。  

## 字符 j 到底经历了什么
根据 Log 信息，`j` 的 `top = -116`，`bottom = 35`，画图如下:
![j](/assets/img/post/j.jpg)
左上角的 `j` 是以（0，0）为基线起点坐标绘制的，`yOffset` 为是中心线与基线之间的距离。   

下面三个从左到右依次是初始情况（文字基线为矩形垂直中心线）、垂直居中和错误情况。
   
可以通过计算说明为什么我第一次画的 `j` 相比居中时位置偏下:    

按照正确的方法，`yOffset = -(35 - 116)/ 2 = 40.5`，而如果我的方法，`yOffset = (116 + 35) / 2 = 75.5`，35个像素之差，也就是说我第一次绘制的 j 位置比垂直居中时的位置偏下 `35` 个像素。

那为什么第一次绘制的时候别的字符看起来都是垂直居中的呢？

是的，他们也只是「看起来垂直居中」。之所以会看起来居中，是因为它们的 `bottom` 值都和 `0` 很接近。   

再来看一遍 Log 就知道了 :  
```
D/Practice13GetTextBounds: correctDraw: A top=-114,bottom=0
D/Practice13GetTextBounds: correctDraw: a top=-87,bottom=2
D/Practice13GetTextBounds: correctDraw: J top=-114,bottom=2
D/Practice13GetTextBounds: correctDraw: j top=-116,bottom=35
D/Practice13GetTextBounds: correctDraw: Â top=-144,bottom=0
D/Practice13GetTextBounds: correctDraw: â top=-120,bottom=2
```

还是通过计算来说明这个问题，按照我的计算方法（ `yOffset = ( bottom - top ) / 2`）和正确的计算方法（`yOffset = -( top + bottom ) / 2`）分别计算`yOffset`，结果如下表：   

| char | top | bottom | yOffset_T | yOffset_F|   
| :--: | :-: | :----: | :-------: | :------: |
|A     |-114 |0       |57         |57        |
|a     |-87  |2       |42.5       |44.5      |
|J     |-114 |2       |56         |58        |
|j     |-116 |35      |40.5       |75.5      |
|Â     |-144 |0       |72         |72        |
|â     |-120 |2       |59         |61        |

> `yOffset_T` :正确结果；`yOffset_F`：错误结果

可以看到，`j` 以外的字符正确结果与错误结果最多就相差 2 个像素（甚至对于 `A` 和 `Â` 这种 `bottom` 为 `0` 的字符，两种计算方式结果是一样的），这样肉眼就很难分辨出来，所以它们「看上去是居中的」；而字符 `j` 的两个结果相差多达 35 个像素，因此就能很明显的看出差别。    

问题解决了。    

总结来看，这次问题的出现是由于我想当然的使用了错误的方法，而又瞎猫碰死耗子的蒙对了部分结果导致的，而这次问题之所以能够解决，还是要感谢那个略微下移的字符 `j` 啊。