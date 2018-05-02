---
layout: post
title: "神奇的 RSA 算法"
date: 2017-10-30 16:10:00 +0800
tags: [Code,CS,Algorithms]
subtitle: "一种实现非对称加密的算法"
---
计算机通信加密有两种方式——对称加密和非对称加密。   

对称加密要求通信双方使用同一个密钥，发送方用这个密钥加密消息，接收方用这个密钥进行解密，比如浏览器和服务器之间的对称加密通信可用下图表示：   
![](/assets/img/post/duichen.png)  

对称加密要解决的问题就是密钥的传输问题，也就是通信进行前以一种安全的方式传送密钥，不被第三方窃取。   

1976 年，Diffie-Hellman 开创了公开密钥密码系统（也就是非对称加密）：每个通信个体生成一对密钥，分为公钥和私钥，其中公钥所有人都知道，私钥只有通信个体知道，并且用公钥加密的信息只有私钥才能解密，反之亦然。   
使用时，发送方用接收方的公钥对消息进行加密，接收方用自己的私钥进行解密，从而完成安全通信，如下图所示：   
![](/assets/img/post/feiduichen.png)

> 当然，非对称加密也需要面临一些问题，比如公钥传输过程中的中间人攻击等，不过这不在本篇文章的讨论范畴。

有一种实现非对称加密的著名算法叫 [RSA 算法](https://zh.wikipedia.org/wiki/RSA%E5%8A%A0%E5%AF%86%E6%BC%94%E7%AE%97%E6%B3%95)（取创始人 RonRivest、Adi Shamir 和 Leonard Adleman 的首字母命名），下面先来看一下它是怎么工作的，然后再简单解释它的原理。
## RSA 密钥生成
为了生成公钥和密钥，需要以下步骤：   
1. 选择两个不相等的大质数 p 和 q（这两个数越大，RSA 越不容易破解，当然，执行加密和解密的时间就越长，一般选择 1024 bit 的数量级）。
2. 计算 n = pq 和 z = (p-1)(q-1)
3. 选择一个数 e（e < z），且使 e 与 z 互质（即 e 和 z 没有除 1 之外的公因数）
4. 找一个数 d，使 ed - 1 可以被 z 整除，即：ed mod n = 1
5. 公布二元组（n，e）为公钥，保留二元组（n，d）为私钥 

## RSA 加密和解密 
假设 A 和 B 需要通信，A 有 B 的公钥（n，e），B 有私钥（n，d），A 发给 B 的消息为 m（**m < n**）。   

- 加密：先做指数运算 m^e，然后计算 m^e 被 n 整除的余数，因此消息 m 对应的加密结果为： 

![math01](/assets/img/post/math_01.png)

- 解密：对 c 做指数运算 c^d，然后计算 c^d 被 n 整除的余数得到结果就是 m，即：  

![math02](/assets/img/post/math_02.png)

## RSA 算法原理
神奇的 RSA 算法背后有严密的数学支撑，这里就简单介绍一下相关知识。   

在加密过程中，将表示消息的整数 m 做 e 次的幂运算，然后做模 n 运算得到密文 c，解密时先把密文 c 做 d 次幂，然后做模 n 运算得到原消息 m，即：  

![math03](/assets/img/post/math_03.png)

将（1）式代入（2），得：       

![math04](/assets/img/post/math_04.png)

根据模运算的运算规则：  
![math05](/assets/img/post/math_05.png) 

得 :  

![math06](/assets/img/post/math_06.png)

这时要用到数论中的一个结论：如果 p 和 q 是素数且 n = pq，则:    
![math07](/assets/img/post/math_07.png)

因此有

![math08](/assets/img/post/math_08.png)

由于开始选择的 e 和 d 满足如下关系：  

![math09](/assets/img/post/math_09.png)

可得  

![math10](/assets/img/post/math_10.png)  

又因为  m < n， 所以 m mod n = m。

也就是  
![math11](/assets/img/post/math_11.png)

这就是 RSA 加密和解密的原理，更详细的论证可以参考阮一峰的这[两篇](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)[文章](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)。

## PS
1. 由于RSA算法中要求的指数运算非常耗时，这导致非对称加密的效率很低，而对称加密速度却很快。因此，在实际应用中，非对称加密用来传输对称加密使用的共享密钥（也叫会话密钥），密钥传输完成后，通信双方使用这个共享密钥通过对称加密来通信。  

2. RSA 算法之所以安全，是因为目前没有已知的算法可以快速对一个大数进行因数分解（即把公开值 n 分解成 p 和 q，如果已知 p 和 q，结合公开值 e，就可以计算出d，那么 RSA 就被破解了），也就是说，如果有人发现出了更快的因数分解算法，那么 RSA 将不再安全。

## Ref
- 《计算机网络——自顶向下方法》  Chapter 8.2.2








