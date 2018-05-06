---
layout: post
title: "ButterKnife 源码简析"
date: 2017-08-19 23:01:00 +0800
tags: [Code,Android]
comments: true
subtitle: " 基于 V 8.7.0"
published: true
code-link: "assets/code/170819.md"
--- 
> 本文从源码角度分析了 ButterKnife（V8.7.0） 框架视图绑定的原理。   

![ButterKnife](https://github.com/JakeWharton/butterknife/raw/master/website/static/logo.png)
 
[ButterKnife](https://github.com/JakeWharton/butterknife) 是 `Android` 大神 [Jake Wharthon](https://github.com/JakeWharton) 的杰作，用于 `Android` 开发中的视图和资源绑定，可以让我们省去写诸如 `findViewById()`，`setOnClickListener()`和`getString()` 等代码，提升开发效率。   
这篇文章是我初步研究了项目源码后写的，只分析了视图绑定，所以只能算是源码简析。    
## ButterKnife 的使用
ButterKnife 使用起来很简单， 这里就简单示例一下。
- 添加依赖     
![code01](/assets/img/post/code/170819_01.png)

- 在 Activity 中使用
![code02](/assets/img/post/code/170819_02.png)

更多用法和介绍可以参考[官方文档](http://jakewharton.github.io/butterknife/)。

## 源码解析(*省略 log 和异常信息* )
我们从调用入口 `ButterKnife.bind()` 开始追踪。
- `ButterKnife#bind()`  
![code03](/assets/img/post/code/170819_03.png)

这里传入的 `target` 就是上面的 `MainActivity` ，然后获取根布局，最后调用了 `createBinding()`。   
`bind()` 方法还有几个重载的方法，用以支持不同类型的 `target`，比如 `Dialog`、`View`，它们最后都会调用 `createBinding()` 方法。   
- `ButterKnife#createBinding()`  
![code04](/assets/img/post/code/170819_04.png)

再看 `findConstructorForClass()` 方法
- `ButterKnife#findConstructorForClass()` 

![code05](/assets/img/post/code/170819_05.png)

可以看出，`Butterknife.bind(Object target) `方法就是寻找 `target类名_ViewBinding.class` （这个类实现了 `UnBinder` 接口），然后通过反射获得它的构造方法并创建 `UnBinder` 实例。  
`UnBinder` 接口中只有一个 `unbind()` 方法。
- `UnBinder`  
![code06](/assets/img/post/code/170819_06.png) 

那这些后缀为`_ViewBinding` 的类的内容是什么呢？   

我们可以在`\app\build\generated\source\apt\debug\packegename\`下找到这些类。比如，这里 `ButterKnife` 为我生成的 Java 文件为 `MainActivity_ViewBinding.java`。   
来看一下里面的内容吧。
- `MainActivity_ViewBinding.java`   
![code07](/assets/img/post/code/170819_07_a.png) 
![code07](/assets/img/post/code/170819_07_b.png)

由于我只绑定了一个 `Button`，所以这个类的代码很简单，但足以说明 `ButterKnife` 的工作原理。  

在 `_ViewBinding` 类的构造方法中，先通过 `findRequiredView()` 找到需要的 `View`，再调用 `castView()` 将 `View` 转换成指定类型，最后给 `Activity` 中的成员变量赋值并设置监听（如果需要的话）。   

接下来看一下 `findRequiredView()` 和 `castView()` 这两个方法。 

- `Utils.findRequiredView()` 
![code08](/assets/img/post/code/170819_08.png)

可以看到 `findRequiredView()` 也是通过 `findViewById()` 的方法来查找 `View` 的。
- `Utils.castView()` 这个方法主要用于类型转换 

![code09](/assets/img/post/code/170819_09.png)

再看设置监听时用到的 `DebouncingOnClickListener`
- `DebouncingOnClickListener`  
![code10](/assets/img/post/code/170819_10.png)

`DebouncingOnClickListener` 类实现了 `View.OnClickListener`，在`onClick()`方法中调用了`doClick()`方法，而在 `_ViewBinding.java` 中 `doClick()` 又调用了我们自己定义的用 `@onClick` 标记的方法。   

到此，`ButterKnife` 视图绑定的源码分析就完成了。   

可以看出 `ButterKnife` 的逻辑并没有很复杂——只是把我们平时写的 `findViewById()`封装到工具类中，然后为需要绑定的字段生成一个名称以 `_ViewBinding` 为后缀的 `Java` 文件，我们只需要通过注解标记一下要绑定的字段，然后通过 `ButterKnife.bind()` 就可以完成 `View` 的查找和字段的赋值。   

如果只介绍这些，就不用写这篇文章了，接下来还有更有意思的内容。

## 注解处理过程
了解了 `_ViewBinding` 类的工作原理，再来看一下这些类的源码是怎么生成的吧。这部分的源码在 `butterknife.compiler` 包下的 `ButterKnifeProcessor.java` 中。   

`ButterKnifeProcessor` 继承了 `javax.annotation.processing.AbstractProcessor`，对注解的处理主要是在 `processs()` 方法中。
- `ButterKnifeProcessor#process()`   
![code11](/assets/img/post/code/170819_11.png)

先来了解一下涉及到的几个类：
- `TypeElement`：`javax.lang.model.element`中的类，代表编程元素中的类或者接口，它提供了一些方法用于访问内部成员（字段、方法等）的信息，在这里它就代表包含视图绑定的 `Activity/Dialog/View`  
- `BindingSet`：一个`TypeElement`（比如一个`Activity`）对应的所有绑定元素的集合，它有一个 `BindingSet` 类型的 `parent` 的字段。这个类的实例可通过 `BindingSet.Builder` 创建

`process()` 的逻辑就是为每个 `TypeElment` 生成 `BindingSet`，再通过 `BindingSet.brewJava()` 生成 Java 源代码，也就是那些 `_ViewBinding`类。    
>`brew`的中文意思是「酿制」

生成 `BindingSet` 的逻辑就在 `findAndParseTarget()` 方法中，这里只分析 `@BindView` 这种情况。  

- `ButterKnifeProcessor#findAndParseTarget()` 

![code12](/assets/img/post/code/170819_12_a.png)   
![code12](/assets/img/post/code/170819_12_b.png)  

这个方法有点长，可以先不管后面的部分，只需要知道这里有一个 `builderMap`，它保存了 `TypeElement` 与 `BindingSet.Builder` 的映射。    

方法中对每个用 `@BindView` 修饰的元素都调用了 `parseBindView()` 方法，来看一下这个方法。
- `ButterKnifeProcessor#parseBindView()` 
![code13](/assets/img/post/code/170819_13_a.png)
![code13](/assets/img/post/code/170819_13_b.png)

>方法有点长，我省略了不必要的代码，注意结合注释看。  

`parseBindView()` 的主要逻辑就是为 `enclosingElement` 创建对应的  `builder` ，然后为 `builder` 添加 `element` 绑定字段，然后将 `builder` 添加到 `builderMap` 中。   

需要注意这里涉及到两个 `TypeElement`，一个是`element`,一个是`enclosingElement`。   
`element` 对应 `Activity/View/Dialog` 中用 `@BindView` 修饰的 **字段** ，而 `enclosingElement` 就对应 `Activity/View/Dialog`，还要注意 `BindingSet.Builder` 是与 `enclosingElement` 对应的。      

下面是 `parseBindView()` 中涉及到的几个方法。
- `ButterKnifeProcessor#elementToQualifiedId()` 由 element 的包名和 id 生成一个 QualifiedId 
![code14](/assets/img/post/code/170819_14.png)

- `ButterKnifeProcessor#getId()` 通过 `QualifiedId` 获取 `Id` 
![code15](/assets/img/post/code/170819_15.png)

- `BindingSet.Builder#findExistingBindingName()`  寻找 `id` 对应的 `builder`，并获取 `builder` 的 `fieldBinding` 的名称 
![code16](/assets/img/post/code/170819_16.png)
  
- `ButterKnifeProcessor#getOrCreateBindingBuilder()` 获取 `enclosingElement` 对应的 `BindingSet.Builder`，如果不存在，就创建一个 
![code17](/assets/img/post/code/170819_17.png)

- `ButterKnifeProcessor#isFieldRequired`  判断字段是否是必须的，如果字段被 `@Nullable` 修饰，则返回 `false` 
![code18](/assets/img/post/code/170819_18.png)

- `BindingSet.Builder#addField()` 为 `BindingSet.Builder` 添加一个绑定字段  
![code19](/assets/img/post/code/170819_19.png)

`addField()` 方法又调用了如下两个方法。
- `BindingSet#getOrCreateViewBindings()` 
![code20](/assets/img/post/code/170819_20.png)

- `ViewBinding.Builder#setFieldBinding()` 
![code21](/assets/img/post/code/170819_21.png)  

这里涉及到了两个类：
- `ViewBinding`: 代表一个 `View` 上的所有绑定，包括绑定的字段、方法，可以通过 `ViewBinding.Builder` 创建实例
- `FieldViewBinding`：表示一个字段与 `View` 之间的绑定 。
>除此之外，ButterKnife 中还有 `FieldAnimationBinding`，`FieldDrawableBinding`，`MethodViewBinding` 等表示绑定的类。   

现在调用栈有点深，涉及的方法也多了，我们现在还在 `parseBindView()`方法中，上面这些方法都是 `parseBinidView()` 中用到的，大致知道他们做了什么工作就可以了。    


到此，`parseBindView()` 方法执行完成，对所有`@BindView`修饰的元素执行此方法后，`builderMap` 中就存储了 `enclosingElement`(一定要注意是 `enclosingElement` ) 与 `BindingSet.Builder` 的映射。   

接下来就又回到了`findAndParseTarget()`方法中，可以继续往下进行了，代码如下：
- `ButterKnifeProcessor#findAndParseTarget()` 
![code22](/assets/img/post/code/170819_22_a.png) 
![code22](/assets/img/post/code/170819_22_b.png)    

方法中涉及到了 `BindingSet.Builder.setParent()`，上面介绍 `Bindingset` 的时候也提到了，它有一个 `parent`字段，这个方法就是为这个字段赋值。
- `BindingSet.Builder#setParent()`  
![code23](/assets/img/post/code/170819_23.png) 

这样`findAndParseTargets()` 返回，我们就拿到了 `bindingMap`，它存储了所有 `enclosingElement` 以及对应的 `BindingSet`。   

终于可以回到 `process()` 方法中了。
- `ButterKnifeProcessor#process()` 
![code24](/assets/img/post/code/170819_24.png)

然后看一下`BindingSet.brewJava()`方法。

- `BindingSet#brewJava()` 
![code25](/assets/img/post/code/170819_25.png)

`brewJava()`方法使用了 JavaPoet 框架来生成 Java 源代码。
> [JavaPoet](https://github.com/square/javapoet) 也是 [Square](https://github.com/square) 出品的 Java 源码生成框架。   

这里有一个`createType()` 方法，代码如下

- `BindingSet#createType()`  
![code26](/assets/img/post/code/170819_26.png)
>`TypeSpec` 在 `JavaPoet` 中对应一个类或接口  
这里主要涉及两个方法：`createBindingConstructorForView()` 和 `createBindingConstructor()`   

- `BindingSet#createBindingConstructorForView()`    
![code27](/assets/img/post/code/170819_27.png)
>`MethodSpec` 在 `JavaPoet` 中对应一个 `Method` 元素  
- `BindingSet#createBindingConstructor()`  
![code28](/assets/img/post/code/170819_28.png)

`Bindset.brewJava()` 执行完成后，就生成了 `JavaFile` 对象，再通过 `JavaFile#wrtieTo()` 方法，一个 Java 源文件就生成了。   
>有关通过 `JavaPoet` 生成 java 源文件的详细内容可以了解一下 [JavaPoet](https://github.com/square/javapoet)。

到此，`ButterKnife` 为我们生成源代码的原理就分析完了。    
现在再来简单梳理一下思路：   
1. 获得所有用 `@BindView` 修饰的元素，他们都有对应的 `enclosingElement`(可以是 Activity/Dialog/View)
2. 为所有`enclosingElement` 生成与之对应的 `BindingSet` 对象，它保存了这个`enclosingElment` 中所有的绑定信息
3. 通过 `BindingSet.brewJava()`方法，借助开源框架 `JavaPoet` 来生成文件名以`_ViewBinding`为后缀的 `Java` 源代码。

## 总结
本文从 `ButterKnife` 的简单使用出发，进行了原理说明和源码分析，也对 `ButterKnife` 注解处理过程进行了简要介绍。需要注意的是，本文主要针对的是绑定 `View` 的情况，其他绑定（如资源、方法等）原理大致相同。   

`ButterKnife` 是个实用又小巧的开源框架，它的代码量没有很多，非常适合作为练习阅读源码能力的项目，也建议大家可以去看一看，肯定会有很大的收获。   

