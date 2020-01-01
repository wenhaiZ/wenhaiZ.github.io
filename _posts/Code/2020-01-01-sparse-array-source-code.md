---
layout: post
title: "SparseArray 源码解析"
date: 2019-11-29 09:05:00 +0800
tags: [Code,Android]
subtitle: "一个短小精悍的工具类"
published: true
---
## 简介

SparseArray（稀疏数组）是 Android 框架提供的工具类，位于 android.util 包下，用于建立整数和对象之间的映射，效果相当于 HashMap&lt;Integer,Object&gt;，不过比 HashMap&lt;Integer,Object&gt; 更适合移动应用开发。

SparseArray 相比 HashMap&lt;Integer,Object&gt; 主要做了两点优化：一是避免了自动装箱过程，二是不依赖一个额外的 Entry 包装对象。

SpareArray 内部采用了两个数组来分别存储 key 和 value，寻找 key 时使用了二分查找，不过，也是由于这个原因，查找效率无法和 HashMap 相比，适用于少量数据的情况。

[SparseArray 源码](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/util/SparseArray.java;l=56;bpv=1;bpt=1?q=SparseArray) 一共才400多行，比较容易懂，这里只分析几个主要方法的源码。

## 属性

```java
public class SparseArray<E> implements Cloneable {
    //所有被删除的元素都会先指向这个对象
    private static final Object DELETED = new Object();
    //是否需要进行空间回收
    private boolean mGarbage = false;
    //存放key的数组
    private int[] mKeys;
    //存放value的数组
    private Object[] mValues;
    //当前元素数量
    private int mSize;
    
    //.. 
}
```

SparseArray 采用了两个数组 `mKeys` 和 `mValues` 分别存储 key 和 value，其中存储 key 的数组为`int[]`类型，避免了 HashMap 的自动装箱过程。

`mGarbage` 表示是否有必要进行「垃圾回收」，当然这里的垃圾回收与 JVM 的垃圾回收不是一个过程，只是进行数组元素的搬移，移除不再需要的元素。

`DELETED` 是一个静态常量，所有被删除的元素会先指向这个对象，可以把它当做一个标记。

## 构造方法

```java
public SparseArray(int initialCapacity) {
    if (initialCapacity == 0) {
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length];
    }
    mSize = 0;
}

public SparseArray() {
    //默认容量是10
    this(10);
}
```

SparseArray 提供了两个构造方法，带参数的构造方法可以指定初始容量，如果使用无参的构造方法，则 SparseArray 默认初始容量是10。

当初始容量为 0 时，代码将 `mkeys` 和 `mValues` 初始化为 [`EmptyArray`](https://cs.android.com/android/platform/superproject/+/master:libcore/luni/src/main/java/libcore/util/EmptyArray.java;bpv=1;bpt=1;l=24?q=EmptyArray) 的常量，避免重复创建对象。

```java
public static final int[] INT = new int[0];
public static final Object[] OBJECT = new Object[0];
```

在初始化 `mValues` 数组时使用了 [ArrayUtils](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/util/ArrayUtils.java;l=24;bpv=1;bpt=0) 的 `newUnpaddedObjectArray` 方法，该方法源码如下：

```java
import dalvik.system.VMRuntime;

public static Object[] newUnpaddedObjectArray(int minLen) {
    return (Object[])VMRuntime.getRuntime().newUnpaddedArray(Object.class, minLen);
}
```

VMRuntime 的 newUnpaddedArray 是一个 native 方法：

```java
/**
* Returns an array of at least minLength, but potentially larger. The increased size comes from
* avoiding any padding after the array. The amount of padding varies depending on the
* componentType and the memory allocator implementation.
*/
@libcore.api.CorePlatformApi
@FastNative
public native Object newUnpaddedArray(Class<?> componentType, int minLength);
```

这里就不关注具体实现了，不过通过注释可以看出，实际返回的数组大小有可能比 minLength大，也就是说，如果 minLength 是15，那么实际返回的数组大小可能是 16。

**增加的大小来源于申请内存时增加的 padding 空间**，而 padding 空间一般就是仅做填充用，不过这里的处理是把 padding 空间也算到了数组的长度里，增加了内存空间的利用率（这也就是unpadded 的语义，并不是说申请内存时没有 padding，而是把 padding 算在了数组长度里）。

> 简单解释一下为什么需要 padding。为了提高 CPU 访问内存的效率，数据的内存地址最好是数据大小的倍数（自然对齐），例如对于一个 32 位的 CPU 来说，如果有一个数据占用连续的 4 个字节，并且第一个字节位于某 4 个字节的边界处，那么就可以把它进行自然对齐。
>
> 而为了保证可以进行自然对齐，就需要对不满足条件的数据插入 padding，例如对一个32位的 CPU 来说，如果有一个16位的数据，那么在申请内存时就可以在它后面插入 16 位的 padding 空间。
>
> 更多详情可以查看维基百科[数据结构对齐](https://en.wikipedia.org/wiki/Data_structure_alignment)。

通过这个方法也可以看到，为了充分利用移动设备上的内存，Android 也是做了很多优化。

## put

put 方法用于放入一个整数到对象的映射，源码如下：

```java
public void put(int key, E value) {
    //通过二分查找来确定 key 的位置，有效长度为mSize
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    //给key已经存在元素，直接替换
    if (i >= 0) {
        mValues[i] = value;
    } else {
        // i 指向大于 key 的第一个位置
        i = ~i;
        //此位置之前有过元素不过已经删除了，直接覆盖
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            //先进行一次gc
            gc();

            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }
        //将新的key插入到数组对应位置
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

主要逻辑如下：

1. 通过二分查找在 mKeys 数组的 mSize 长度中（因为只有这一段是有效的）查看 key 是否已经存在，如果存在，则直接更新 mValues 数组中对应位置的值
2. key 不存在，那么**将二分查找返回值取反**，得到 key 要插入的位置，检查该位置的 value 是否标记为已删除，如果是，直接在该位置更新 key 和 value。
3. 如果有必要，进行一次「垃圾回收」，并更新即将插入的位置（垃圾回收过程数组元素发生了移动，所以插入位置可能会变动）
4. 在 mKeys 和 mValues 数组中的对应位置分别插入 key 和 value，size 加 1。

那为什么将二分查找的返回值按位取反就能得到新元素的位置呢，这就要看看二分查找在失败时的返回值是什么。二分查找通过 [ContainerHelpers](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/util/ContainerHelpers.java;bpv=1;bpt=1;l=22?q=Containerh&ss=android%2Fplatform%2Fsuperproject) 的 binarySearch 方法实现，源码如下：

```java
static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

在上面代码中，未找到时，返回值是对 lo 按位取反，而此时 lo 即为第一个大于 key 的位置。

举个简单的例子，如果数组元素为 \[-1,0,1,3,6\]，在使用二分查找法查找元素 5 时，lo 最后的值是4，也就是元素 5 应该在数组中下标为 4 的位置。在经过按位取反后，binarySearch 的返回值就是个负数。 而**在 put 方法中，通过** `i = ~i`**可以得到这个位置下标，然后执行更新或插入即可。**

我不知道有多少人跟我一样，写二分查找时如果未找到就直接返回-1，却忽略了这么一条很重要的特性。借助这个特性，我们在维护一个有序数组时，使用一次二分查找既可以检查数组中是否已经包含待添加的元素，又可以知道它应该存在的位置。

insert 方法逻辑比较简单，就是在数组的指定位置插入元素，如果当数组空间不足时进行扩容。

```java
public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
        assert currentSize <= array.length;
    //如果数组空间足够，直接把index后面的元素后移，然后更新index位置的值
    if (currentSize + 1 <= array.length) {
        System.arraycopy(array, index, array, index + 1, currentSize - index);
        array[index] = element;
        return array;
    }
    //数组空间不足时，先申请一个新的数组（大小为原数组的2倍）
    @SuppressWarnings("unchecked")
    T[] newArray = ArrayUtils.newUnpaddedArray((Class<T>)array.getClass().getComponentType(),
            growSize(currentSize));
    //复制 index 之前的元素
    System.arraycopy(array, 0, newArray, 0, index);
    //新元素插入到 index 位置
    newArray[index] = element;
    //复制原数组index之后的元素
    System.arraycopy(array, index, newArray, index + 1, array.length - index);
    return newArray;
}
```

扩容后的大小使用 `growSize()` 方法计算，源码如下:

```java
public static int growSize(int currentSize) {
    return currentSize <= 4 ? 8 : currentSize * 2;
}
```

可以看到扩容后的容量为数组当前容量的二倍。

至此 SparseArray 源码中的最精华的部分已经分析完了。

## get

get 方法就比较简单了，直接通过二分查找法来查找 key 是否在 `mKeys` 数组中，如果有则返回`mValues` 数组中相同位置的值，如果没有或者 `mValues` 中对应位置的元素被标记为已删除，就返回默认值（如果没有设置就会返回`null`）。

```java
public E get(int key) {
    return get(key, null);
}
```

```java
public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    //如果没有找到key或者key对应的元素被标记为删除
    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

## remove

```java
public void remove(int key) {
    delete(key);
}
```

移除元素调用了 delete 方法，源码如下：

```java
public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```

出于性能方面的考虑，delete 方法在删除元素后并未立即搬移数据，而只是把`mValues`对应位置的 value 指向 `DELETED`，并将 `mGarbage`设为 true，意味着有必要进行「垃圾回收」了，当下次垃圾回收时，再进行元素的搬移操作。

## size

size 方法返回当前 SparseArray 中的映射的数量，不过在返回之前，如果 `mGarbage` 为 true，则需要先调用 gc 方法进行一次「垃圾回收」，因为在移除元素时并没有更新`mSize`的值，此时 `mSize` 和真实的映射对的数量可能是不一致的。

```java
public int size() {
    if (mGarbage) {
        gc();
    }
    return mSize;
}
```

## gc

进行「垃圾回收」的代码如下：

```java
private void gc() {
    //只处理下标在 mSize 之前的元素 
    int n = mSize;
    //记录有效元素的个数
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;
    for (int i = 0; i < n; i++) {
        Object val = values[i];
        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }
            o++;
        }
    }
    mGarbage = false;
    mSize = o;
}
```

代码也很好理解，思路就是把所有不是 DELETED 的元素都替换到数组的前半部分中是 DELETED 的元素，同时也更新 `mKeys` 中的对应的 key，然后把后半部分的元素都赋值为null并把 mGarbage 设为 false，最后更新 mSize 的值。

这里用到了典型的双指针法，在 LeetCode 的算法题 [26. 删除排序数组中的重复项](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/) 中，移动重复元素时也可以采用相同的逻辑。

> 如果我是面试官，我会先让候选人写一遍这个算法题，再问他 SparseArray 的源码，哈哈。

## clone

SparseArray 实现了 Cloneable 接口， clone 方法同样值得关注下，因为这里涉及深拷贝和浅拷贝的问题。

```java
public SparseArray<E> clone() {
    SparseArray<E> clone = null;
    try {
        clone = (SparseArray<E>) super.clone();
        clone.mKeys = mKeys.clone();
        clone.mValues = mValues.clone();
    } catch (CloneNotSupportedException cnse) {
        /* ignore */
    }
    return clone;
}
```

通过代码可以看到对 mKeys 和 mValues 直接调用了数组的 clone 方法，那对数组调用 clone 是深拷贝还是浅拷贝呢？

其实是浅拷贝，也就是重新创建了数组，但两个数组中的元素指向的对象是一致的。。关于这一点可以写几行代码验证下，这里就不过多赘述了。

SparseArray 还提供了一些其他方法，比如append、clear、valueAt等，逻辑都比较简单，有没有特别大的研究价值，所以就不浪费篇幅展示了。

## 总结

SparseArray 用于建立整数到对象的映射，，针对移动开发进行了优化，主要是想解决 HashMap 存在的自动装箱以及内存占用的问题，但并不适用于数据量较大的情况。

SparseArray 内部通过两个数组 mKeys 和 mValues 来分别存储 key 和 value，其中mKeys是一个有序整数数组，在查找 key 时使用了二分查找。

在添加元素时，借助二分查找在未找到元素时 lo 指针指向第一个大于该元素的下标的特性，直接将元素直接插入到对应位置，保证了 mKeys 的有序性。

在删除元素时，只是将 mValues 数组中对应位置的元素指向 DELETED 常量标记为已删除，在需要时（插入元素或调用size）通过 gc 方法来移除被删除元素。

SparseArray 代码虽然不多，但还是有很多值得研究的地方，比如申请数组空间时对内存的利用、维护有序数组时对二分查找的巧妙利用、删除元素时先标记再统一移除的方式等，甚至移除已删除元素的过程还可以直接指导我们做算法题，可以说是短小精悍了啊。