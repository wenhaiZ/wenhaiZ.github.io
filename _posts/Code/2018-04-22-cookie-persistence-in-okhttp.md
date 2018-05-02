---
layout: post
title: "在 OkHttp 中进行 Cookie 持久化"
date: 2018-04-22 11:50:00 +0800
tags: [Code,Android]
subtitle: ""
code-link: "assets/code/180422.md"
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
![code01](/assets/img/post/code/180422_01.png)

其中 `encodeCookie`方法用于将 Cookie 整合成一个字符串，其代码如下：
![code02](/assets/img/post/code/180422_02.png)

在 `saveCookie` 方法中，将 Cookie 字符串以 host 为键保存在 SharedPreferences 中：  
![code03](/assets/img/post/code/180422_03.png)

### LoadCookiesInterceptor
`LoadCookiesInterceptor` 用于加载本地保存的 Cookie 信息并将其附加到请求头中，其操作同样需要在 `intercept` 方法中完成：
![code04](/assets/img/post/code/180422_04.png)  

`loadCookie` 方法用于从 SharedPreferences 中以 host 为键读取对应的 Cookie 字符串：   
![code05](/assets/img/post/code/180422_05.png) 

### 配置 OkHttpClient
有了上述两个拦截器，接下来就是在初始化 `OkHttpClient` 时通过 `OkHttpClient.Builder` 的 `addInterceptor` 方法将两个拦截器实例添加进去：   
![code06](/assets/img/post/code/180422_06.png) 

## 方法二：使用 CookieJar
OkHttp 提供了 `CookieJar` 用于对 Cookie 进行管理，CookieJar 提供了 `saveFromResponse` 和 `loadForRequest` 两个方法用于保存和加载 Cookie。因此，使用 OkHttp 进行 Cookie 持久化还可以通过 CookieJar 实现，Cookie 的持久化操作可以在 `saveFromResponse` 方法中可以进行。    

这里提供一个**简单**的例子：在 `saveFromResponse` 方法中把所有 Cookie 拼接成一个以‘#’为分隔符字符串并保存到 `SharedPreferences` 中，然后在 `loadForRequest` 读取该字符串并解析成 Cookie 对象。
### 创建 CookieJar
创建一个类并实现 CookieJar 接口，在对应回调方法中实现 Cookie 的持久化保存和读取：  
![code07](/assets/img/post/code/180422_07.png) 

拼接 Cookie 的操作在 `encodeCookie` 方法中：  

![code08](/assets/img/post/code/180422_08.png) 

Cookie 的保存和读取对应 `saveCookie` 和 `loadCookie` 方法：

![code09](/assets/img/post/code/180422_09.png) 

### 配置 OkHttpClient
创建完自定义的 CookieJar，只需调用 `OkHttp.Builder` 的 `cookieJar` 方法将自定义的 CookieJar 实例传入即可：  
![code10](/assets/img/post/code/180422_10.png) 


## 参考文章
- [Cookie、Session、Token那点事儿](https://www.jianshu.com/p/bd1be47a16c1)