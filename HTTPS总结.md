---
title: HTTPS总结
date: 2016-05-13 14:52:12
tags:
	- HTTPS
categories:
	- extention
---

## HTTP存在的问题

[W3C](https://zh.wikipedia.org/wiki/%E4%B8%87%E7%BB%B4%E7%BD%91%E8%81%94%E7%9B%9F)和[IETF](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E5%B7%A5%E7%A8%8B%E4%BB%BB%E5%8A%A1%E7%BB%84)设计HTTP协议的时候没有考虑安全问题，导致使用HTTP协议通信时会出现以下三个问题：

* **窃听**
* **伪装**
* **中间人攻击**

<!-- more -->

### 窃听
窃听相同IP段上的通信并非难事，只需要收集在互联上流动的数据包就行。抓包工具(Packet Capture)或嗅探器(Sniffer)可以解析收集的数据包。
常用的抓包工具Wireshark、Charles、Fiddler，还有Linux下的Aircrack-ng破解Wi-Fi之前也必须先进行抓包;

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E7%AA%83%E5%90%AC.png?raw=true)

### 伪装

HTTP协议中的请求和响应都不会对通信双方进行身份确认，所以客户端和服务器都可能是伪装过的，无法确定正在通信的对方是否具有访问权限。

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E4%BC%AA%E8%A3%85.png?raw=true)

### 中间人攻击

HTTP协议无法证明通信的完整性，获取到的数据可能是被精心篡改过的。

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB.png?raw=true)


## HTTPS = HTTP + SSL/TLS

### HTTPS的结构

HTTPS在HTTP的基础上增加了SSL/TLS，弥补 HTTP存在的3个安全缺陷。HTTP和HTTPS的结构如下:

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/HTTP%E7%BB%93%E6%9E%84.png?raw=true)

### SSL/TLS

1. SSL/TLS为互联网通信提供安全及数据完整性保障，SSL是TLS的前身;
2. 1994 ，NetScape设计了SSL协议，未发布;
3. 1995 ，NetScape发布了SSL2.0版本，有严重安全漏洞;
4. 1996 ，SSL3.0问世，得到大量应用;
5. 1999 ，IETF接手SSL，发布了SSL的升级版TLS;
6. 2014 ，Google发布SSL3.0中的设计缺陷，建议禁止此协议;
7. TLS1.0 ≈ SSL3.0、TLS1.1 ≈ SSL3.1、TLS1.2 ≈ SSL3.2;

## 解决HTTP的问题

1. 加密
2. 数字证书认证
3. 完整性校验（数字签名）

### 加密

对通信内容使用加密算法和秘钥进行加密，然后用加密后的内容进行通信，这样即使通信内容被窃听，没有秘钥(及时加密算法公开)，就无法解密。

#### 对称秘钥加密技术
* 加密、解密使用同一个密钥;
* 加密、解密速度快；
* 如何将同一个密钥交给通信双方是个问题;
* DES、3-DES、AES、RC5、RC6;

#### 对称(公开)密钥加密技术
* 加密、解密不使用同一个密钥，分公钥、私钥;
* 公钥公开，私钥自己保存;
* 公钥可以解密私钥加密的内容，私钥可以解密公钥加密的内容;
* 加密、解密速度慢;
* RSA；

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E5%8A%A0%E5%AF%86.png?raw=true)

**HTTPS使用混合加密机制**

>先用非对称密钥加密技术建立通信，间接交换对称密钥，然后用对称密钥加密。下面有详细介绍。


### 数字签名

利用数字签名就可以对通信内容进行完整性校验。

数字证书也用到了数字签名，所以先说数字签名。看图:

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E7%AD%BE%E5%90%8D.png?raw=true)

### 数字证书

先看数字证书的构成:

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6.png?raw=true)

擦，写了一堆还是描述不清楚，看图:

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6-%E8%A7%A3%E5%AF%86.png?raw=true)


## HTTPS通信流程

So, At last. HTTPS的完整通信流程

![](https://github.com/pjocer/blogSource/blob/master/HTTPS%E6%80%BB%E7%BB%93/HTTPS%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B.png?raw=true)


## 参考

* 《HTTP权威指南》 
* 《图解HTTP》
* [RSA part1](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
* [RSA part2](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)
* 维基百科
* [HTTP2](https://http2.github.io/)
* [HTTP/2 Frequently Asked Questions](https://http2.github.io/faq/#what-are-the-key-differences-to-http1x)
* [Charles从入门到精通](http://blog.devtang.com/2015/11/14/charles-introduction/)




