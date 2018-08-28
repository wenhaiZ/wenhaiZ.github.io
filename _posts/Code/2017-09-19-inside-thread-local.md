---
layout: post
title: "ThreadLocal 原理分析"
date: 2017-09-19 11:03:00 +0800
tags: [Code,Java]
comments: true
subtitle: "基于 JDK 1.8"
---
[ThreadLocal](https://developer.android.google.cn/reference/java/lang/ThreadLocal.html) 对于我来说是一个没有用过但是偶尔会看到的类。在分析 Android 消息机制源码时就看到了它，这篇文章简单介绍它的用法并从源码的角度来分析它的工作原理。   

## ThreadLocal 的作用
对于 ThreadLocal， 官方文档是这么解释的：
>This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable.       

翻译过来就是：ThreadLocal 提供了一个「线程局部变量」，这个变量与别的变量不同的地方在于，每个通过 set 或 get 方法访问它的线程都拥有该变量的一个独立的拷贝。   

也就是说，线程之间是不共享这个变量的，按照这个意思，`ThreadLocal` 改名为 `ThreadLocalVariable` 比较合适。

## ThreadLocal 的应用场景  
根据 ThreadLocal 的特点，可以知道它的一些典型的应用场景：
### 数据以线程为作用域并且不同线程拥有不同的副本   
这比较符合 ThreadLocal 设计的初衷，比如 Android 中每个 HandlerThread 都有自己的 Looper ，而通过 ThreadLocal 的 get 方法就可以获取当前线程对应的 Looper 对象。
### 复杂逻辑下的对象传递  
这理解起来有点抽象，举个例子比较好懂，也许不是很恰当，但主要是说明问题。  

比如在一个线程内的任务涉及到了许多函数，调用栈比较深，而有一个监听器对象需要贯穿整个任务过程，这个时候可以将监听器对象保存在 ThreadLocal 中，然后在需要的地方通过 get 方法来获取就可以了。  

对于第二种情况，如果不用 ThreadLocal，还可以通过其他三种方式，但它们都有一定的局限性：
- 参数传递：调用栈过深导致每个函数都必须要有一个监听器参数，这是一种糟糕的代码设计
- 静态变量：如果存在多个线程实例，那么就需要多个静态变量，这操作起来太麻烦    
- 实例变量：如果需要传递多个对象，那么就需要多个实例变量，这同样是一种糟糕的设计  

所以对于这种情况，ThreadLocal 是最好的方式。 

## ThreadLocal 的使用
前面也提到了，ThreadLocal 的使用主要涉及 set 和 get 两个方法，通过名字可以看出，它们一个用来存，一个用来取。  

下面展示一个 demo：
```java
 public static void main(String[] args) {
    //创建一个泛型为 Integer 的 ThreadLocal 变量
    ThreadLocal<Integer> local = new ThreadLocal<>();
    //在当前线程修改
    local.set(10);

    //创建两个线程，分别修改并输出
    new Thread() {
        @Override
        public void run() {
            super.run();
            local.set(20);
            System.out.println("local in " + getName() + "=" + local.get());
        }
    }.start();

    new Thread() {
        @Override
        public void run() {
            super.run();
            local.set(30);
            System.out.println("local in " + getName() + "=" + local.get());
        }
    }.start();
    //主线程 sleep 1秒
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    //输出ThreadLocal在主线程的值
    System.out.println("local in " + Thread.currentThread().getName() + "=" + local.get());
}
```

上面代码创建了一个 ThreadLocal 变量，然后在三个线程中分别修改它的值并输出，为了使效果更有说服力，特意把主线程休眠了 1 秒，如果线程之间共享变量，那么主线程最后输出的值一定不是 10，然而上面输出结果为：
```
local in Thread-0=20
local in Thread-1=30
local in main=10
```

因此可以看到 ThreadLocal 的确是线程局部变量。
## ThreadLocal 的原理
简单使用后，来看一下 ThreadLocal 是如何实现变量在线程之间的隔离的。  

ThreadLocal 的使用主要涉及到 get 和 set 方法，先来看这两个方法的源码。
### ThreadLocal#get()
```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程对应的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //从 map 中取出以 ThreadLocal 为键的 Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            //获取对应的 value
            T result = (T)e.value;
            return result;
        }
    }
    //如果没有找到，返回初始值
    return setInitialValue();
}
```

### ThreadLocal#set ( T value)
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    //获取当前线程对应的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //map 存在，设置对应键值对
    if (map != null)
        map.set(this, value);
    else//否则为线程创建一个map并存储该键值对
        createMap(t, value);
}
``` 

通过上面两个方法的源码可以看出，ThreadLocal 是通过分别对每个线程的 ThreadLocalMap 进行操作来实现变量在线程间的隔离的。  
ThreadLocalMap 存储了以 ThreadLocal 为 key，以对应的值为 value 的键值对。   

上面的方法中涉及到了 ThreadLocalMap、ThreadLocalMap.Entry 两个类以及 getMap 和 createMap 两个方法，先来看一下getMap 和 createMap 两个方法。
### ThreadLocal#getMap(Thread t)
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

getMap 直接返回了 Thread 实例的 threadLocals 字段，也就是说，这个ThreadLocalMap 是存储在 Thread 实例中的。  
### ThreadLocal#createMap(Thread t, T firstValue)
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

createMap 创建了一个 ThreadLocalMap 对象存储第一个 value，并赋值给线程的 threadLocals.  

接下来就要看一下 ThreadLocalMap 了。
### ThreadLocal.ThreadLocalMap
ThreadLocalMap 是 ThreadLocal 的静态内部类，它是一个专门为了存储线程局部变量而定制的哈希表，主要包含了一个数组用来存储 Entry 对象以及一些哈希表的对应操作，这里主要看一下刚才涉及的 Entry 类和 ThreadLocalMap 的 getEntry 和 set 方法。
### ThreadLocalMap.Entry
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

Entry 是 ThreadLocalMap 的静态内部类，值得注意的是，它继承了 WeakReference（这主要是避免一些长期存活并且占用内存较大的对象可能引发的问题），并以 ThreadLocal 为泛型，内部包含一个类型为 Ojbect 的 value。  
构造方法也很简单，就是为 key 和 value 赋值，其中对 key 的赋值调用了 WeakReference 的构造方法。    

接下来看一下 ThreadLocalMap 的 set 方法。
### ThreadLocalMap#set(ThreadLocal<?> key, Object value)
```java
private void set(ThreadLocal<?> key, Object value) {
    //table 就是 ThreadLocalMap 中存储 Entry 的数组
    Entry[] tab = table;
    int len = tab.length;
    //hash
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
            if (k == key) {
                e.value = value;
                return;
            }

            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

除了一些调整的操作外，set 的逻辑主要就是根据 key 的 hashCode 来确定 Entry 在哈希表中的位置，然后存储 Entry。
### ThreadLocalMap#getEntry(ThreadLocal<?> key)
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else //处理 key 不存在的情况
        return getEntryAfterMiss(key, i, e);
}
```

get 方法通过 key 来获取对应的 Entry，并通过 getEntryAfterMiss() 来处理 key 没有找到的情况，相关代码就不展示了。   

刚才还涉及到了一个构造方法，它主要是用于初始化 ThreadLocalMap 并存储第一个 Entry 对象，源码如下：
### ThreadLocalMap#ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue)
```java
 ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //初始化 table
    table = new Entry[INITIAL_CAPACITY];
    //存储第一个 Entry 对象
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
``` 

到这里，ThreadLocal 原理涉及的核心代码已经全部展示完了。   

## 总结
在 Thread 类中，有一个 ThreadLocalMap 类型的实例变量 threadLocals，它是一个存储了以 ThreadLocal 为 key，以对应值为 value 的 Entry 对象的哈希表。   

ThreadLocal 的 get 和 set 方法在执行时，先获取当前线程的 threadLocals，然后再以 ThreadLocal 为 key 进行存储和读取的操作。 
如果线程对应的 threadLocals 为 null，那么会在 set 方法中进行初始化。  

> 需要注意的是，ThreadLocal 并没有存储值，它只是操作当前线程的 threadLocals 变量，threadLocals 存储了该线程对应的所有 Entry 对象。

## PS
- 如果想为 ThreadLocal 指定初始值（即在通过 set 赋值之前使用 get 返回的值），可以在创建 ThreadLocal 时重写 initialValue() 方法，这个方法将在 setInitialValue() 中调用。例如：
```java
ThreadLocal<Integer> local = new ThreadLocal<Integer>() {
    @Override
    protected Integer initialValue() {
        return 2;//指定初始值
    }
};
```

- 在 Java 中还有一个 [InheritableThreadLocal](https://developer.android.google.cn/reference/java/lang/InheritableThreadLocal.html) ，它继承了 ThreadLocal，用于使线程继承其父线程的变量。在 Thread 类的 init() 方法中就有如下代码来通过父线程的 inheritableThreadLocals 来初始化线程的 inheritableThreadLocals：
```java
// inheritThreadLocals 标识是否继承父线程 inheritableThreadLocals 中的变量
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```


## Refs:  
- *[《Android 开发艺术探索》](https://item.jd.com/11760209.html) 10.2.1*