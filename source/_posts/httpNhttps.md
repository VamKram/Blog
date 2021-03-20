title: HTTP！！！
date: 2019/05/15
cover: https://technologybook.tech/assets/img/osi.png
categories:
- protocol
tags:
- http

---
# HTTP（S）协议的故事

> The **Hypertext Transfer Protocol** (**HTTP**) is an [application layer](https://en.wikipedia.org/wiki/Application_layer) protocol for distributed, collaborative, [hypermedia](https://en.wikipedia.org/wiki/Hypermedia) information systems 
>
> -- wikipedia



## 为什么需要 http ？

http设计的初衷是提供一种接受和发布html页面的方法，随着协议的发展逐渐演变成当代互联网的基础，即提供可靠的信息交换途径。

## 原理

### 网络分层

不同的分层相应的解决了一些网络中的问题。例如数据丢包，重复，完整性问题，信号转换，信号衰减等，限制了层与层之间的接口就实现的网络升级低耦合的目标。

- ***OSI*** 七层网络模型（物理层，数据链入层，网络层，传输层，会话层，表现层，应用层）。

  ![image-20210319210204161](https://technologybook.tech/assets/img/osi.png)



一个请求在七层模型中运转图示

![image-20210319212258553](https://technologybook.tech/assets/img/osi1.png)

### TCP/IP

TCP的一些特点

- 基于链接（数据之间传输需要建立连接）
- 全双工： 双向
- 字节流
- 可靠
- 拥塞控制

可靠： TCP 三次握手四次挥手

​								![osi3](https://technologybook.tech/assets/img/osi3.png)

通过报标示

ACK：响应报文

SYN： 同步序列号建立连接

FIN： 结束连接的报文

URG： 优先处理

![image-20210319213733857](https://technologybook.tech/assets/img/osi2.png)

## https

> 对抗中间人攻击的唯一的办法

https = http + ssl/tsl

摘要算法: 能够把任意长度的数据压缩成固定长度的摘要。md5/sha1/sha2/sha256

对成加密算法: xor AES RC4

非对称加密算法: 有两个密钥，公钥和私钥，公钥加密私钥解密。 DH/DSA/RSA/ECC

 数字证书: 先生成公钥和私钥，提供公钥，域名等信息给CA机构，机构审查，通过后提供包含了签名（通过摘要算法获得），CA信息，有效时间，序列号和公钥的数字证书。

### 过程

1. TCP握手建立连接
2. 客户端的加密套件（客户端支持哪些加密算法）
3. 服务端查看支持的加密方式并传递支持的加密方式和公钥数字证书（CA/公钥用户信息/公钥/权威机构签名/有效期）
4. 客户端验证证书 （通过hash算法获得摘要， 通过公钥获得CA的摘要，比对）
5. 生成随机密钥（对成加密密钥），并将生成的密钥用公钥进行加密
6. 服务端私钥解密
7. 使用对成加密密钥加密数据