---
layout: post
title:  "使 OkHttp 信任自签名证书"
date:   2017-09-02 16:37:39 +0800
tags: [Develop,Code]
comments: true
subtitle: "困惑了很久的问题，终于有了答案。"
---  

在使用 `OkHttp` 向使用自签名证书的网站（比如:[12306](https://kyfw.12306.cn/otn/login/init)）发送请求时会直接失败，并且抛出一个 `SSLHandShakeException` 异常. 这是由证书认证失败导致的。之前遇到这个问题在网上东找西找了半天，最后才发现原来官方给了用于添加证书的 [Sample](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CustomTrust.java)。我结合其他处理方案，把它整理记录下来，同时查阅文档对用到的 API 做一些解释。　　　

## 为 OkHttp 添加自证书
### 获得自签名证书  
自签名（根）证书可以到要访问的网站下载，这里我用的是 [12306 的根证书](http://www.12306.cn/mormhweb/ggxxfw/wbyyzj/201106/srca12306.zip)，解压后会有一个 `srca.cer` 文件，把它复制到 Android 项目的 `resource/raw/`中。   
![getCer](/assets/img/https/get_cer.jpg?raw=true)    
### 开始配置   
由于是 demo，我就直接在 Activity 中写了一个 `customTrust()` 方法，用于初始化　`OkHttpClient` 。  
- `customTrust()`
```java
 private void customTrust() {
    X509TrustManager trustManager;
    SSLSocketFactory sslSocketFactory;
    //读取自签名证书
    InputStream cerIn = getResources().openRawResource(R.raw.srca);
    try {
        //通过trustManagerForCertificates方法为证书生成 TrustManager
        trustManager = trustManagerForCertificates(cerIn);
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[]{trustManager}, null);
        sslSocketFactory = sslContext.getSocketFactory();
    } catch (GeneralSecurityException e) {
        throw new RuntimeException(e);
    }
    //设置 OkHttpClient
    mOkHttpClient = new OkHttpClient.Builder()
                    .sslSocketFactory(sslSocketFactory, trustManager)
                    .build();
}
```
这个方法中调用了 `trustManagerForCertificates` 和 `newEmpryKeyStore` 两个方法，他们的内容如下：    
- `trustManagerForCertificates(InputStream is)`    
```java
public static X509TrustManager trustManagerForCertificates(InputStream in) 
    throws GeneralSecurityException {
    //InputStream 可以包含多个证书

    //CertificateFactory 用于生成 Certificate，也就是数字证书
    CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
    //由输入流生成证书
    Collection<? extends Certificate> certificates = certificateFactory.generateCertificates(in);
    if (certificates.isEmpty()) {
        throw new IllegalArgumentException("expected non-empty set of trusted certificates");
    }

    // 将证书放入 keyStore
    char[] password = "password".toCharArray(); // "password"可以任意设置
    KeyStore keyStore = newEmptyKeyStore(password);
    int index = 0;
    for (Certificate certificate : certificates) {
        String certificateAlias = Integer.toString(index++);
        keyStore.setCertificateEntry(certificateAlias, certificate);
    }

    // 用　KeyStore 生成 X509 trust manager.
     KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(
                KeyManagerFactory.getDefaultAlgorithm());
    keyManagerFactory.init(keyStore, password);
    TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);
    TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
    if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
        throw new IllegalStateException("Unexpected default trust managers:" + Arrays.toString(trustManagers));
    }
    return (X509TrustManager) trustManagers[0];
}
```
- `newEmpryKeyStore()`    
```java
private static KeyStore newEmptyKeyStore(char[] password) throws GeneralSecurityException {
    try {
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        InputStream in = null;
        // 传入 'null' 会生成一个空的 Keytore
        //password 用于检查 KeyStore 完整性和 KeyStore 解锁
        keyStore.load(in, password);
        return keyStore;
    } catch (IOException e) {
        throw new AssertionError(e);
    }
}
```
在 `customTurst()` 执行完后，自签名证书就被信任了，接下来就可以使用 `mOkHttpClient` 进行网络请求。
> 注意：这样配置以后，OkHttp 就只会信任添加的证书，系统内嵌的证书都会失效。如果现在对 `https://www.baidu.com` 发起一个 `get` 请求，同样会请求失败并抛出`SSLHandShakeException`， 因为没有添加对应的证书。如果想是系统原来证书有效，可以采取 `Certificate Pinning` 的形式，可以参考 [官方Sample](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CertificatePinning.java)   

## API Guides

- [
javax.net.ssl.X509TrustManager](https://developer.android.google.cn/reference/javax/net/ssl/X509TrustManager.html)  `X509TrustManager`是一个`interface`，继承了 `TrustManager`，这个接口的实例管理着用于验证远程服务器的 **X509 证书**。`X509` 是由 `ITU-T`国际组织指定的数字证书标准。
- [javax.net.ssl.TrustManagerFactory](https://developer.android.google.cn/reference/javax/net/ssl/TrustManagerFactory.html)   
生成 `TrustManager` 实例的工厂类。每个 `TrustManager` 都对应一种信任材料，由 `KeyStore` 或者 [java.security.Provider](https://developer.android.google.cn/reference/java/security/Provider.html) 指定。这里用到了它的几个方法：
   - `getInstance(String algorithm)`：用来创建使用指定算法的 `TrustManagerFactory`
   - `getDefaultAlgorithm()` ：用来获得默认的算法名称
   - `init(KeyStore keyStore)`： 使用一个 `KeyStore`实例来对 `TrustManagerFactory`进行初始化
   - `getTrustMananers()` ：为每种信任材料生成一个 `TrustManager` 
- [	java.security.KeyStore](https://developer.android.google.cn/reference/java/security/KeyStore.html)   
`KeyStore` 用于存储密钥和证书的键值对，每个键值对通过一个 `alia` 字符串区分。  
    - getInstance( String type )：获取指定类型的 `KeyStore`
    - load(InputSteam is,char[] password)： 以从指定输入流中加载`KeyStore`，`password` 用于检查 `KeyStore`的完整性和进行解锁。`is`传入`null`会生成一个空的`KeyStore`实例。
    - `serCertificateEntry(String alias, Certificate cert)`：为证书指定别名，用于区分
- `SSLSocketFactory`
用于创建`SSLSocket`的工厂类。`SSLSocket`继承自`Socket`，代表通过`SSL/TLS`加密的`Socket`.
- [javax.net.ssl.SSLContext](https://developer.android.google.cn/reference/javax/net/ssl/SSLContext.html)  
代表一种安全套接字协议的实现，并作为创建 `SSLSocketFactory` 的工厂。
    - `init(KeyManager[] km, 
                TrustManager[] tm, 
                SecureRandom random)`: 初始化 `SSLContext`
    - `getSocketFactory()` 生成一个 `SocketFactory`
- `CertificateFactory`
是用于生成 `Certificate` 的工厂类，这里使用的就是 `X509` 证书工厂，生成的就是 X509 证书
- [javax.net.ssl.KeyManagerFactory](https://developer.android.google.cn/reference/javax/net/ssl/KeyManagerFactory.html)  
通过信任材料生成 `KeyManager`的工厂类，信任材料由`keyStore`或`Provider`提供
    - `init(KeyStore ks,char[] password)`: 使用 `KeyStore` 初始化 `KeyManagerFactory`,`password` 用于保护 `keyStore`
