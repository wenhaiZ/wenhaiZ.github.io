---
layout: post
title: "Android 中的 Drawable （一）"
date: 2017-11-17 01:00:00 +0800
tags: [Code,Android]
subtitle: "一些可以直接用于显示的 Drawable"
---
>我想用两篇文章结合我看的书籍和官方文档总结 Android 中 `Drawable` 的用法，以便查阅，这是第一篇。  

## Drawable 简介
在 Android 开发中，`Drawable` 代表着一系列可绘制资源，它们通常用来作为 `View` 的背景或者 `ImageView` 等包含图像的 View 的显示内容，一般通过 XML 的形式定义，并且存放在项目的 `res/drawable` 目录下。   

Drawable 有内部宽/高的概念，可通过 `getInstrinsicWidth/getInstrinsicHeight` 获取，但并不是所有 Drawable 都有内部宽高——如果 Drawable 对应一张图片，那么内部宽高就是图片的宽高，但像颜色/形状等 Drawable 默认没有内部宽高，此时 getInstrinsicWidth/getInstrinsicHeight 返回值为 `-1`。   
还需要注意的是，*Drawable 的内部宽高并不是 Drawable 的实际大小，Drawable 作为 View 的背景时会被拉伸或压缩到与 View 同等大小。*  Drawable 的实际大小可通过 `getBounds()` 方法获取，这个方法会返回一个 `Rect` 。 


在开发中会用到很多种 Drawable，比如 BitmapDrawable,ShapeDrawable,StateListDrawable 等，它们都对应一个类，它们都是 Drawable 类的子类，Drawable 的所有子类可以查看[这篇文章](https://developer.android.google.cn/reference/android/graphics/drawable/Drawable.html)。 

这篇文章主要介绍一些不用代码配置，可直接用于显示（比如作为 View 的背景）的 Drawable，关于其他 Drawable 的介绍在[第二篇博客](/drawable-in-android-2) 

## [BitmapDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Bitmap) 
`BitmapDrawable` 代表一张图片资源，一般可以通过 `R.drawable.file_name`直接使用，但是通过 `<bitmap>` 标签可以定义更多细节。    
其 XML 定义形式如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<bitmap
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:src="@[package:]drawable/drawable_resource"
    android:antialias= [ "true" | "false"]
    android:dither=["true" | "false"]
    android:filter=["true" | "false"]
    android:gravity=["top" | "bottom" | "left" | "right" | "center_vertical" |
                      "fill_vertical" | "center_horizontal" | "fill_horizontal" |
                      "center" | "fill" | "clip_vertical" | "clip_horizontal"]
    android:mipMap=["true" | "false"]
    android:tileMode=["disabled" | "clamp" | "repeat" | "mirror"] />
```
其中的几个属性含义如下：
- **android:src**：**图片资源的 id**
- **android:antialias**:**是否开启抗锯齿**，一般设为 `true`。开启抗锯齿会使图像显示的更平滑，但会略微降低图像的质量。
- **android:dither**：**是否开启抖动效果**。当图片的像素配置与手机屏幕不一致时(例如图片色彩模式为 ARG8888，而手机屏幕支持的色彩模式为 RGB555)，开启这个选项可以使高质量的图片在低质量的手机屏幕上保持很好的显示效果。
- **android:filter** ：**是否开启过滤**。开启过滤可以是位图在拉伸或者压缩时能平滑的显示。
- **android:mipMap** ：**是否使用 mipmap**。mipmap 叫纹理映射，如果图像需要缩小后绘制，开启这个选项能获得较高的图片质量，但同时也会额外消耗内存，而且这个属性不保证一定起作用，因此一般为 false。 关于这个属性更详细的介绍[参照这里](https://developer.android.google.cn/reference/android/graphics/Bitmap.html#setHasMipMap(boolean))。
- **android:gravity**：**当位图小于容器尺寸时，通过此属性来设定图像的位置**，它的可选值及其含义如下表：  

| 可选值 | 说明 |
|:----:|:----:|
|top|将图片放在容器顶部，不改变图片大小|
|bottom|将图片放在容器底部，不改变图片大小|
|left|将图片放在容器左部，不改变图片大小|
|right|将图片放在容器右部，不改变图片大小|
|center_vertical|使图片在容器内垂直方向居中，不改变图片大小|
|center_horizontal|使图片在容器内水平方向居中，不改变图片大小|
|center|使图片在容器内水平和垂直方向居中，不改变图片大小|
|fill_vertical|图片在垂直方向填充容器，图片高度小于容器高度时会垂直拉伸|
|fill_horizontal|图片在水平方向填充容器，图片宽度小于容器宽度时会水平拉伸|
|fill|图片在水平方向和垂直方向填充容器，图片尺寸小于容器尺寸时会进行拉伸|
|clip_vertical|附加选项。基于垂直重力对图片进行裁切|
|clip_horizontal|附加选项。基于水平重力对图片进行裁切|    

- **android:tileMode** ：**平铺模式**。开启平铺会忽略 `gravity` 属性，其可选值和说明如下表：     


|可选值|说明|
|:----:|:-------:|
|disable|关闭平铺模式|
|repeat|平铺时重复图像|
|mirror|平铺时对图像进行交替镜像|
|clamp|平铺时重复边缘颜色|    

下图直观说明了这三种平铺模式的效果：    
![tile_mode_preview](/assets/img/post/tile_mode.png)

## [NinePatchDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#NinePatch)
`NinePatchDrawable` 对应 `<nine-patch>`标签，代表一张 `.9` 格式的图片。
例如：
```xml
<?xml version="1.0" encoding="utf-8"?>
<nine-patch xmlns:android="http://schemas.android.com/apk/res/android"
    android:src="@drawable/android"
    android:dither="true"
    android:antialias="true">
</nine-patch>
```
在 `.9` 图片里我们可以自定义需要拉伸的部分以及显示内容的区域，详情可以参考[这里](https://developer.android.google.cn/guide/topics/graphics/2d-graphics.html#nine-patch)。   
BitmapDrawable 的属性同样适用于 NinePatchDrawable。
## [ColorDrawable](https://developer.android.google.cn/guide/topics/resources/more-resources.html#Color)
`ColorDrawable` 代表颜色资源，它在 `res/values/colors.xml` 中定义：
```xml
<resources>
    <color name="colorPrimary">#3F51B5</color>
    <color name="colorPrimaryDark">#303F9F</color>
    <color name="colorAccent">#FF4081</color>
</resources>
```
ColorDrawable 可通过 `R.drawable.color_name` 引用用。  

## [ShapeDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Shape)
`ShapeDrawable` 代表用纯色或渐变色填充的简单图形，例如矩形、圆形等，它通过`<shape>` 标签来定义，语法稍显复杂：
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape=["rectangle" | "oval" | "line" | "ring"] >
    <corners
        android:radius="integer"
        android:topLeftRadius="integer"
        android:topRightRadius="integer"
        android:bottomLeftRadius="integer"
        android:bottomRightRadius="integer" />
    <gradient
        android:angle="integer"
        android:centerX="float"
        android:centerY="float"
        android:centerColor="integer"
        android:endColor="color"
        android:gradientRadius="integer"
        android:startColor="color"
        android:type= ["linear" | "radial" | "sweep"]
        android:useLevel=["true" | "false"] />
    <padding
        android:left="integer"
        android:top="integer"
        android:right="integer"
        android:bottom="integer" />
    <size
        android:width="integer"
        android:height="integer" />
    <solid
        android:color="color" />
    <stroke
        android:width="integer"
        android:color="color"
        android:dashWidth="integer"
        android:dashGap="integer" />
</shape>
```
`shape` 属性表示形状，包括矩形（`rectangle`）、椭圆(`oval`)、线（`line`）和圆环（`ring` ）。   
针对 ring 形状还有几个特殊属性：    

|属性值|说明|
|:---:|:----:|
|android:innerRadius|圆环内半径|
|android:thickness|圆环厚度，即外半径和内半径之差|
|android:innerRadiusRatio|内半径占整个 Drawable 宽度的比例，如果为 n，则内半径 = 宽度/n（和 innerRadius 同时存在时，以 innerRadius 为准）|
|android:thicknessRatio|圆环厚度占整个 Drawable 宽度的比例，如果为 n，则厚度 = 宽度/n|
|android:useLevel|一般设为 false，否则无法达到预期显示效果，除非它被当作 LevelListDrawable|   


\<shape> 可包含的子标签说明如下：
- **\<corners>** ：**圆角角度，只适用于矩形（rectangle）。**`radius`表示为四个角设置相同的角度，其他四个属性用于分别为对应的四个角设置圆角角度，如果此时也设置了 `radius`，那么`radius`会被覆盖。
- **\<gradient>** : **渐变填充**。其中 `angle` 表示渐变角度，其值必须为 45 的倍数，会影响渐变方向（0 表示从左到右，45 表示左上到右下，90 表示从上到下，以此类推）；`centerX` 和 `centerY` 表示渐变中心坐标；`startColor`、`centerColor`和`endColor`表示渐变的起始色、中间色和结束色；`type`表示渐变种类，分为线性渐变（`linear`，默认值）、径向渐变（`radial`）和流线型渐变(`sweep`)；`gradientRadius`表示渐变半径，仅在渐变类型为`radial`时有效。
- **\<solid>** ：**纯色填充**，与渐变填充对应，通过`color`指定填充颜色。
- **\<stroke>** ：**描边**。`width`表示描边宽度；`color`表示描边颜色；`dashWidth` 和 `dashGap` 表示虚线描边的宽度和间隔，二者之一为 0 则无虚线描边效果。
- **\<padding>** ：**包含该 ShapeDrawable 的 View 的内边距**。
- **\<size>** ：**shape 的大小**，通过 `width` 和 `height` 定义宽高。这个标签设置 ShapeDrawable 的内部宽高，但其实际宽高会根据包含它的 View 的大小进行拉伸或缩小   

## [LayerDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#LayerList)
`LayerDrawable` 表示一组带有层次结构的 Drawable 集合，对应标签为`<laryer-list>`，定义语法如下：  
```xml
<layer-list
    xmlns:android="http://schemas.android.com/apk/res/android" >
    <item
        android:drawable="@[package:]drawable/drawable_resource"
        android:id="@[+][package:]id/resource_name"
        android:top="dimension"
        android:right="dimension"
        android:bottom="dimension"
        android:left="dimension" />
</layer-list>
```
每个 `<item>` 对应一个 Drawable，也就是 LayerDrawable 中的每一层，下面的 item 会覆盖上面的 item 。`top`,`right`等属性表示每层 Drawable 相对于 View 的左上角的偏移量。
例如：
```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/shape1"
        android:left="10dp"
        android:top="10dp"/>
    <item
        android:drawable="@drawable/shape2"
        android:left="20dp"
        android:top="20dp"/>
    <item
        android:drawable="@drawable/shape3"
        android:left="40dp"
        android:top="40dp"/>
</layer-list>
```
显示效果如下：   
![layer-preview](/assets/img/post/layer_preview.png)

## [StateListDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#StateList)
`StateListDrawable` 是比较常用的一种 Drawable，用于给控件设定在不同状态（比如 Button 的按下和非按下）时所显示的 Drawable，对应标签为`<selector>`，定义语法如下：
```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android"
    android:constantSize=["true" | "false"]
    android:dither=["true" | "false"]
    android:variablePadding=["true" | "false"] >
    <item
        android:drawable="@[package:]drawable/drawable_resource"
        android:state_pressed=["true" | "false"]
        android:state_focused=["true" | "false"]
        android:state_hovered=["true" | "false"]
        android:state_selected=["true" | "false"]
        android:state_checkable=["true" | "false"]
        android:state_checked=["true" | "false"]
        android:state_enabled=["true" | "false"]
        android:state_activated=["true" | "false"]
        android:state_window_focused=["true" | "false"] />
</selector>
```
每个 item 都表示一个 Drawable，通过 `state_xxx` 来设定对应状态：       

|状态|说明|
|:---:|:-----:|
|android:state_pressed|是否按下|
|android:state_focused|是否获得输入焦点|
|android:state_hovered|光标是否停留|
|android:state_selected|是否被用户选择|
|android:state_checkable|是否**可选中**|
|android:state_checked|是否**被选中**|
|android:state_enabled|是否可用|
|android:state_activated|是否激活作为持续选择|
|android:state_window_focused|应用窗口是否具有焦点（在前台）|  

StateListDrawable 可设置的属性说明如下：
- **android:dither**：**是否开启抖动**，与 BitmapDrawable 相同
- **android:constantSize**：**StateListDrawable 的固有大小是否不随状态改变**，因为对应不同状态的 Drawable 可能具有不同的固有大小。如果为 true ，那么固有大小为所有 Drawable 的最大值。
- **android:variablePadding**：**StateListDrawable 的 padding 是否随状态改变**，若为 false，则 padding 为所有 Drawable 的 padding 的最大值。 


## [InsetDrawable](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html#Inset)
`InsetDrawable` 可以以一定距离插入其他 Drawable，对应标签`<inset>`,常用于 View 的背景比实际区域小的情况，声明语法：
```xml
<inset xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/background"
    android:insetTop="10dp"
    android:insetBottom="10dp"
    android:insetLeft="10dp"
    android:insetRight="10dp">
    <!-- other drawables.... -->
</inset>
```
其中 `insetXxx`属性表示各边内边距。

## Refs
1. [Android API Guides : Drawable Resource](https://developer.android.google.cn/guide/topics/resources/drawable-resource.html)
3. [Reference : Drawable](https://developer.android.google.cn/reference/android/graphics/drawable/Drawable.html)
2. 《Android 开发艺术探索》 Chapter 6