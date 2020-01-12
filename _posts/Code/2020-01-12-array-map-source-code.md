---
layout: post
title: "ArrayMap 源码解析"
date: 2020-01-12 09:35:00 +0800
tags: [Code,Android]
subtitle: "ArrayMap 是如何高效利用内存的？"
published: true
---

## 简介

ArrayMap 是一个支持泛型的哈希表，位于 android.util 包下，实现了 Map 接口，但它比 HashMap 对内存的利用更有效。

ArrayMap 内部基于数组和二分查找实现，所以查找效率不及 HashMap，适用于少量元素的情况。

为了更好的利用内存，ArrayMap 会缓存已经创建的数组，以避免频繁创建数组引起的垃圾回收。

这篇文章通过分析 [ArrayMap 源码](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/util/ArrayMap.java)的核心部分，来弄清楚 ArrayMap 是如何存储映射对以及如何进行数组缓存的。

## 属性

ArrayMap 的属性主要有下面四个：

```java
// 是否保证 HashCode 唯一
final boolean mIdentityHashCode;
// 保存 HashCode 的数组
int[] mHashes;
// 保存 key 和 value 的数组
Object[] mArray;
// ArrayMap 的映射对数量
int mSize;
```

其中 `mHashes` 是一个整型数组，用于存储所有 key 的哈希值；而 `mArray` 用于存储 key 和 value。

除此之外，ArrayMap 还有一些重要的静态变量和常量：

```java
//是否在并发修改时抛出异常
private static final boolean CONCURRENT_MODIFICATION_EXCEPTIONS = true;

private static final int BASE_SIZE = 4;
//缓存的数组的最大数量
private static final int CACHE_SIZE = 10;

static final int[] EMPTY_IMMUTABLE_INTS = new int[0];
public static final ArrayMap EMPTY = new ArrayMap<>(-1);

//缓存 size 为 4 的数组
static Object[] mBaseCache;
static int mBaseCacheSize;
//缓存 size 为 4*2 的数组
static Object[] mTwiceBaseCache;
static int mTwiceBaseCacheSize;
```

其中 `mBaseCache` 和 `mTwiceBaseCache` 用于缓存已经创建过的数组，至于如何缓存下面会详细介绍，先来看看 ArrayMap 的构造方法。

## 构造方法

ArrayMap 暴露了三个构造方法，如下面代码所示，最终都会调用到有两个参数的构造方法，但是该构造方式被 `@hide` 标记了，因此不能直接调用。通过构造函数的调用关系可以看到，`mIdentityHashCode` 始终是`false`的，也就是允许哈希冲突。

```java
public ArrayMap(ArrayMap<K, V> map) {
    //调用无参构造方法
    this();
    if (map != null) {
        putAll(map);
    }
}

public ArrayMap() {
    this(0, false);
}

public ArrayMap(int capacity) {
    this(capacity, false);
}

/** {@hide} */
public ArrayMap(int capacity, boolean identityHashCode) {
    mIdentityHashCode = identityHashCode;

    // If this is immutable, use the sentinal EMPTY_IMMUTABLE_INTS
    // instance instead of the usual EmptyArray.INT. The reference
    // is checked later to see if the array is allowed to grow.
    if (capacity < 0) {
        mHashes = EMPTY_IMMUTABLE_INTS;
        mArray = EmptyArray.OBJECT;
    } else if (capacity == 0) {
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
    } else {
        allocArrays(capacity);
    }
    mSize = 0;
}
```

在构造方法中，除了对`capacity <=0` 的情况进行特殊处理外，主要是调用了 `allocArrays` 方法来创建数组并赋值给`mHashes`和 `mArray`。

 allocArrays 方法代码如下：

```java
private void allocArrays(final int size) {
    if (mHashes == EMPTY_IMMUTABLE_INTS) {
        throw new UnsupportedOperationException("ArrayMap is immutable");
    }
    //优先利用缓存的数组
    if (size == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCache != null) {
                final Object[] array = mTwiceBaseCache;
                mArray = array;
                mTwiceBaseCache = (Object[])array[0];
                mHashes = (int[])array[1];
                //将前两个元素置为空
                array[0] = array[1] = null;
                mTwiceBaseCacheSize--;
                return;
            }
        }
    } else if (size == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCache != null) {
                final Object[] array = mBaseCache;
                mArray = array;
                mBaseCache = (Object[])array[0];
                mHashes = (int[])array[1];
                array[0] = array[1] = null;
                mBaseCacheSize--;
                return;
            }
        }
    }
    //没有缓存的数组或者缓存的数组长度不满足条件
    mHashes = new int[size];
    //mArray 的容量是 size 的 2 倍
    mArray = new Object[size<<1];
}
```

乍一看 allocArrays 的前面一部分代码可能会有点懵，可以先不要陷在细节里面。这里我们只要知道，如果要 ArrayMap 的容量是`BASE_SIZE`或者`BASE_SIZE`的 2 倍，那么就优先利用已经缓存过的数组，如果没有缓存数组或者申请的数组长度不符合这两种情况，再创建新数组。至于数组时怎么被缓存和复用的，后面会详细解释。

通过最后两行代码可以看到 mArray 的容量是 mHashes 的 2 倍，这和 ArrayMap 如何存储 key 和 value 有关。我们先看 put 方法，随着对 put 方法的研究，所有的疑问都会解开。

## put

put 方法是重写自 Map 接口的，用于存入一个键值对。在 key 存在时会更新 value 的值并返回旧的 value，在 key 不存在时就插入key 和 value 返回 null。

put 方法代码如下：

```java
public V put(K key, V value) {
    final int osize = mSize;
    final int hash;
    int index;
    // 在mHashes数组中查找对应的 hash 值的下标
    if (key == null) {
        hash = 0;
        index = indexOfNull();
    } else {
        //由前面构造方法可以看到，mIdentityHashCode 始终为false，因此 hash=key.hashCode();
        hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
        index = indexOf(key, hash);
    }
    //已经存在与 key 对应的映射，直接更新value并返回旧的value
    if (index >= 0) {
        //index 为 key 的 hash 在 mHashes 数组中的下标， 
        //在 mArray 数组中，键所在的下标为index*2,值所在的下标为index*2+1
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value;
        //返回旧值
        return old;
    }
    //没有找到对应的key，执行插入
    //index 为应该插入的位置
    index = ~index;
    if (osize >= mHashes.length) {
        //数组空间不足，需要先扩容
        //扩容策略为 如果当前大小大于 BASE_SIZE*2=4*2=8,那么扩容为原来的1.5倍
        //如果 当前大小小于8但是大于4，那么扩容后数组大小为8；
        //如果当前大小小于4，那么扩容为 4
        final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        
        //通过 allocArrays 创建数组并赋值给 mHashes 和 mArray
        allocArrays(n);

        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            //存在并发修改，抛出异常
            throw new ConcurrentModificationException();
        }

        if (mHashes.length > 0) {
            //将值从旧数组拷贝到新数组
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }
        //释放数组空间
        freeArrays(ohashes, oarray, osize);
     }

    if (index < osize) {
       //在数组中间插入，需要移动插入位置后面的元素
        System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }

    if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
        if (osize != mSize || index >= mHashes.length) {
            //存在并发修改，抛出异常
            throw new ConcurrentModificationException();
        }
    }
    //插入新值到 mHashes 和 mArray
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    //mSize + 1
    mSize++;
    return null;
}
```

put 方法的主要逻辑为：**先根据 key 的哈希值在 mHashes 数组中查找这个哈希值是否存在，如果存在且 mArray 对应的位置也存在该 key，那么更新 value 并返回旧的 value；否则，就执行插入，必要时要进行数组扩容。**

put 方法有点长，并且里面有很多细节，我们一段一段的看。

### 查找为 null 的 key

对于 key 为 null 时的查找，调用了 `indexOfNull` 方法，该方法代码如下：

```java
int indexOfNull() {
    final int N = mSize;

    // 如果没有数据，直接返回
    if (N == 0) {
        return ~0;
    }
    // 在 mHashes 数组的 0~N-1 范围内，使用二分查找法查找 0 是否存在
    int index = binarySearchHashes(mHashes, N, 0);

    // 没有找到
    if (index < 0) {
        return index;
    }

    // mHashes 数组中存在 0
    // 并且 mArray 对应的位置 key 也是 null，返回该下标
    if (null == mArray[index<<1]) {
        return index;
    }

    // mHashes 数组中存在 0，但 mArray 对应的位置 key 不是 null
    // 存在哈希冲突，继续向后查找
    int end;
    for (end = index + 1; end < N && mHashes[end] == 0; end++) {
        if (null == mArray[end << 1]) return end;
    }

    // mHashes 数组中存在 0，但 mArray 对应的位置 key 不是 null
    // 存在哈希冲突，继续向前查找
    for (int i = index - 1; i >= 0 && mHashes[i] == 0; i--) {
        if (null == mArray[i << 1]) return i;
    }

    // 没有找到，把第一个等于 hash 的下标取反后返回
    return ~end;
}
```

代码中先调用 `binarySearchHashes` 通过二分查找法在`mHashes`中查找是否存在 0\(null 对应的哈希值\)**，**二分查找方法 `binarySearchHashes` 代码如下：

```java
private static int binarySearchHashes(int[] hashes, int N, int hash) {
    try {
        return ContainerHelpers.binarySearch(hashes, N, hash);
    } catch (ArrayIndexOutOfBoundsException e) {
        //CONCURRENT_MODIFICATION_EXCEPTIONS=true
        if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
            throw new ConcurrentModificationException();
        } else {
            throw e; // the cache is poisoned at this point, there's not much we can do
        }
    }
}
```

可以看到调用了 `ContainerHelpers.binarySearch` ，该方法源码如下：

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

关于这个方法在[分析 SparseArray 时](sparse-array-source-code/#put)已经解释过它的巧妙之处，当没有找到目标值时，会将**第一个大于目标值的下标取反后返回，这样调用方对返回结果再次取反后就可以得到这个下标值，这样做同时也可以保证 mHashes 是有序的。**

如果在 `mHashes` 数组中找到了hash，但是在 mArray 中的对应位置没有找到key，那么还需要进一步进行搜索，为了阅读方便，我把这部分代码再贴一遍：

```java
// mHashes 数组中存在 0，但 mArray 对应的位置 key 不是 null
// 存在哈希冲突，继续向后查找
int end;
for (end = index + 1; end < N && mHashes[end] == 0; end++) {
    if (null == mArray[end << 1]) return end;
}

// mHashes 数组中存在 0，但 mArray 对应的位置 key 不是 null
// 存在哈希冲突，继续向前查找
for (int i = index - 1; i >= 0 && mHashes[i] == 0; i--) {
    if (null == mArray[i << 1]) return i;
}

// 没有找到，把第一个等于 hash 的下标取反后返回
return ~end;
```

我们可以举个例子来捋一下上面代码的逻辑，假设 mHashes 中的元素为\[-2,-1,0,0,0,0,3,4\]（也就是说有四个 key 的哈希值都是0），要查找的哈希值是0，那么通过二分查找法，回先返回下标3，也就是第二个 0，如果 mArray 中对应位置的 key不是null，那么就会执行上面代码的逻辑，先向后搜索，假设直到最后一个0依然没有在 mArray 中找到 null，那么此时 **end=6,**也就是元素 3 的位置。

接下来向前搜索，假设也没有找到 null，那么此时就返回-7（对6按位取反的结果）。

那为什么不返回第一个等于目标哈希值的下标呢？

因为当插入新 key 时，需要移动插入位置以及其后面的元素，在上面的例子中，如果返回的下标是6，这时只需要移动 3 和 4 两个元素就可以了，如果返回第一个 0 的下标，那么需要移动的元素就是6个，所以这么做是为了减少插入新 key 时向后移动元素的数量。

**综上，对于 key 不存在的情况， indexOfNull 会返回一个负值，这个负值是将 mHashes 中第一个大于 key 的哈希值的下标取反后得到。**

通过上面的代码也可以看出 **ArrayMap 使用了线性探测法处理哈希冲突。**

### 查找不是 null 的 key

对于不为 null 的 key，调用了 `indexOf(key,hash)` 来查找，该方法代码如下：

```java
int indexOf(Object key, int hash) {
    final int N = mSize;

    // 没有数据直接返回
    if (N == 0) {
        return ~0;
    }

    //使用二分查找法查找在 mHashes 数组中查找 hash
    int index = binarySearchHashes(mHashes, N, hash);

    // hash 没有找到，不存在映射对
    if (index < 0) {
        return index;
    }

    // hash 存在且 mArray 对应位置的 key 匹配 
    if (key.equals(mArray[index<<1])) {
        return index;
    }

    // hash 存在但是key 不匹配，继续向后搜索
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }

    // hash 存在但是key 不匹配，继续向前搜索
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }

    // 没有找到符合的key，返回负数。同时把第一个不等于hash的下标返回
    // 以便下次插入时尽量少的移动元素
    return ~end;
}
```

indexOf 和 indexOfNull 的逻辑是一样的，区别就是在比较 key 时是通过 equals 方法来进行的。

### 更新已存在 key 对应的 value

当 index &gt;=0 时，也就是 ArrayMap 中已经存在相同 key 的映射，只需要更新值就可以了，这部分操作对应的代码如下：

```java
if (index >= 0) {
    //index 是 key 的 hashCode 在 mHashes 中的下标 
    //在 mArray 数组中，key 所在的下标为index*2,value 所在的下标为index*2+1
    index = (index<<1) + 1;
    final V old = (V)mArray[index];
    mArray[index] = value;
    //返回旧值
    return old;
}
```

上面几行代码的重点是，**对于 hashCode 在 mHashes 数组中的下标为 index 的 key，对应的value 在 mArray 数组中的下标为 index\*2+1,而 key 在 mArray 中的下标为 index\*2**，这一点从前面对于 key 的搜索逻辑也可以看出来。

举例来说，对于一个 key，如果它的hashCode 在 mHashes 中的下标为 1，那么这个 key 在mArray 中的下标为 1\*2=2，它对应的 value 在 mArray 中的位置为 1\*2+1=3。

我们可以通过插入新值的逻辑再次验证一下，put 方法中对应的代码如下：

```java
//......
//插入新值到 mHashes 和 mArray
mHashes[index] = hash;
// 在 index*2 的位置插入 key
mArray[index<<1] = key;
// 在 index*2+1 的位置插入 value
mArray[(index<<1)+1] = value;
//mSize + 1
mSize++;
```

这样我们就搞清楚了 ArrayMap 到底是怎么存储 hashCode、key 和 value 的，它们之间的关系如下图所示：

![mHashes mArray &#x5143;&#x7D20;&#x5BF9;&#x5E94;&#x5173;&#x7CFB;](/assets/img/post/mhash_marray.jpg)

### 插入新的映射

在 put 方法中，当 key 不存在时，在执行插入前，就有这么一句代码：

```java
index = ~index;
```

前面看到，对于当 key 不存在时，查找时返回了一个负值，**这个负值就是 mHashes 数组中第一个大于 key 的哈希值的下标按位取反后的值**。**这里通过再次取反，就得到了这个下标，也就是key要插入的位置。**

确定了新映射的位置，如果 mHashes 容量充足，直接把新映射的 hashCode、key、value加入到数组中就可以了。

插入新值对应的代码如下：

```java
if (index < osize) {
   //在数组中间插入，需要移动待插入位置以及后面的元素
    System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
    System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
}
// 插入新值到 mHashes 和 mArray
mHashes[index] = hash;
// key 的下标为 index*2
mArray[index<<1] = key;
// value 的下标为 index*2+1
mArray[(index<<1)+1] = value;
// mSize + 1
mSize++;
```

这部分代码比较容易理解，不过在这是数组的大小已经满足需求了。对于不满足的情况，则需要先进行扩容。

### 数组扩容

数组扩容部分的代码如下：

```java
if (osize >= mHashes.length) {
    //数组空间不足，需要先扩容
    final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
            : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

    //保存旧数组
    final int[] ohashes = mHashes;
    final Object[] oarray = mArray;
        
    //通过 allocArrays 创建（复用）数组并赋值给 mHashes 和 mArray
    allocArrays(n);

    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
        //存在并发修改，抛出异常
        throw new ConcurrentModificationException();
    }

    if (mHashes.length > 0) {
        //将值从旧数组拷贝到新数组
        System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
        System.arraycopy(oarray, 0, mArray, 0, oarray.length);
    }
    //释放（缓存）数组空间
    freeArrays(ohashes, oarray, osize);
 }
```

扩容策略为：

1. 如果当前ArrayMap size 大于 8（BASE\_SIZE\*2），那么扩容为原来的1.5倍
2. 如果 size 小于 8 但是大于4（BASE\_SIZE），那么扩容后数组大小为 8
3. 如果 size 小于4，那么扩容为 4。

扩容后的容量大小确定后，通过 allocArrays 方法创建数组并把新数组赋值给 mHashes 和 mArray。

前面在分析构造方法时，已经看到过这个方法，不过只看了大概逻辑，这里仔细分析下，该方法代码如下：

```java
private void allocArrays(final int size) {
    if (mHashes == EMPTY_IMMUTABLE_INTS) {
        throw new UnsupportedOperationException("ArrayMap is immutable");
    }
    //优先利用缓存的数组
    if (size == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCache != null) {              
                final Object[] array = mTwiceBaseCache;
                mArray = array;              
                mTwiceBaseCache = (Object[])array[0];             
                mHashes = (int[])array[1];              
                array[0] = array[1] = null;               
                mTwiceBaseCacheSize--;
                return;
            }
        }
    } else if (size == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCache != null) {
                //1.
                final Object[] array = mBaseCache;
                mArray = array;
                //2.
                mBaseCache = (Object[])array[0];
                //3.
                mHashes = (int[])array[1];
                //4.
                array[0] = array[1] = null;
                //5.
                mBaseCacheSize--;
                return;
            }
        }
    }
    //没有缓存的数组或者缓存的数组长度不满足条件
    mHashes = new int[size];
    //mArray 的容量是 size 的 2 倍
    mArray = new Object[size<<1];
}
```

方法中，针对容量为 BASE\_SIZE 或者 BASE\_SIZE \*2 的情况，优先利用已经缓存的数组，我们以容量为 BASE\_SIZE 时的情况分析，代码逻辑是：

1. 将 mArray 赋值为 mBaseCache
2. 将 mBaseCache 赋值为 mArray\[0\]
3. 将 mHashes 赋值为 mArray\[1\]
4. 将 mArray\[0\] mArray\[1\]的值置空
5. 缓存数量减一

要想弄清楚这段逻辑，就要知道 mBaseCache 究竟存储的究竟是什么，先来看看注释：

```java
/**
* Caches of small array objects to avoid spamming garbage.  The cache
* Object[] variable is a pointer to a linked list of array objects.
* The first entry in the array is a pointer to the next array in the
* list; the second entry is a pointer to the int[] hash code array for it.
*/
static Object[] mBaseCache;
```

说实话，这段注释我看了好几遍依然有点懵。意思是 **mBaseCache 是指向链表的指针，而这个链表是由\(所有被缓存的mArray\)数组组成的。**其中， mBaseCache\[0\] 指向下一个被缓存的数组，mBaseCache\[1\] 指向的当前数组对应的 mHashes 数组。

所以 mBaseCache 里的内容如下图所示：

![mBaseCache &#x7F13;&#x5B58;&#x793A;&#x610F;&#x56FE;](/assets/img/post/arraymap_basecache.png)

> 虽然 BASE\_SIZE 是4，但实际 mBaseCache 缓存的数组容量是8，4是映射对的数量，也就是mHashes 的大小。

对照着图，再看一遍上面的逻辑就很容易理解了，而 mTwiceBaseCache 与 mBaseCache 除了缓存的数组长度不一致以外，其他都是相同的。

那么 mBaseCache 和 mTwiceBaseCache 是什么时候被赋值的呢？一定是回收数组时。在put方法中，在分配完新数组并从旧数组拷贝完元素之后，调用了 `freeArrays` 方法释放旧的数组，该方法代码如下：

### freeArrays

```java
private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
    if (hashes.length == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCacheSize < CACHE_SIZE) {
                //组成链表结构
                array[0] = mTwiceBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mTwiceBaseCache = array;
                mTwiceBaseCacheSize++;
            }
        }
    } else if (hashes.length == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCacheSize < CACHE_SIZE) {
                //1.
                array[0] = mBaseCache;
                //2.
                array[1] = hashes;
                //3.
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                //4.
                mBaseCache = array;
                mBaseCacheSize++;
            }
        }
    }
}
```

可以看到只针对 mHashes 数组长度为 BASE\_SIZE 和 BASE\_SIZE \*2 的情况做了缓存。还是只看 BASE\_SIZE 的情况:

1. 将 array\[1\]指向了对应的 hashes 数组
2. 将 array\[0\] 指向 mBaseCache
3.  array 中下标大于1的元素置空，否则会存在内存泄漏。
4. 在把 mBaseCache 指向 array

其中步骤2和4就把缓存的数组组织成了链表，mBaseCache 始终指向链表头的数组，这一步参照上面画的示意图就比较容易理解了。

以上就是数组被缓存和复用的逻辑，这也是我认为 ArrayMap 最难理解的地方。

put 方法的逻辑分析完了，由于涉及的内容较多，如果一时半会儿弄不清楚可以多看几遍，通过举例画图的方式帮助理解。

## get

get 方法的代码比较简单：

```java
@Override
public V get(Object key) {
    final int index = indexOfKey(key);
    return index >= 0 ? (V)mArray[(index<<1)+1] : null;
}
```

通过 indexOfKey 查找 key 的哈希值在 mHashes 数组中的位置，如果 key 存在，再从 mArray 数组的对应位置取出对应的value。

其中 indexOfKey 代码如下：

```java
public int indexOfKey(Object key) {
    return key == null ? indexOfNull()
            : indexOf(key, mIdentityHashCode ? System.identityHashCode(key) : key.hashCode());
}
```

就是针对 key 是否为 null 去调用 indexOfNull 或者 indexOf 方法。

## remove

remove 方法用于根据 key 来删除一个映射对，这个方法代码如下：

```java
@Override
public V remove(Object key) {
    final int index = indexOfKey(key);
    if (index >= 0) {
        return removeAt(index);
    }
    return null;
}
```

找到 key 的索引后，主要的移除逻辑在`removeAt`方法中：

```java
public V removeAt(int index) {
    final Object old = mArray[(index << 1) + 1];
    final int osize = mSize;
    final int nsize;
    if (osize <= 1) {
        // 移除后为空
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        freeArrays(ohashes, oarray, osize);
        nsize = 0;
    } else {
        nsize = osize - 1;
        if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
            //如果 mHashes 数组长度大于 (BASE_SIZE*2)，并且剩余空间大于2/3,缩减数组长度
            //缩减的数组长度不会小于BASE_SIZE*2，主要是为了利用缓存的数组
            final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);
            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            //重新对 mHashes 和 mArray 赋值
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            if (index > 0) {
                //将待移除位置之前的元素复制到新数组
                System.arraycopy(ohashes, 0, mHashes, 0, index);
                System.arraycopy(oarray, 0, mArray, 0, index << 1);
            }
            if (index < nsize) {
                //将待移除位置之后的元素复制到新数组
                System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                        (nsize - index) << 1);
            }
            //移除完成
        } else {
            //不用缩减数组的情况
            if (index < nsize) {
                //将待删除位置后面的元素前移一位（此时最后一位就是多余元素）
                System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                        (nsize - index) << 1);
            }
            //将最后一个元素置空，完成删除
            mArray[nsize << 1] = null;
            mArray[(nsize << 1) + 1] = null;
        }
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
        throw new ConcurrentModificationException();
    }
    mSize = nsize;
    return (V)old;
}
```

remove 方法的逻辑也比较好理解，**重点就是针对 mHashes 数组大于 BASE\_SIZE\*2并且空间利用率不足1/3的情况，要进行数组缩减**。缩减后的大小不会小于BASE\_SIZE\*2，这主要是为了能优先利用缓存的数组。

ArrayMap 的其他方法代码比较好理解，不是本篇的重点，就不再分析了。

## 总结

ArrayMap 是 Android 提供的工具类，用于存储键值映射，不过比 HashMap 对内存的利用效率更高。一是不需要一个额外的包装类来封装 key 和 value，二是通过对分配过的数组进行缓存，减少了数组的频繁创建和销毁以及因此导致的不必要的垃圾回收。

ArrayMap 内部采用了两个数组来存储映射对：一个整型数组 mHashes 用来存储所有 key 的哈希值，mHashes 是有序的，查找 key 时使用了二分查找，这也是ArrayMap 查找效率低于HashMap的原因；还有一个数组 mArray 用来存储 key 和 value，其中 key 和 value 是成对存储的，它们在 mArray 中的位置 ki,vi 与 key 的哈希值在 mHashes 中的下标 hi 的对应关系为：

```java
ki = hi*2=hi<<1;
vi = ki+1 = hi*2+1=(hi<<1)+1;
```

由于数组在内存中占用的空间时连续的，而随着很多对象的创建和销毁，不能保证内存中始终有足够的连续内存空间用于分配数组。如果这个时候需要给数组分配空间，就只能先进行垃圾回收才可以，如果频繁的创建和销毁数组，势必引发很多不必要的垃圾回收，从而影响应用性能。ArrayMap 通过缓存一定数量已经分配过的数组，保证了在需要数组时可以立即使用，而不用再创建数组甚至进行垃圾回收，有效的提升了应用的性能。

ArrayMap 通过 allocArrays 方法创建数组并赋值给 mHashes 和 mArray，在该方法内部，对于 size=4或者size=8时，会优先利用已经缓存的数组。已经缓存的 mArray 数组被组织成链表的形式，链表头就是 mBaseCache\(size=4时\) 和 mTwiceBaseCache\(size=8时\)，mBaseCache 的第一个元素指向下一个 mArray 数组，第二个元素指向 mBaseCache 对应的 mHashes 数组。

数组的缓存是在 freeArrays 中方法进行的，该方法将 ArrayMap 的容量为4或8时创建的数组组织成链表，由 mBaseCache 和 mTwiceBaseCache 存储。

为了充分利用已经缓存的数组，当数组空间不足进行扩容时，如果当前数组容量小于4，会扩容成4\(可以复用由 mBaseCache 存储的数组\)；如果大于4小于8，会扩容成8\(可以复用由 mTwiceBaseCache 存储的数组\)，如果容量大于8，则会扩容为原容量的 1.5 倍。

当移除映射时，对于 mHashes 长度大于 BASE\_SIZE\*2并且空间利用率小于1/3的情况，会缩小数组长度，主要也是为了能及时释放大数组并充分利用缓存的数组。

如果我们在应用中多使用 ArrayMap，那么就能充分利用已经申请过的数组，减少由于数组的频繁创建和销毁引起的垃圾回收以及由此对应用性能产生的不良影响。
