---
layout: post
title: "Android 中的序列化"
date: 2017-09-18 23:08:00 +0800
tags: [Code,Android]
comments: true
subtitle: "Serializable & Pacelable"
---
>本文介绍了 Android 开发常用的两种序列化方式以及它们之间的区别。

## 什么是序列化 
序列化是将一个对象转换成可存储或可传输的状态（字节流）。   
序列化后的对象可以在网络上进行传输，也可以存储到本地。   
序列化对象的时候只是针对实例变量进行序列化，不针对方法进行序列化。

## 为什么要序列化
- 数据持久化：将对象数据保存在文件中
- 便于数据在网络上传输：由于网络传输是以字节流的方式对数据进行传输的，因此序列化的目的是将对象数据转换成字节流的形式
- 将数据在进程之间传递： Activity 之间传递对象数据时，需要在当前的Activity 中对对象数据进行序列化操作，在另一个 Activity 中需要进行反序列化操作讲数据取出

## 怎样进行序列化
在 Android 中，实现序列化的方式就是让需要序列化的类实现特定的接口，并提供必要的实现，这样的接口有两个：
- Serializable （Java）
    - 优点：易于实现，类只需要实现 Serializable 接口，并指定一个 serialVersionUID (`long` 型，标识当前类的版本，辅助序列化和反序列化，同时防止重复序列化)
    - 缺点：基于反射，会产生大量临时变量，可能会触发GC，效率较低

- Parcelable（Android 特有）  
 原理：Parcelable 方式的实现原理是将一个完整的对象进行分解，而分解后的每一部分都是 Bundle 中所支持的数据类型，这样就实现传递对象的功能。
    - 优点：效率较高，内存开销小，速度是 Serializable 的十几倍
    - 缺点：实现较复杂，Parcelable 无法很好的将数据进行持久化，原因是在不同的 Android 版本当中，Parcelable 可能会不同。
 > 注：实现 Parcelable 时，boolean 类型的值采用整型替代，反序列化时根据整型值恢复。

## 序列化方法的选择
  - 在内存中序列化的时候，Parcelable 比 Serializable 性能好，所以尽量使用 Parcelable
  - 在将数据持久化或进行网络传输的时候，应该使用 Serialzable

## Sample
下面通过两个简单的例子来展示这两种序列化方式的使用。　　　

- 实现 Serializable
```java
class PlayStatus implements Serializable {
        //可以不指定
        static long serialVersionUID = 998;
}
```


- 实现 Parcelable 
```java
public class Artist implements Parcelable {
    private String artistId;
    private boolean isChinese;

    protected Artist(Parcel in) {
        artistId = in.readString();
        //boolean 类型的反序列化
        isChineses = in.readInt() == 1;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(artistId);
        //boolean 类型的序列化
        dest.writeInt(isChinese ? 1 : 0);
    }

    @Override
    public int describeContents() {
        //默认返回0
        return 0;
    }
    //固定模式
    public static final Creator<Artist> CREATOR = new Creator<Artist>() {
        //创建一个实例
        @Override
        public Artist createFromParcel(Parcel in) {
            return new Artist(in);
        }

        //创建一个数组
        @Override
        public Artist[] newArray(int size) {
            return new Artist[size];
        }
    };
}
```
