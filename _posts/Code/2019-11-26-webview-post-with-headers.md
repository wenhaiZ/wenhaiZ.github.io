---
layout: post
title: "WebView 使用心得之 Post 请求添加 Header"
date: 2019-11-26 09:05:00 +0800
tags: [Code,Android]
subtitle: ""
published: true
---

在 Android 开发中，需要用到 WebView 加载网页时，我们一般直接通过 WebView 的 [loadUrl](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadUrl(java.lang.String))方法进行加载：
```java
String  webUrl = "http://page.example.com/index.html"
mWebView.loadUrl(webUrl);
```

`loadUrl` 方法发出的是`GET`请求，如果需要为`GET`请求添加`Header`，可以使用 `loadUrl` 方法的[重载版本](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadUrl(java.lang.String,%20java.util.Map%3Cjava.lang.String,%20java.lang.String%3E))：  

```java
Map<String,String> headers =  new HashMap<>();
headers.put("key1","value1");
headers.put("key2","value2");
String webUrl = "http://page.example.com/index.html"
//第二个参数为 header
mWebView.loadUrl(webUrl,headers);
```

那如果请求页面时需要发送 `POST` 请求呢？

如果只是一个普通的`POST`请求，解决起来也很简单，因为 `WebView` 提供了[postUrl](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#postUrl(java.lang.String,%20byte%5B%5D))方法用于通过 `POST` 方式来请求页面：   

```java
String webUrl = "http://page.example.com/index.html"
byte[] postData = ....;
mWebView.postUrl(webUrl,postData);
``` 

那如果想要给 `POST` 请求添加 `Header` 呢？   

然而 WebView 并没有提供一个 `postUrl` 的重载版本，所以我们不能简单的通过调用 `WebView`的方法来实现了。    


这时可以换一种思路，我们使用 `WebView` 的最终目的就是加载一个 `HTML` 页面，至于这个页面由谁请求其实并不重要，而且 WebView 提供了[loadData](https://developer.android.google.cn/reference/android/webkit/WebView.html?hl=en#loadData(java.lang.String,%20java.lang.String,%20java.lang.String))方法，可以用于直接加载现成的 `HTML` 页面，示例代码如下：

```java
 String unencodedHtml = "<html><body>'%28' is the code for '('</body></html>";
 String encodedHtml = Base64.encodeToString(unencodedHtml.getBytes(), Base64.NO_PADDING);

 webView.loadData(encodedHtml, "text/html", "base64");
```

因此，我们完全可以自己发起一个携带 Header 的`POST`请求，然后再把响应得到的 HTML 页面像上面代码那样通过`webView.loadData()`方法来加载就可以了。     


不过，还有更优雅一点的方式，就是通过重写 `WebViewClient` 的 `shouldInterceptRequest` 方法，来拦截WebView 发出的请求。   

对于一般的请求，直接由 WebView 来发起和处理， 而对于需要添加 Header 的 `POST` 请求，我们自己来发起，然后把响应结果包装成`WebResourceResponse`交给 WebView 渲染即可。  

示例代码如下:

```java
public class MyWebViewClient extends WebViewClient{
    @Override
    public WebResourceResponse shouldInterceptRequest(WebView view, String url) {
        //一般请求不处理，只处理需要添加 Header 的 Post请求（通过 url 标识）
        if (!postWithHeaderUrl.equals(url)) {
            return super.shouldInterceptRequest(view, url);
        }
        try {
            //使用 OkHttp 构造并发送请求
            OkHttpClient okHttpClient = HttpHelper.getInstance().getOkHttpClient();
            RequestBody b = new MultipartBody.Builder()
                    .addFormDataPart("jsonfield", requestData)
                    .build();
            //创建 POST 请求并增加 Header        
            Request request = new Request.Builder().url(url)
                    .post(b)
                    .addHeader("Content-Type", "multipart/form-data")
                    .addHeader("Authorization", token)
                    .build();
            Response response = okHttpClient.newCall(request).execute();
            //通过响应结果构造 WebResourceResponse 对象并返回
            return new WebResourceResponse("text/html", response.header("content-encoding", "utf-8"), response.body().byteStream());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.shouldInterceptRequest(view, url);
    }
}
```  

另外，要使自定义 WebViewClient 生效，需要在加载页面前调用 `WebView.setWebViewClient` 方法来把 `WebViewClient` 设置给 `WebView` ：
```java
webView.setWebViewClient(MyWebViewClient());
```

