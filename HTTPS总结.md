---
title: HTTPS总结
date: 2017-04-26 14:52:12
tags:
	- HTTP
categories:
	- extention
---

## HTTP存在的问题

[W3C](https://zh.wikipedia.org/wiki/%E4%B8%87%E7%BB%B4%E7%BD%91%E8%81%94%E7%9B%9F)和[IETF](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E5%B7%A5%E7%A8%8B%E4%BB%BB%E5%8A%A1%E7%BB%84)设计HTTP协议的时候没有考虑安全问题，导致使用HTTP协议通信时会出现以下三个问题：
>* **窃听**
>* **伪装**
>* **中间人攻击**

### 窃听
窃听相同IP段上的通信并非难事，只需要收集在互联上流动的数据包就行。抓包工具(Packet Capture)或嗅探器(Sniffer)可以解析收集的数据包。
常用的抓包工具Wireshark、Charles、Fiddler，还有Linux下的Aircrack-ng破解Wi-Fi之前也必须先进行抓包;





