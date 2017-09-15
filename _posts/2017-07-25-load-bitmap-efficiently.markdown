---
layout: post
title:  "Bitmap 的高效加载"
date:   2017-08-04 15:43:39 +0800
tags: [UsingCorrectly,Code]
comments: true
subtitle: "解锁新分类"
---
>这篇博客来自我读 《Android 开发艺术探索》 第 12 章《Bitmap 的加载和 Cache》时的笔记，我对内容进行了一些整理和扩展。    

在实际的开发工作中，为了提高工作效率，图片的加载通常都使用开源框架完成，比如 [Bump Technologies](https://github.com/bumptech) 的 [Glide](https://github.com/bumptech/glide)、[Square](https://github.com/square)  的 [Picasso](https://github.com/square/picasso)、[Facebook](https://github.com/facebook) 的 [Fresco](https://github.com/facebook/fresco)。   

会使用这些框架还只是表面功夫，我们还得了解 `Bitmap` 加载时可能会发生的问题以及解决办法。

在进入正题之前，先来简单了解一下 Bitmap。
## 什么是 Bitmap
Bitmap（位图）是一种通过像素点阵来保存图片信息的文件，它为每一个像素分配特定的位置和颜色值信息。根据位深度（每个像素使用多少位二进制位来保存信息），可将位图分为1、4、8、16、24 及 32 位图像等。   

每个像素使用的信息位数越多，可用的颜色就越多，颜色表现就越逼真，相应的数据量越大。这样在同等图片尺寸下，位深度越大，Bitmap 文件占用空间越大。   
Bitmap 文件(`.bmp`)一般不直接用于网络传输而是使用 `.jpg` 等压缩格式。  

另外，Bitmap 在放大后会有失真现象。与之对应的另外一种图片格式叫 [矢量图](https://zh.wikipedia.org/wiki/%E7%9F%A2%E9%87%8F%E5%9B%BE%E5%BD%A2)，它用点、直线或者多边形等基于数学方程的几何图元来表示图像，这样图像放大后就不会出现失真。
## Android 中如何加载 Bitmap
在 Android 中，主要是通过 [BitmapFactory](https://developer.android.google.cn/reference/android/graphics/BitmapFactory.html) 工厂类来加载 `Bitmap`，它提供了多个静态方法，用于从不同来源加载 `Bitmap`.
- 从字节数组中加载
    - `BitmapFactory.decodeByteArray(byte[] data, int offset, int length,BitmapFactory.Options opts)`
    - `BitmapFactory.decodeByteArray(byte[] data, int offset, int length)`
- 从文件中加载
    - `BitmapFactory.decodeFile(String pathName)`
    - `BitmapFactory.decodeFile(String pathName, BitmapFactory.Options opts)`
- 从应用资源中加载
    - `BitmapFactory.decodeResource(Resources res, int id)`
    - `BitmapFactory.decodeResource(Resources res, int id, BitmapFactory.Options opts)`
- 从输入流中加载
    - `BitmapFactory.decodeStream(InputStream is)`
    - `BitmapFactory.decodeStream(InputStream is, Rect outPadding, BitmapFactory.Options opts)`   
- 从 FileDescriptor 中加载
    - `BitmapFactory.decodeFileDescriptor(FileDescriptor fd)`
    - `BitmapFactory.decodeFileDescriptor(FileDescriptor fd, Rect outPadding, BitmapFactory.Options opts)`  
    >`FileDescriptor` 用来表示打开的流、字节或者 `Socket`，一般通过 `FileInputStream` 和 `FileOutputStream` 来创建。  比如：`FileDescriptor fd = fileInputStream.getFD();`   
  
示例代码：   
```java
/**
* 从应用资源中加载 bitmap
*/
public void setImgFromResource() {
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.she);
    mImageView.setImageBitmap(bitmap);
}

/**
* 从输入流中加载 bitmap
*/
public void setImgFromInputStream() {
    InputStream imgStream = getResources().openRawResource(R.raw.lala_land);
    Bitmap bitmap = BitmapFactory.decodeStream(imgStream);
    mImageView.setImageBitmap(bitmap);
}
//其他两种省略
```
按照上面这样的方式加载 `Bitmap` 会遇到一个问题：当图片太大时，会占用较多的内存，而 Android 系统对每个应用所用内存的大小有限制，如果不经任何处理的加载多张 `Bitmap`，应用就会崩掉，然后会有一条异常信息：   
`java.lang.OutofMemoryError:bitmap size exceeds VM budget.`   
就是说 Bitmap 所占内存超过了虚拟机分配的内存。      
那该如何减少甚至避免这种情况发生呢？  
## 高效加载 Bitmap
仔细看上面列出的 `BitmapFactory` 的五种静态方法，每一种都有一个带有 `BitmapFactory.Options` 参数的重载方法。这个 [BitmapFactory.Options](https://developer.android.google.cn/reference/android/graphics/BitmapFactory.Options.html) 决定了`BitmapFactory` 解析 `Bitmap` 的结果。它有两个重要的字段 `inSampleSize` 和 `inJustDecodeBounds` 可以帮我们解决 `Bitmap` 占用内存过多的问题。
- `inSampleSize`  
    这个值如果设置成大于 `1`，那么`BitmapFactory` 解析后的图片将是原图的缩放版本。  
    例如：设置 `inSampleSize = 2`，那么解析图片的宽和高是原图的 `1/2`,大小就是原图的`1/4`。  
    官方文档中还有这么一句：
    >Note: the decoder uses a final value based on powers of 2, any other value will be rounded down to the nearest power of 2.    
       
    也就是 `BitmapFactory` 最终采用的 `inSampleSize` 的值是 `2` 的幂次。如果设置成其他数字那将采用小于这个数的 `2` 的正数次幂。   
    例如把 `inSampleSize` 设置成`5`，最终采用的将会是 `4`.  
- `inJustDecodeBounds`   
如果这个参数设置为 `true`，`BitmapFactory` 只会解析图片的宽高信息，并赋值给 `BitmapFactory.Options` 的 `outWidth` 和 `outHeight` ；设置为 `false` 时，`BitmapFactory` 才会真正的解析图片。
>注意：`BitmapFactory` 获取的图片的尺寸信息受运行设备的屏幕密度、图片所在资源目录影响。
- 使用套路    
了解了上面两个参数，再借助 `BitmapFactory`，就可以施展我们的加载套路了：  
    1. 将 `BitmapFactory.Options` 的 `inJustDecodeBounds` 设为 `ture`，通过 `BitmapFactory` 解析图片尺寸信息  
    2. 从 `BitmapFacory.Options` 中取出 `outWidth` 和 `outHeight`
    3. 根据目标 `View` 的大小，计算出 `inSampleSize`
    4. 为 `BitmapFacory.Options` 的 `inSampleSize` 赋值，并将 `inJustDecodeBounds` 设为 `false`，开始解析图片     

示例代码：
```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    //1.解析图片尺寸信息
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(getResources(), R.mipmap.she, options);
    //2.获取图片尺寸信息
    int imgWidth = options.outWidth;
    int imgHeight = options.outHeight;
    //3.获取控件大小
    int imgViewHeight = mImageView.getHeight();
    int imgViewWidth = mImageView.getWidth();
    //3.计算 inSampleSize
    int widthInSampleSize = imgWidth / imgViewWidth;
    int heightInSampleSize = imgHeight / imgViewHeight;
    //取二者中较小值
    int inSampleSize = widthInSampleSize > heightInSampleSize ? heightInSampleSize : widthInSampleSize;
    //4.设置inSampleSize和inJustDecodeBounds,解析图片
    options.inSampleSize = inSampleSize;
    options.inJustDecodeBounds = false;
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.she, options);
    mImageView.setImageBitmap(bitmap);
    }
```
>注意：  
1. `inSampleSize` 应该选择宽高缩放比例中较小的一个。例如：`ImageView` 尺寸为 `100 * 100`，而图片尺寸为 `200 * 400`，宽的缩放比例为 `2`，高的缩放比例为 `4`。如果 `inSampleSize` 设置为4，那么加载后的图片大小就是 `50 * 100` ,这样图片比 `ImageView` 小，就会被拉伸，导致图片失真；如果 `inSampleSize` 设为 `2`，加载的图片大小为 `100 * 200`，这样就不会被拉伸。  
2. 方便起见，这里通过 `decodeResource()` 来加载图片，其他来源与此相同。

## Bitmap 缓存策略
缓存的主要目的就是为用户节省流量，同时提高加载速度，提升用户体验——同一个图片如果已经通过网络加载过，那么就将它保存在内存或存储设备上，这样当用户再次请求该图片时，先从内存或者存储设备上获取，如果没有在通过网络加载。    

当然，不只是图片，其他文件（比如音乐和视频文件）同样也可以通过这种方式缓存。  

通常缓存区会设定成一定的大小，不能占用过多空间。这样当存储空间满了的时候就会面对一个问题：如何处理新添加的数据与已经缓存了的数据。 

一种常用的算法是 `LRU(Least Recently Used)` 算法，总体思想就是当存储空间快要占满的时候，优先删除那些近期最少使用的缓存对象。   

采用 `LRU` 算法的缓存工具有 `LruCache` 和 `DiskLruCache` ，它们内部都是通过一个 `LinkedHashMap`来管理缓存数据。`LinkedHashMap` 内部通过 `HashMap` 和双向链表来管理对象，这样既能根据 `Key` 进行迅速查找，又可以在访问对象后及时调整元素间的顺序（把最近一次访问的对象放在表头），以保证缓存空间满的时候能优先删除那些最近很少使用的数据。我们可以使用这两个工具实现在运行内存和外部存储中缓存文件。
### [LruCache](https://developer.android.google.cn/reference/android/util/LruCache.html)  
`LruCache` 是在 `Android 3.1(API level 12)`时添加的 API，用于在 `内存` 中进行数据缓存。内部使用 `LinkedHashMap` 通过 `强引用` 的方式保存 `value`。每当一个 `value` 被访问，它就会被放到队头，当缓存区满时，处于队尾的 `value` 就会被回收。  

- 初始化
```java
private void initLruCache() {
    //获取应用运行内存，单位转成 KB
    int runMemory = (int) Runtime.getRuntime().maxMemory() / 1024;
    //设置缓存区大小
    int cacheSize = runMemory / 8;
    //初始化 LruCache,指定最大缓存容量
    mLruCache = new LruCache<Integer, Bitmap>(cacheSize) {
         //重写计算每个 value 所占空间的方法
        @Override
        protected int sizeOf(Integer key, Bitmap value) {
            return value.getByteCount() / 1024;
        }

        //value 被移除时会调用此方法，如果 value 引用了资源，可以在这里释放
        @Override
        protected void entryRemoved(boolean evicted, Integer key, Bitmap oldValue, Bitmap newValue) {
            super.entryRemoved(evicted, key, oldValue, newValue);
        }
    };
}
```
- 使用
```java
public void loadImgFromResource() {
    int key = 1;
    //从缓存中取 bitmap
    Bitmap bitmap = mLruCache.get(key);
    if (bitmap == null) {
        //如果不存在，则进行加载
        bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.she);
        //将加载好的图片进行缓存
        mLruCache.put(key, bitmap);
    }
    //为 ImageView 设置图片
    mImageView.setImageBitmap(bitmap);
}
```
> `LruCache` 还提供了 `remove(Object key)` 来根据`key`手动删除缓存数据。   

### [DiskLruCache](https://github.com/JakeWharton/DiskLruCache)   
`DiskLruCache` 并不是 Android SDK，但是获得了谷歌官方推荐。它的作者是 [Jake Wharton](https://github.com/JakeWharton)。   
它用来在`外部存储设备`上进行缓存，实现原理与 `LruCache` 类似，不过在 `DiskLruCache` 中，一个 `key` 值可以对应多个 `value`。 另外，`key` 的长度不能超过 `64`个字符并且只能包含 `a-z`、`-`、`_`以及`0-9`。    

`DiskLruCache` 提供了 `edit` 方法用于写入缓存，`get` 方法用于读取缓存。
     
这里通过一个加载图片的小 demo 来演示 `DiskLruCache` 的使用。

- `loadImg()` ：通过一个 `url` 加载图片并设置给 `ImageView`
```java
public void loadImg(String url) {
    //先尝试从本地缓存加载
    Bitmap bitmap = loadImgFromCache(url);
    if (bitmap != null) {
        //从缓存加载成功，为 ImageView 设置图片
        mImageView.setImageBitmap(bitmap);
    } else {
        //如果本地缓存中没有缓存该图片，再从网络加载
        loadImgFromNetwork(url);
    }
}
```
- `loadImgFromCache(String url)`: 通过 `DiskLruCache` 从缓存加载
```java
private Bitmap loadImgFromCache(String url) {
    Bitmap bitmap = null;
    if (mDiskLruCache == null || mDiskLruCache.isClosed()) {
        //初始化 initDiskLruCache
        initDiskLruCache();
    }
    final String key = keyForUrl(url);
    try {
        DiskLruCache.Snapshot snapshot = mDiskLruCache.get(key);
        if (snapshot != null) {
             InputStream inputStream = snapshot.getInputStream(0);
            bitmap = BitmapFactory.decodeStream(inputStream);
            inputStream.close();
        } else {
            //缓存中没有该 key
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return bitmap;
}
```
`DiskLruCache` 的 `get` 方法用于通过 `key` 来访问 `value` ,会返回一个`Snapshot` 对象，通过 `snapshot.getInputStream(int index)` 来拿到输入流。
> `index` 值是 `key` 所对应的 `value` 的索引，从 `0` 开始。如果初始化 `DiskLruCache` 时指定一个 `key` 对应两个 `value` ,那么`index = 1`时拿到的就是第二个 `value` 的输入流。如果缓存中还没有添加过该`key`，那么 `get` 就会返回 `null`。
- `initDiskLruCache()` : 初始化 `DiskLruCache`
```java
private void initDiskLruCache() {
    // 缓存目录
    File diskCacheDir = new File(getCacheDir(),"bitmap");
    if (!diskCacheDir.exists()){
        diskCacheDir.mkdir();
    }
    / /缓存容量 10M
    long cacheSize = 1024*1024*10;
    try {
        mDiskLruCache = DiskLruCache.open(diskCacheDir,1,1,cacheSize);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
`DiskLruCache` 需要通过静态方法 `open()` 来打开，这个方法需要四个参数: 
   - `File directory`：缓存目录，可以是应用自身目录，也可以是公共目录。注意：多个进程不要使用同一个缓存目录。  
   - `int appVersion`：App 版本，版本变化会清空所有缓存，一般设为 `1`
   - `int valueCount`：一个 `key` 对应的 `value` 数量，一般设为 `1`
   - `long maxSize`：最大缓存容量，单位为 `字节`    

- `loadImgFromNetwork` :通过 `AsyncTask` 从网络中获取图片
```java
private void loadImgFromNetwork(final String url) {
    //创建 AsyncTask 并启动
    new LoadImgTask(this).execute(url);
}
``` 
- `LoadImgTask`：异步加载图片并写入缓存
```java
static class LoadImgTask extends AsyncTask<String, Void, Bitmap> {

    SoftReference<MainActivity> mActivitySoftReference;
    ProgressDialog mDialog;

    LoadImgTask(MainActivity mainActivity) {
         mActivitySoftReference = new SoftReference<>(mainActivity);
    }

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        //显示 loading 对话框
        MainActivity mainActivity = mActivitySoftReference.get();
        if (mainActivity != null) {
            mDialog = new ProgressDialog(mainActivity);
            mDialog.setMessage("loading...");
            mDialog.show();
        }
    }

    @Override
    protected Bitmap doInBackground(String... strings) {
        String url = strings[0];
        MainActivity mainActivity = mActivitySoftReference.get();
        if (mainActivity == null) {
            return null;
        }

        String key = mainActivity.keyForUrl(url);
        Bitmap bitmap = null;
        try {
            URL imgUrl = new URL(url);
            HttpURLConnection connection = (HttpURLConnection) imgUrl.openConnection();
            InputStream inputStream = connection.getInputStream();
            // 通过 edit() 获得 Editor
            DiskLruCache.Editor edit = mainActivity.mDiskLruCache.edit(key);
            if (edit != null) {
                //通过 Editor 获得 outputStream
                OutputStream outputStream = edit.newOutputStream(0);
                int hasRead;
                byte[] buffer = new byte[2048];
                while ((hasRead = inputStream.read(buffer)) > 0) {
                    //写入缓存
                    outputStream.write(buffer, 0, hasRead);
                }
                outputStream.close();
                //提交写入
                edit.commit();
                inputStream.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            //生成 bitmap
            DiskLruCache.Snapshot snapshot = mainActivity.mDiskLruCache.get(key);
            bitmap = BitmapFactory.decodeStream(snapshot.getInputStream(0));
        } catch (IOException e) {
            e.printStackTrace();
        }
        return bitmap;
    }

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        super.onPostExecute(bitmap);
        mDialog.dismiss();
        MainActivity mainActivity = mActivitySoftReference.get();
        if (mainActivity != null) {
            if (bitmap != null) {
                //网络加载成功，给 ImageView 设置图片
                mainActivity.mImageView.setImageBitmap(bitmap);
            }
        }
    }
}
```
`DiskLruCache` 的 `get` 方法会返回一个 `editor` 对象，通过 `editor.newOutPutStream(int index)` 可以拿到输出流，然后就可以向缓存中写入数据。如果当前 `value` 正在被编辑，也就是 `get` 已经调用，那么 `get` 方法将返回 `null`。数据写入完成后需要调用 `editor.commit()` 进行提交，如果写入出错还可以调用 `editor.abort()`  取消写入。

>1. 如果需要通过`BitmapFactory.Options`来进行缩放，为了避免第一次解析输入流的尺寸后，再一次通过解析输入流获取`Bitmap`为`null`,可以通过`FileDescriptor`来进行尺寸解析：
```java
DiskLruCache.Snapshot snapshot = mDiskLruCache.get(key);
    if(snapshot!=null){
        FileInputStream fileInputStream = (FileINputStream)snapshot.getInputStream(0);
        FileDescriptor fd = fileInputStream.getFD();
        BitmapFactory.Options options = new BitmapFactory.Options();
        //解析尺寸
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFileDescriptor(fd,null,options);
        //计算并设置 inSampleSize
        //....
        //解析图片
        options.inJustDecodeBounds = false;
        Bitmap bitmap = BitmapFactory.decodeFileDescriptor(fd,null,options);
    }
```
2. 由于 `DiskLruCache` 对 key 有限制，所以这里用了一个 `keyForUrl(String url)` 方法来获得 url 的 MD5 值，并以此作为 key    
```java
private String keyForUrl(String url) {
    String key;
    try {
        MessageDigest messageDigest = MessageDigest.getInstance("MD5");
        messageDigest.update(url.getBytes());
        key = bytesToHexString(messageDigest.digest());
    } catch (NoSuchAlgorithmException e) {
        key = url.hashCode() + "";
    }
    return key;
}
//将字节转换成十六进制字符
private String bytesToHexString(byte[] digest) {
    StringBuilder sb = new StringBuilder();
    for (byte digestByte : digest) {
        String hex = Integer.toHexString(0xff & digestByte);
        if (hex.length() == 1) {
            sb.append("0");
        }
        sb.append(hex);
    }
    return sb.toString();
}
``` 

- 其他 API
    - `close()`：关闭已经打开的 `DiskLruCache`,与 `open` 对应，可以在 `Activity` 的 `onDestroy()` 中调用
    - `size()` ：返回当前缓存所占空间大小,单位是字节
    - `flush()` ：同步日志文件，不需要每次写入都调用，可以在 `Activity` 的 `onPause()` 方法中调用
    - `delete()` ：清空缓存
更多关于 `DiskLruCache` 的内容可以通过[源码](https://github.com/JakeWharton/DiskLruCache)以及[这篇博客](http://blog.csdn.net/guolin_blog/article/details/28863651)了解。

为了更好的用户体验，可能还需要同时使用`LruCache`和`DiskLruCache`来进行二级缓存，这样加载图片时，程序先在内存中查找图片 （对应 `LruCache`），如果没有，再到存储设备中查找（对应 `DiskLruChae` ），如果还没有，那就只能通过网络进行加载。   
## 优化列表卡顿
在使用 `RecyclerView`、`ListView` 或者 `GridView` 时，需要同时加载多张图片，如果这时候用户滚动列表过于频繁，就会同时启动许多图片加载任务，造成页面卡顿，滚动不流畅。  
解决这个问题的办法就是通过 `setOnScrollChangedListener()` 为布局添加滚动事件监听，在 `OnScrollChangedListener` 当布局滚动时，停止加载；当布局停止滚动后，再开始加载对应图片。  
以 `ListView` 为例：
```java
listView.setOnScrollListener(new AbsListView.OnScrollListener() {
    @Override
        public void onScrollStateChanged(AbsListView absListView, int scrollState) {
            if (scrollState == AbsListView.OnScrollListener.SCROLL_STATE_IDLE) {//停止滚动
            //开始加载图片
            } else {
            //停止加载图片
            }
    }
});
```
如果使用以上方法后还有明显卡顿，可以在 `AndroidManifest.xml` 中为相应的 `Activity` 开启硬件加速。   

```xml
<activity android:name=".MainActivity"
          android:hardwareAccelerated="true">
</activity>
```
## 总结
`Bitmap` 加载主要会遇到内存占用过大和缓存的问题：通过缩放，可以减少应用内存占用，避免 `OOM` 的出现；通过内存和存储设备缓存，为用户节省流量，提高加载速度；另外还需要注意列表同时加载多张图片时的卡顿问题。  
等有时间我要去研究一下开源框架源码，看看大神们在 Bitmap 加载过程中是怎么做的，那才是真正的正确姿势。（完）
