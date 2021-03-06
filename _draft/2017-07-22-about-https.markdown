---
layout: post
title:  Https 那些事
date:   2017-07-22 07:37:39 +0800
category: code
published: false
image: https/https.jpg
--- 
[HTTPS](https://en.wikipedia.org/wiki/HTTPS) 全称是 `HTTP over SSL`，又叫 `HTTP over TLS` 或 `HTTP Secure` .可以简单理解为是安全版的 HTTP 协议。HTTPS 之所以是安全的协议，就是因为有 TLS/SSL ,至于 TLS 和 SSL ,我们稍后在研究。我们先来看看 从 HTTP 到 HTTPS 的发展过程，先对 HTTPS 的工作原理有个初步了解。这里以浏览器和服务器之间的数据传输为例。   

## 从 HTTP 到 HTTPS    
- ### 明文传输 
起初浏览器和服务器之间通过 HTTP 协议传送数据都是**明文**的。
![明文传输](/assets/img/https/mingwen.png)   
这种方式虽然方便快捷，但是非常不安全，数据轻而易举就被第三方获取到了。     
- ### 对称加密      
为了让通信更安全，人们创造了对称加密。对称加密有公开的加密和解密算法，还有一个密钥钥，这个密钥只有浏览器和服务器知道。浏览器在发送数据前，先使用加密算法和密钥对数据进行加密生成密文，然后再通过网络发送给服务器。服务器在收到密文后，使用解密算法和密钥对密文进行解密，然后获取到原始数据信息。服务器给浏览器发送数据也是如此。  
![对称加密](/assets/img/https/duichen.png)  
但是对称加密有个问题，加密算法和解密算法是公开的，但密钥是收发双方私有的，但这个密钥怎么在网络上进行传输呢？如果密钥被中间人截获，那对称加密将变的毫无意义。这时候就要非对称加密登场了。   
- ### 非对称加密      
非对称加密有有一对密钥，一个叫公钥，是可以公开的，另一个叫私钥，是不能公开的。非对称加密有个特点：经公钥加密后的信息，只能通过私钥解密；而经过私钥加密的信息，只能通过公钥解密。根据这个特性，所以数据传输就过程就变成了这样：   
![非对称加密](/assets/img/https/feiduichen.png)   
因为公钥和加密解密算法都是公开的，所以只需要通过网络发送公钥，就可以进行安全通信了。但是很快人们就发现这个非对称加密有个严重的缺点——它太慢了，比对称加密慢太多，这就使传输效率大大降低。不过有人很快就找到了解决方法。   
- ### 非对称加密+对称加密      
这种方式工作原理就是：浏览器与服务器之间先通过非对称加密的方式传输密钥，之后就通过这个密钥进行对称加密通信。这样既解决了密钥传输问题，又解决了非对称加密速度慢的问题，可谓是一举两得。但是事情并没有这么简单。   
- ### 中间人攻击    
由于公钥是公开的，所以所有人都可以知道。假如有人在服务器与浏览器之间发送公钥时截获了信息，并且换成了自己的公钥。这样浏览器和服务器拿到的都是中间人的公钥，但却以为是对方的。这样通信过程就变成了这样：     
![中间人攻击](/assets/img/https/mid_attack.jpg)   
这个时候数据已经改变，但服务器却完全不知情。     
那该如何保证公钥能安全的被传输呢？     
- ### 数字签名与数字证书      
上面的问题说白了就是无法证明服务器的公钥确实是属于服务器的。 于是有人就想到了建立一个具有公信力的认证中心（`CA`），由它为每服务器颁发**数字证书**。  
具体操作过程就是：服务器将它的基本信息和它的公钥通过 Hash 算法生成消息摘要( Hash 算法能保证如果数据被篡改，那么消息摘要就会有很大变动，从而可以验证数据)， 消息摘要经过 CA 的私钥进行加密，生成**数字签名**。数字签名和 服务器的信息（包括公钥）合在一起，就构成了数字证书。
![生成数字证书](/assets/img/https/generate_certificate.jpg)  
当用户浏览器收到服务器的数字证书后，先用同样的 Hash 算法对服务器的信息处理，生成一个消息摘要，然后在用 CA 的公钥对数字签名进行解密，又会得到一个消息摘要，将两个消息摘要进行对比，如果相同，则证明服务器的身份没有问题，就可以取出公钥，进行下一步通信；否则，就说明服务器的身份验证有问题。
![验证服务器身份](/assets/img/https/identify_server.jpg)    
那么问题又来了，如何安全传递 CA 的公钥呢？这仿佛是个「鸡生蛋，蛋生鸡」的死循环。
解决方法就是，人们为 CA 设计了树状的信用层级，根 CA 可以为底层 CA 做信用担保。这些 CA 也有证书证明自己的身份。在浏览器或操作系统中，预先安装这些根 CA 的证书，这样就可以通过安全的方式拿到了 CA 的公钥，从而打破了死循环。   
- ### HTTPS     
 HTTPS 就是这样随着问题的产生和解决诞生的。  
  再来梳理一下 HTTPS 通信的大致流程。   
  1. 浏览器或者操作系统内部安装了根 CA 的证书，也就是有了根 CA 的公钥。
  2. 当浏览器请求向服务器发送请求时，服务器先发送自己的数字证书，颁发这个数字证书的 CA 可以是底层 CA ，因为有根 CA 为它担保，可以完全信任，这样浏览器拿到了 CA 的公钥。
  3. 浏览器将服务器的数字证书中的信息用 hash 算法处理，生成消息摘要A, 然后用 CA 的公钥对服务器数字证书中的数字签名进行解密，拿到消息摘要 B
  4. 若消息摘要 A 和消息摘要 B 完全相同，那么服务器的身份就可信任，进行步骤5；否则服务器身份验证有问题，停止请求。
  5. 浏览器获取服务器的公钥
  6. 浏览器生成一个随机密钥，使用服务器的公钥通过非对称加密的方式发送给服务器
  7. 服务器收到这个密钥之后，浏览器和服务器就可以通过对称加密进行通信了。
  ![https](/assets/img/https/https_full.jpg)
  由于密钥只有浏览器和服务器知道，所以即使有中间人获取到了传递的信息，也无法解密。   

## SSL/TLS  