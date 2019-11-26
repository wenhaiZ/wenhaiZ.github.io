---
layout: post
title: "WebView 使用心得之 Post 请求添加 Header"
date: 2019-11-26 09:05:00 +0800
tags: [Code,Android]
subtitle: "跳出思维的固定模式"
published: true
---

在Android开发中，需要用到 WebView 加载网页时，我们一般会使用一个 URL然后直接通过 WebView 的 [loadUrl](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadUrl(java.lang.String))方法进行加载：
```java
String  webUrl = "http://page.example.com/index.html"
mWebView.loadUrl(webUrl);
```

`loadUrl` 方法发出的是get请求，如果需要为get请求添加Header，可以使用 loadUrl 方法的[重载版本](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadUrl(java.lang.String,%20java.util.Map%3Cjava.lang.String,%20java.lang.String%3E))：  

```java
Map<String,String> headers =  new HashMap<>();
headers.put("key1","value1");
headers.put("key2","value2");
String  webUrl = "http://page.example.com/index.html"
mWebView.loadUrl(webUrl,headers);
```


## 如果是 Post 请求呢？

如果只是一个普通的Post请求，也很简单，因为 WebView 提供了[postUrl](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#postUrl(java.lang.String,%20byte%5B%5D))方法用于发送 Post 请求：
```java
String  webUrl = "http://page.example.com/index.html"
byte[] postData = ....;
mWebView.postUrl(webUrl,postData);
``` 

如果想要给 Post 请求添加 Header 呢？然后我们发现 WebView 并没有提供一个 `postUrl` 的重载版本，所以我们不能简单的通过调用WebView的API来实现了。  

WebView 提供了[loadData](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadData(java.lang.String,%20java.lang.String,%20java.lang.String))方法，用于直接加载html页面：

```java
 String unencodedHtml = "<html><body>'%28' is the code for '('</body></html>";
 String encodedHtml = Base64.encodeToString(unencodedHtml.getBytes(), Base64.NO_PADDING);

 webView.loadData(encodedHtml, "text/html", "base64");
```

所以，我们只需要**通过别的方式来发起网络请求，再把响应的结果（HTML页面）交给 WebView 加载**就行了。   

我们可以用任何一个网络框架来发起携带 Header的Post请求，

