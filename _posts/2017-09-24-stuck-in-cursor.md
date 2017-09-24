---
layout: post
title: "使用 Cursor 时要注意的一件小事"
date: 2017-09-24 14:01:00 +0800
tags: [Develop,Code]
comments: true
subtitle: "记录一个小 bug"
---  
昨天在整理去年的一个日记应用 [Dear Diary](https://github.com/wenhaiz/DearDiary) 时，发现定时提醒总是报错，通过断点调试发现是按天查询日记后获取日记内容时抛了一个异常  

```
CursorIndexOutOfBoundsException: Index -1 requested, with a size of 1
``` 
抛出异常的方法代码如下：
```kotlin
private fun getDiaryFromCursor(cursor: Cursor): Diary {
    return Diary().apply {
        year = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_YEAR)) }
        month = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_MONTH)) }
        dayOfMonth = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_DAY_OF_MONTH)) }
        dayOfWeek = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_DAY_OF_WEEK)) }
        minute = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_MINUTE)) }
        hour = with(cursor) { getInt(getColumnIndex(DiaryDbHelper.COLUMN_HOUR)) }
        title = with(cursor) { getString(getColumnIndex(DiaryDbHelper.COLUMN_TITLE)) }
        content = with(cursor) { getString(getColumnIndex(DiaryDbHelper.COLUMN_CONTENT)) }
    }
}
```
根据异常栈信息，是第一句通过调用 Cursor#getInt() 时抛出的。   
我最初以为是 getColumnIndex 方法有问题，后来发现并不是它的问题。  

通过断点调试时找到执行的方法是在 AbstractWindowedCursor 中的 getInt 中，这个方法的代码如下：
```java
public int getInt(int columnIndex) {
    checkPosition();
    return mWindow.getInt(mPos, columnIndex);
}
```
在 checkPositon 方法中，调用了父类的方法，AbstarctWindowedCursor 的父类是 AbstractCursor，代码如下：
```java
protected void checkPosition() {
    if (-1 == mPos || getCount() == mPos) {
        throw new CursorIndexOutOfBoundsException(mPos, getCount());
    }
}
```
可以看出，当 mPos 为 -1 或者等于记录数量时，就抛出了上面的异常。   

经过研究，这个 mPos 就是数据在 Cursor 中对应的位置，相当于数组的下标，初始值为 -1。

了解这些后，我看了一下我的查询方法：
```kotlin
override fun queryByDay(calendar: Calendar, callBack: DataSource.LoadDiaryCallBack) {
    //....
    val cursor = db.query(DiaryDbHelper.TABLE_NAME, null, selection, null, null, null, DiaryDbHelper.COLUMN_DAY_OF_MONTH)

    if (cursor == null || cursor.count == 0) {
        callBack.onDataNotAvailable()
    }else{ 
        val diary = getDiaryFromCursor(cursor)
        callBack.onDiaryGot(diary)
    }
    cursor.close()
    db.close()
}
```
也就是说，在查到记录后，我直接把 Cursor 传给 getDiaryFromCursor，这时 Cursor 的 mPos 还是 -1，所以就会抛出异常。   

解决的办法很简单，就是在调用 getDiaryFromCursor 前加一句：
```kotlin
cursor.moveToFirst()
```
通过看源码就知道 moveToFirst 做了什么：
```java
public final boolean moveToFirst() {
    return moveToPosition(0);
}
```
而 moveToPositon 的逻辑就是安全的给 mPos 赋值:
```java
public final boolean moveToPosition(int position) {
    // Make sure position isn't past the end of the cursor
    final int count = getCount();
    if (position >= count) {
        mPos = count;
        return false;
    }

    // Make sure position isn't before the beginning of the cursor
    if (position < 0) {
        mPos = -1;
        return false;
    }

    // Check for no-op moves, and skip the rest of the work for them
    if (position == mPos) {
        return true;
    }

    boolean result = onMove(mPos, position);
    if (result == false) {
        mPos = -1;
    } else {
        mPos = position;
    }

    return result;
}
```
所有的 moveXxx 方法最终都会调用 moveToPosition 来完成对 mPos 的修改，修改成功就返回 true，出现意外就返回 false 。

另外，正是由于 mPos 初始值为-1，所以才可以像下面这样通过 moveToNext 方法来完成对 Cursor 的遍历：
```java
while(cursor.moveToNext()){
    //....
}
```
因为 moveToNext() 代码如下：
```java
public final boolean moveToNext() {
    return moveToPosition(mPos + 1);
}
```
第一次调用 moveToNext() 后，mPos 被修改为 0，这样就指向了第一条记录。  

因此，上面的 moveToFirst 也可以改为 moveToNext。 

虽然数据库操作有很多开源框架使用，但是了解一下 Cursor 原理也没什么坏处。   

这篇文章总结起来就是一句话：Cursor 中的位置指针默认为 -1，在使用 Cursor 前，记得通过 moveToXxx 方法把指针移动到你想要的位置。