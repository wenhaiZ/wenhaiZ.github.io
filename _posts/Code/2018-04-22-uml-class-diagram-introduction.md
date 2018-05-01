---
layout: post
title: "常用 UML 类图总结"
date: 2018-04-24 21:26:00 +0800
tags: [Code,Android,Java,CS]
subtitle: ""
---
>本文介绍了类及其关系在 UML 中的表示。   

## [UML](https://zh.wikipedia.org/wiki/%E7%BB%9F%E4%B8%80%E5%BB%BA%E6%A8%A1%E8%AF%AD%E8%A8%80) 
UML 是 Unified Modeling Language 的缩写，中文全称为统一建模语言，用于将一个软件系统（的架构和用例等）可视化。   

UML 中的模型主要分为三种：   
- 功能模型：从用户角度展示软件功能。包括用例图；
- 对象模型：通过对象、属性、操作、关联等概念展示系统的结构。包括类别图、对象图；
- 动态模型：展现系统的内部行为。包括序列图，活动图，状态图。     

本文只介绍对象模型中的[类图](https://zh.wikipedia.org/wiki/%E9%A1%9E%E5%88%A5%E5%9C%96)，其主要作用就是描述系统或模块的类集合、类的属性以及类之间的关系。

## 类与接口 
在 UML 类图中，使用一个矩形来表示类（class），矩形分为三层，从上到下分别表示类名、属性和方法。   
例如下面的类图描述了一个名为 `User` 的类:   

![User类](/assets/img/post/user.png)   

在 UML 类图中，用`访问修饰符 属性名：类型`的形式表示类的属性,用`访问修饰符 方法名（参数列表）：返回值类型`的形式表示类的方法，其中访问修饰符有四种：   

- \+ ：`public`
- \- : `private`
- \# : `protected`
- ~ : `package`    

*如果类属性和方法是静态的，属性名和方法名需要加**下划线**；表示抽象类时，类名以及抽象方法的名称均为**斜体**。*

接口（interface）在 UML 中同样用矩形表示，不过在名称上面一行有 `<<interface>>` 标记。一般情况下接口只有两部分，即名称和方法，如果有常量可以像类一样放在第二层。   
下图表示一个名为 `Fly` 并且含有一个 `fly` 方法的接口。  

![Fly 接口](/assets/img/post/interface.png) 
## 类与类之间的关系 
在软件系统中类并不是单独存在的，所以 UML 提供了多种图形来描述类与类之间的各种关系。
### 泛化(Generalization)
泛化其实就是继承，只不过继承是从子类的角度出发，而泛化是从父类的角度出发，子类继承父类，而反过来父类就是对子类的泛化。   

泛化表示`is-a`的关系，是对象之间耦合度最大的一种关系，对应编程语言中的继承（比如 `Java` 中的 `extends`）。  

在 UML 类图中使用带三角箭头的实线表示，箭头从子类指向父类。例如，下面 UML 类图表示了泛化关系：   
![generalization](/assets/img/post/generalization.png)    
上面类图对应代码如下：   
![code01](/assets/img/post/code/180422_01.png)
### 接口实现（Interface Realization）   
看名字就知道，这是描述一个类实现了某个接口，比如在 Java中使用 `implements` 来表示接口实现。   

类图中，接口实现用带三角箭头的虚线来表示，如下图：
![interface_realization](/assets/img/post/interface_realization.png)   
对应Java代码如下：
![code02](/assets/img/post/code/180422_02.png)
### 依赖(Dependency)
依赖关系是一种使用关系，即一个对象使用另一个对象。大多数情况下，依赖关系在代码中体现为某个类的方法使用另一个类的对象作为参数。    

在UML中，依赖关系用带箭头的虚线表示，由依赖的一方指向被依赖的一方： 

![dependency](/assets/img/post/dependency.png) 
 
对应 Java 代码为：
![code03](/assets/img/post/code/180422_03.png)
### 关联(Association)
关联(Association)关系是类与类之间最常用的一种关系，它是一种结构化关系，用于表示一类对象与另一类对象之间有联系，这种关系比依赖关系要强。关联关系在代码中一般表现为一个类做为另一个类的成员变量。   

在UML类图中，关联关系由实线表示。 


例如，Android 中的一个 Activity 可能有一个 Button 对象，就可以用下面类图表示：    

![association](/assets/img/post/association.png) 

对应代码为：   
![code04](/assets/img/post/code/180422_04.png)


>在表示关联关系时可以在关联线上标注角色名，一般用一个表示两者关系的动词或者名词表示（有时该名词为实例对象名），关系的两端代表两种不同的角色，角色名不是必须的，其目的是使类之间的关系更加明确。  

上面例子表示的是双向关联，除了双向关联，关联关系还包括单向关联、自关联、多重性关联。   
#### 单向关联 
下面类图表示了单向关联，一个 Activity 包含一个Button。   

![directed_association](/assets/img/post/directed_association.png)

对应代码：
![code05](/assets/img/post/code/180422_05.png)
#### 自关联 
自关联表示一个类拥有一个与该类类型相同的成员变量，例如一个 Node 类型的结点包含一个 Node 类型的next 变量表示下一个结点。 
 ![self_association](/assets/img/post/self_association.png)  
 对应代码：
![code06](/assets/img/post/code/180422_06.png)
#### 多重性关联 
多重性关联表示两个关联对象在数量上的对应关系。   
在UML中，对象之间的多重性可以直接在关联直线上用一个数字或一个数字范围表示，常见的多重性表示方式如下表所示：   

|表示方式|多重性说明|
|:-----:|:-----:|
|0..1|表示另一个类的一个对象与该类的零个或一个对象有关系|  
|1|表示另一个类的一个对象只与该类的一个对象有关系| 
|0..*|表示另一个类的一个对象与该类的零个或多个对象有关系| 
|1..*|表示另一个类的一个对象与该类的一个或多个对象有关系|
|m..n|表示另一个类的一个对象与该类最少m，最多n个对象有关系 (m≤n)|    

例如，一个Activity可能包含零个或多个按钮，其 UML 类图可表示如下：   

 ![multi_association](/assets/img/post/multi_association.png) 

对应代码：
![code07](/assets/img/post/code/180422_07.png)

### 聚合（Aggregation）
聚合表示整体与部分的关系，在聚合关系中，成员对象是整体对象的一部分，**但是成员对象可以脱离整体对象独立存在**，例如公司与其地址。  

在代码实现聚合关系时，成员对象通常作为构造方法、Setter 方法或业务方法的参数注入到整体对象中。   

在 UML 类图使用空心的菱形和实线表示聚合，菱形从局部指向整体（多重性同样适用于聚合）：   

 ![aggregation](/assets/img/post/aggregation.png) 
对应代码：
![code08](/assets/img/post/code/180422_08.png)
### 组合(Composition)
组合关系也表示类之间整体和部分的关系，但它是一种更强的聚合关系。在组合关系中整体对象负责控制成员对象的生命周期，成员对象不能脱离整体对象而存在，比如人有鼻子和嘴，但鼻子和嘴不能独立于人而存在。   
在代码实现组合关系时，通常在整体类的构造方法中直接实例化成员类。  

在 UML 类图使用实心的菱形和实现类表示组合关系，菱形从局部指向整体（多重性同样适用于组合关系）：   

![composition](/assets/img/post/composition.png)
上图对应代码：
![code09](/assets/img/post/code/180422_09.png)


### 注释（Comments）
UML 通过一个带卷角的长方形中注释，并使用虚线将这个带卷角的长方形和所注释的实体连接起来。   
在类图中添加注释如下图所示：
![comments](/assets/img/post/comments.png)

## 参考
- [深入浅出 UML 类图](http://www.uml.org.cn/oobject/201211231.asp)
- [《大话设计模式》](https://item.jd.com/10079261.html) Chapter 1.11
- [聊聊UML类图的简单使用](http://blog.longjiazuo.com/archives/1133)

