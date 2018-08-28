---
layout: post
title: "在 OkHttp 中进行 Cookie 持久化"
date: 2018-04-22 11:50:00 +0800
tags: [Code,Android]
subtitle: ""
---
>本文简单介绍了 Session 和 Cookie 的概念并详细阐述了在 Android 开发中使用 OkHttp 进行 Cookie 持久化的两种方法。   

## Session 和 Cookie
众所周知，HTTP 协议是一个无状态的协议，也就是说服务器只是针对每次请求进行响应，并不记录请求的状态，两次请求之间理论上是没有什么关联的。但是，出于应用上的考虑，有时候需要服务端记录客户端的状态，比如购物网站记录用户购物车里的商品、一些网站记录用户登录状态等。   

为了应对上述场景，可以在服务端采用 `Session` 的形式。Session 代表客户端与服务端的一次会话，这个会话可以包括多次请求及其响应。Session 可以包含一些用户信息，比如用户名、操作记录等等，它被保存在服务器的数据库或者文件系统中，从而实现了记录客户端状态的功能。   

由于每个客户端都会对应一个 Session，为了标识不同客户端与服务端进行的 Session，服务端会给 Session 设置一个唯一标识，通常是 `SessionId`。每次客户端发起请求，服务端根据客户端发送的 SessionId 去查找对应 Session，从而达到识别用户的目的。 

SessionId 的传递就需要用到 `Cookie`。HTTP 协议在请求头和响应头中都设定了Cookie 信息，请求头中使用 “Cookie”，而响应头中使用 “Set-cookie”。   

有了 Cookie，Session 的工作流程就可以这样进行：
1. 服务器在用户第一次访问网站或第一次登录时为客户端创建一个 Session，并通过响应头以`Set-cookie：session_id=12345` 的形式将 SessionId 发送给客户端
2. 客户端将 Cookie 进行保存
3. 客户端以后每次向该网站发送请求时，都在请求头中把 SessionId 以 `Cookie：session_id=12345` 形式发送给服务端。   

如此这般，服务器就可以利用 Cookie 并配合 Session 完成客户端识别。
## 为什么进行 Cookie 持久化
在使用浏览器浏览网页时，浏览器默认进行了 Cookie 的持久化保存，当用户关闭浏览器后再次访问相同网站时，浏览器依然可以读取 Cookie 信息并附加到请求头中与服务器进行通信。   

而在 App 中，OkHttp 默认不进行 Cookie 持久化操作，当用户退出 App 后再打开时，就需要用户重新登录，用户体验很差（解决此类问题还有其他方式，这里只讨论使用 Cookie 的情况），因此需要将服务器返回的 Cookie 信息进行持久化保存。

## 方法一：使用拦截器 (Intercepter)
使用 `OkHttp` 进行 Cookie 持久化的操作非常容易进行，因为 OkHttp 提供了拦截器（`Intercepter` 接口）用于对请求和响应进行预处理。   

因此，此种方法的思路就是在收到响应时读取请求头中的 Cookie 信息并进行持久化保存（本例采用了 `SharedPreferences`），然后在请求发送前读取保存的 Cookie 信息并附加到请求头中。  

首先，需要创建两个拦截器，一个用于保存 Cookie，一个用于读取 Cookie。

### SaveCookiesInterceptor
`SaveCookiesInterceptor` 用于从响应头中获取 Cookie，并以字符串的形式保存在 SharedPreferences 中，这些操作需要在 `intercept` 方法中完成，代码如下：
```java
public class SaveCookiesInterceptor implements Interceptor {

    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        //获取请求及其响应
        Request request = chain.request();
        Response response = chain.proceed(request);
        if (!response.headers("set-cookie").isEmpty()) {
            //获取响应头“set-cookie”值，即为服务器发送的 Cookie
            List<String> cookies = response.headers("set-cookie");
            //将 Cookie 拼接成字符串
            String cookie = encodeCookie(cookies);
            //保存 Cookie 字符串
            saveCookie(request.url().host(), cookie);
        }
        return response;
    }
}
```  

其中 `encodeCookie`方法用于将 Cookie 整合成一个字符串，其代码如下：
```java
private String encodeCookie(List<String> cookies) {
    StringBuilder sb = new StringBuilder();
    Set<String> set = new HashSet<>();
    for (String cookie : cookies) {
        String[] arr = cookie.split(";");
        for (String s : arr) {
            set.add(s);
        }
    }
    for (String cookie : set) {
        sb.append(cookie).append(";");
    }
    sb.deleteCharAt(sb.lastIndexOf(";"));
    return sb.toString();
}
```

在 `saveCookie` 方法中，将 Cookie 字符串以 host 为键保存在 SharedPreferences 中：  
```java
private void saveCookie(String host, String cookie) {
    SharedPreferences.Editor editor = MyApplication.getAppContext()
            .getSharedPreferences("cookie_oref", Context.MODE_PRIVATE)
            .edit();
    if (!TextUtils.isEmpty(host)) {
        editor.putString(host, cookie);
    }
    editor.apply();
}
```

### LoadCookiesInterceptor
`LoadCookiesInterceptor` 用于加载本地保存的 Cookie 信息并将其附加到请求头中，其操作同样需要在 `intercept` 方法中完成：
```java
public class LoadCookiesInterceptor implements Interceptor {

    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        Request request = chain.request();
        //获取请求构造器
        Request.Builder builder = request.newBuilder();
        //读取本地 Cookie 信息
        String cookie = loadCookie(request.url().host());
        if (!TextUtils.isEmpty(cookie)) {
            //将 Cookie 添加到请求头中
            builder.addHeader("Cookie", cookie);
        }
        //重新构造请求并进行处理
        return chain.proceed(builder.build());
    }
}
```

`loadCookie` 方法用于从 SharedPreferences 中以 host 为键读取对应的 Cookie 字符串：   
```java
private String loadCookie(String host) {
    SharedPreferences sp = MyApplication.getAppContext()
            .getSharedPreferences("cookie_oref", Context.MODE_PRIVATE);
    if (!TextUtils.isEmpty(host) && sp.contains(host)) {
        return sp.getString(host, "");
    }
    return null;
}
```

### 配置 OkHttpClient
有了上述两个拦截器，接下来就是在初始化 `OkHttpClient` 时通过 `OkHttpClient.Builder` 的 `addInterceptor` 方法将两个拦截器实例添加进去：   
```java
private static final OkHttpClient INSTANCE = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                //添加拦截器
                .addInterceptor(new SaveCookiesInterceptor())
                .addInterceptor(new LoadCookiesInterceptor())
                .build();
```

## 方法二：使用 CookieJar
OkHttp 提供了 `CookieJar` 用于对 Cookie 进行管理，CookieJar 提供了 `saveFromResponse` 和 `loadForRequest` 两个方法用于保存和加载 Cookie。因此，使用 OkHttp 进行 Cookie 持久化还可以通过 CookieJar 实现，Cookie 的持久化操作可以在 `saveFromResponse` 方法中可以进行。    

这里提供一个**简单**的例子：在 `saveFromResponse` 方法中把所有 Cookie 拼接成一个以‘#’为分隔符字符串并保存到 `SharedPreferences` 中，然后在 `loadForRequest` 读取该字符串并解析成 Cookie 对象。
### 创建 CookieJar
创建一个类并实现 CookieJar 接口，在对应回调方法中实现 Cookie 的持久化保存和读取：  
```java
public class MyCookieJar implements CookieJar {

    @Override
    public void saveFromResponse(@NonNull HttpUrl url, @NonNull List<Cookie> cookies) {
        String cookieStr = encodeCookie(cookies);
        saveCookie(url.host(), cookieStr);
    }

    @Override
    public List<Cookie> loadForRequest(@NonNull HttpUrl url) {
        List<Cookie> cookies = new ArrayList<>();
        String cookieStr = loadCookie(url.host());
        if (!TextUtils.isEmpty(cookieStr)) {
            //获取所有 Cookie 字符串
            String[] cookieStrs = cookieStr.split("#");
            for (String aCookieStr : cookieStrs) {
                //将字符串解析成 Cookie 对象
                Cookie cookie = Cookie.parse(url, aCookieStr);
                cookies.add(cookie);
            }

        }
        //此方法返回 null 会引发异常
        return cookies;
    }
}
``` 

拼接 Cookie 的操作在 `encodeCookie` 方法中：  

```java
private String encodeCookie(List<Cookie> cookies) {
    StringBuilder sb = new StringBuilder();
    for (Cookie cookie : cookies) {
        //将Cookie转换成字符串
        sb.append(cookie.toString());
        //以#为分隔符
        sb.append("#");
    }
    sb.deleteCharAt(sb.lastIndexOf("#"));
    return sb.toString();
}
``` 


Cookie 的保存和读取对应 `saveCookie` 和 `loadCookie` 方法：

```java
private String loadCookie(String host) {
    SharedPreferences sp = MyApplication.getAppContext()
            .getSharedPreferences("cookie_oref", Context.MODE_PRIVATE);
    if (!TextUtils.isEmpty(host) && sp.contains(host)) {
        return sp.getString(host, "");
    }
    return null;
}

private void saveCookie(String host, String cookie) {
    SharedPreferences.Editor editor = MyApplication.getAppContext()
            .getSharedPreferences("cookie_oref", Context.MODE_PRIVATE)
            .edit();
    if (!TextUtils.isEmpty(host)) {
        editor.putString(host, cookie);
    }
    editor.apply();
}
``` 

### 配置 OkHttpClient
创建完自定义的 CookieJar，只需调用 `OkHttp.Builder` 的 `cookieJar` 方法将自定义的 CookieJar 实例传入即可：  
```java
private static final OkHttpClient INSTANCE = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                .cookieJar(new MyCookieJar())
                .build();
```


## 参考文章
- [Cookie、Session、Token那点事儿](https://www.jianshu.com/p/bd1be47a16c1)