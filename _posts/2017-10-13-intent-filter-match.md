---
layout: post
title: "IntentFilter 匹配规则"
date: 2017-10-14 14:15:00 +0800
tags: [Code,Develop]
subtitle: "action/category/data"
---
在 Android 中可以通过显示和隐式 Intent 来启动 Activity、Service 和BroadcastReceiver。    

在使用隐式 Intent 时，需要为 Intent 指明 Action、Category 和 Data 等附加信息，这些信息用来匹配对应组件的 IntentFilter。

IntentFilter 在 `AndroidManifest.xml` 文件中的 Activity/Service/BroadcastReceiver 标签内使用 `<intent-filter>` 声明，可以包含如下三种信息：
- action : 在 name 属性中，声明接受的 Intent 操作，该值是一个字符串值
- category : 在 name 属性中，声明接受的 Intent 类别，该值是一个字符串值
- data : 指定数据 URI （scheme、host、port、path 等）和 MIME 类型。MIME 类型通过 mimetype 属性声明，比如 image/jpeg,video/*等。URI 包含多个部分，可表示为：`<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]`   


例如：默认的 MainActivity 包含如下的 IntentFilter ：
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```
这篇文章着重介绍 IntentFilter 的匹配规则，即隐式 Intent 的附加信息满足什么条件时才会启动对应的组件。

## action 的匹配规则
action 的值是一个字符串，一个 IntentFilter 可以包含多个 action。  

action 匹配规则很简单：只要 Intent 设置的 action 和 IntentFilter 中声明的 action 的其中一个相同，就算匹配成功。

例如在清单文件中声明这样一个 Activity：
```xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.wenhaiz.ACTION_ONE"/>
        <action android:name="com.wenhaiz.ACTION_TWO"/>
        <!-- 必须要有默认的 category -->
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```
它的 IntentFilter 包含两个 action，可以通过下面的代码启动这个 Activity：
```java
Intent intent = new Intent();
intent.setAction("com.wenhaiz.ACTION_ONE");
// intent.setAction("com.wenhaiz.ACTION_TWO"); //同样可以启动
startActivity(intent);
```
## category 匹配规则
category 的匹配规则与 action 不太相同，可以概括为：   
如果 Intent 设置了 category，那么每个 category 都要匹配 IntentFilter 声明的 category 中的一个。从集合的角度来说，Intent 声明的 category 必须是 IntentFilter 声明的 category 的子集。

> 需要注意的是，startActivity 或者 startActivityForResult 方法会为 Intent 加上一个值为 "android.intent.category.DEFAULT" 的 category，因此，在声明 IntentFilter 时，需要加上这个 category。  

例如，在 SecondActivity 的 intent-filter 中声明如下 category：
```xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.wenhaiz.ACTION_ONE"/>
        <action android:name="com.wenhaiz.ACTION_TWO"/>

        <!--必须要有 DEFAULT  -->
        <category android:name="android.intent.category.DEFAULT"/>
        <!-- 两个自定义的 category -->
        <category android:name="com.wenhaiz.CATEGORY_ONE"/>
        <category android:name="com.wenhaiz.CATEGORY_TWO"/>
    </intent-filter>
</activity>
```
那么，启动该 Activity 的代码可以是下面这样：
```java
Intent intent = new Intent();
intent.setAction("com.wenhaiz.ACTION_ONE");
//可以去掉下面两个中的任何一个，也可以不加 category，那么就会匹配默认的 category
intent.addCategory("com.wenhaiz.CATEGORY_ONE");
intent.addCategory("com.wenhaiz.CATEGORY_TWO");
//加上下面的代码就会启动失败
//intent.addCategory("com.wenhaiz.CATEGORY_THREE");
startActivity(intent);
```
## data 匹配规则
开头简单介绍了 data 的结构，下面解释一下每部分的作用：
- scheme: URI 模式，比如 file/http/content.如果 scheme 未指定，整个 URI 无效
- host: 主机名，比如 www.google.com，如果 host 未指定，整个 URI 无效
- port: 端口号，比如 80。仅当 URI 指定了 scheme 和 host 时 port 才有意义
- path/pathPrefix/pathPattern:路径信息。path 表示完整的路径，pathPatter 也表示完整路径，但是可以包含通配符 "*"，pathPrefix 表示路径前缀信息 

例如 URI http://www.google.com:80/search/info 可以表示成如下形式：
```xml
<data
    android:host="www.google.com"
    android:path="/search/info"
    android:port="80"
    android:scheme="http"/>
```

data 的匹配规则可以概括为：   
如果 IntentFilter 中声明了 data 标签，那么隐式 Intent **必须设置** data，并且 IntentFilter 中声明的 data 必须包含在 Intent 设置的 data 中。    

例如 SecondActivity 声明 data 如下：
```xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.wenhaiz.ACTION_ONE"/>

        <category android:name="android.intent.category.DEFAULT"/>

        <data
            android:mimeType="image/*"
            android:host="www.google.com"
            android:path="/search/info"
            android:port="80"
            android:scheme="http"/>

        <data
            android:mimeType="video/*"
            android:port="80"
            android:scheme="http"/>    
    </intent-filter>
</activity>
```
那么启动该 Activity 的代码如下：
```java
Intent intent = new Intent();
intent.setAction("com.wenhaiz.ACTION_ONE");
intent.setDataAndType(Uri.parse("http://www.google.com:80/search/info"),"image/jpeg");
//下面代码同样可以启动该 Activity
//intent.setDataAndType(Uri.parse("http://www.google.com:80/search"),"video/mp4");
startActivity(intent);
```
> 1. 如果 data 中没有指明 URI 的 scheme ,那么默认为 file 或者 content
> 2. 如果需要为 Intent 指定 URI 和 mimeType，需要使用 setDataAndType，单独使用 setData 或者 setType 都会清楚已设置的 type 或着 uri 信息。 
> 3. 可以通过 pathPrefix 来指定路径的同一前缀，例如以 /search 开头的路径可以设置 pathPrefix="/search"

## 启动前检查隐式 Intent 能否成功匹配
对于 Activity ，有两种方法可以做到。
1.  Intent 的 resolveActivity 方法   
```java
Intent intent = new Intent();
intent.setAction("com.wenhaiz.ACTION_ONE");
intent.setDataAndType(Uri.parse("file://www.google.com:80/search"), "video/mp4");
//传入 packageManager
ComponentName componentName = intent.resolveActivity(getPackageManager());
if (componentName == null) {
    Toast.makeText(MainActivity.this, "没有找到对应的Activity", Toast.LENGTH_SHORT).show();
} else {
    startActivity(intent);
}
```
2. PackageManager 的 queryIntentActivities  
```java
Intent intent = new Intent();
intent.setAction("com.wenhaiz.ACTION_ONE");
intent.setDataAndType(Uri.parse("http://www.google.com:80/search"), "video/mp4");
List<ResolveInfo> resolveInfos = getPackageManager()
                        .queryIntentActivities(intent, PackageManager.MATCH_DEFAULT_ONLY);
if (resolveInfos == null || resolveInfos.size() == 0) {
    Toast.makeText(MainActivity.this, "没有找到对应的Activity", Toast.LENGTH_SHORT).show();
} else {
    startActivity(intent);
}
```   

注意，在 queryIntentActivities 方法中传入的第二个参数为 `MATCH_DEFAULT_ONLY` ,也就是只匹配那些声明了值为  `"android.intent.category.DEFALUT"` 的 category 的 Activity，因为不含有这个 category 无法接收隐式 Intent。

对于 Service/BroadcastReceiver ，由于 Intent 没有相关方法，只能通过 PackageManager 的 queryIntentServices 和 queryBroadcastReceivers 方法来检查，用法同上。  

## ref
- 《Android 开发艺术探索》 chapter 1.3