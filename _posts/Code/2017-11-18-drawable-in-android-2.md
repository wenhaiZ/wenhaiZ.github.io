---
layout: post
title: "Android 中的 Drawable （二）"
date: 2017-11-18 21:54:00 +0800
tags: [Code,Android]
subtitle: "一些需要进行代码配置的 Drawable"
code-link: "assets/code/171118.md"
---
>我想用两篇文章结合我看的书籍和官方文档总结 Android 中 `Drawable` 的用法，以便查阅，这是第二篇。 

关于 Drawable 的[第一篇博客](/drawable-in-android-1)中介绍了一些定义后就可以直接引用的 Drawable，这一篇文章中的 Drawable 在通过 xml 文件定义后，还需要通过代码配置才能达到预期效果。

## AnimationDrawable
`AnimationDrawable` 代表一种动画资源——[FrameAnimation](https://developer.android.google.cn/guide/topics/resources/animation-resource.html#Frame)，也就是所谓的帧动画，它通过循环播放一系列静态图片来达到动画效果。   
AnimationDrawable 对应`<animation-list>`标签，定义语法如下：
![code01](/assets/img/post/code/171118_01.png)

`oneshot` 属性表示动画是否只播放一次，每个`<item>` 对应动画的一帧，可通过`drawable`和`duration`来定义该帧的 Drawable 和显示时长。  

AnimationDrawable 需要配合如下代码配置：
![code02](/assets/img/post/code/171118_02.png)

在 `start()` 方法调用后，就会显示动画效果。
## [LevelListDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#LevelList)
`LevelListDrawable` 代表一系列具有级别值的 Drawable 的集合，根据设置的级别，LevelListDrawable 会显示对应的 Drawable。   
LevelListDrawable 对应标签为`<level-list>`，声明语法如下： 
![code03](/assets/img/post/code/171118_03.png)

每个 `<item>` 都有 `maxLevel` 和 `minLevel`两个属性，分别表示最大和最小级别。当通过 `setLevel(level)`(或者 ImageView 的 setImageLevel) 设置级别时，符合`minLevel<=level<=maxLevel` 的第一个 item 会被显示。    

例如下面定义的 LevelList：  

![code04](/assets/img/post/code/171118_04.png)

在下面代码设置下，会随着 Button 点击而依次变换显示的 Drawable。 

![code05](/assets/img/post/code/171118_05.png)

## [TransitionDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Transition)
`TransitonDrawable` 用于实现两个 Drawable 之间淡入淡出的转换效果，对应标签为`<transition>`，可通过如下形式定义： 
![code06](/assets/img/post/code/171118_06.png)

**TransitonDrawable 只支持两个 Drawable。**   
`top`、`right`等属性代表距离四边的偏移量。可通过 TransitionDrawable 的 `startTransition(int duration)` 和 `reverseTransition(int duration)` 两个方法开始转换效果，传入参数为转换需要的时间（毫秒）。   

需要注意的是，startTransition 效果为从第一个 item 转换为 第二个 item；而 reverseTransition 的效果为从当前 item 转换到另一个 item，即通过 reverseTransition 可以实现在两个 Drawable 之间反复转换的效果。
## [ScaleDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Transition)
`ScaleDrawable` 可根据自己的等级将一个 Drawable 缩放至一定的比例，对应标签为`<scale>`，定义语法如下：

![code07](/assets/img/post/code/171118_07.png)

`scaleGravity`表示 Drawable 缩放后的在 View 的重心位置，`scaleHeight` 和 `scaleWidth` 是缩放百分比，其值为`xx%`，如果想缩放为原来的`40%`，那么需要设定值为`60%`。  

需要注意的是，直接将 ScaleDrawable 设为 View 的背景是不会显示的，因为此时 ScaleDrawable 的 level 为 `0`，需要通过 `setLevel(int level)` 来设定级别，level 取值为 `1~10000`，**level 越小，缩放就越明显，一般设为 1。**    

例如，定义一个将原图缩放为20% 的 ScaleDrawable： 

![code08](/assets/img/post/code/171118_08.png)

![code09](/assets/img/post/code/171118_09.png)

## [ClipDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Transition)
`ClipDrawable` 可根据自身级别对一个 Drawable 进行裁剪，对应`<clip>`标签，定义方式如下：
![code10](/assets/img/post/code/171118_10.png)

`clipOrientation` 表示裁剪方向，分为水平裁剪和垂直裁剪，最终裁剪效果由 `clipOrientation`、`gravity` 以及 `level` 共同决定。   
`gravity` 和 `clipOrientation` 对裁剪方式的影响如下表：     

|gravity 属性值|含义|
|:---:|:-------:|
|top|将对象放在其容器顶部，不改变其大小。当 clipOrientation 是 "vertical" 时，在可绘制对象的底部裁剪。|
|bottom|将对象放在其容器底部，不改变其大小。当 clipOrientation 是 "vertical" 时，在可绘制对象的顶部裁剪。|
|left|将对象放在其容器左边缘，不改变其大小，这是默认值。当 clipOrientation 是 "horizontal" 时，在可绘制对象的右边裁剪。|
|right|将对象放在其容器右边缘，不改变其大小。当 clipOrientation 是 "horizontal" 时，在可绘制对象的左边裁剪。|
|center|将对象放在其容器的水平和垂直轴中心，不改变其大小。当 clipOrientation 是 "horizontal" 时，在左边和右边裁剪。当 clipOrientation 是 "vertical" 时，在顶部和底部裁剪。|
|center_vertical|将对象放在其容器的垂直中心，不改变其大小。裁剪行为与重力为 "center" 时相同。|
|center_horizontal|将对象放在其容器的水平中心，不改变其大小。裁剪行为与重力为 "center" 时相同。|
|fill|按需要扩展对象的垂直大小，使其完全适应其容器。不会进行裁剪，因为可绘制对象会填充水平和垂直空间（除非可绘制对象级别为 0，此时它不可见）。|
|fill_vertical|按需要扩展对象的垂直大小，使其完全适应其容器。当 clipOrientation 是 "vertical" 时，不会进行裁剪，因为可绘制对象会填充垂直空间（除非可绘制对象级别为 0，此时它不可见）。|
|fill_horizontal|按需要扩展对象的水平大小，使其完全适应其容器。当 clipOrientation 是 "horizontal" 时，不会进行裁剪，因为可绘制对象会填充水平空间（除非可绘制对象级别为 0，此时它不可见）。|
|clip_vertical|可设置为让子元素的上边缘和/或下边缘裁剪至其容器边界的附加选项。裁剪基于垂直重力：顶部重力裁剪上边缘，底部重力裁剪下边缘，任一重力不会同时裁剪两边。|
|clip_horizontal|可设置为让子元素的左边和/或右边裁剪至其容器边界的附加选项。裁剪基于水平重力：左边重力裁剪右边缘，右边重力裁剪左边缘，任一重力不会同时裁剪两边。|   

`level` 决定了裁剪区域的大小，0 为完全裁剪，10000 为不裁剪，即如果 level=2000，那么将会裁剪 Drawable 的 80%。

下面代码设置了一个会在底部裁剪的 ClipDrawable： 
![code11](/assets/img/post/code/171118_11.png)

当 `level`分别设为 `10000` 和 `6500` 时，效果如下：  
![](/assets/img/post/clip_preview.png)

## 自定义 Drawable
Andorid 提供了丰富的 Drawable 资源，可以满足大多数开发场景，但特殊情况下也可以自定义 Drawable。   
自定义 Drawable 可以通过继承 Drawable 类并重写相应方法实现： 
![code12](/assets/img/post/code/171118_12.png)

通过 `draw` 方法绘制内容，如果需要重绘，可调用 `invalideSelf` 方法。
需要注意的是，自定义的 Drawable 不能在 Xml 中使用。   

-----------------------
*关于 Android 中的 Drawable 的总结，到此结束了，如果以后遇到相关内容，再做更新。*

## Refs
1. [Android API Guides : Drawable Resource](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html)
2. [Reference :Drawable](https://developer.android.google.cn/reference/android/graphics/drawable/Drawable.html)
3. [FrameAnimation](https://developer.android.google.cn/guide/topics/resources/animation-resource.html#Frame)
4. 《Android 开发艺术探索》 Chapter 6