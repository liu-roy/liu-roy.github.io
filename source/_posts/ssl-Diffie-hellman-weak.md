---
title: ssl Diffie-Hellman弱密码问题
date: 2017-08-16 16:41:55
categories: 网络安全
tags: ssl Diffie-Hellman
---
## 开发相关
- tomcat8
- jdk1.8
- springboot
- 扫描软件  Nessus
## 异常信息
  ![这里写图片描述](http://img.blog.csdn.net/20170816164030069)
## 关于Diffie-Hellman
Diffie-Hellman:一种确保共享KEY安全穿越不安全网络的方法，它是OAKLEY的一个组成部分。Whitefield与Martin Hellman在1976年提出了一个奇妙的密钥交换协议，称为Diffie-Hellman密钥交换协议/算法(Diffie-Hellman Key Exchange/Agreement Algorithm).这个机制的巧妙在于需要安全通信的双方可以用这个方法确定对称密钥。然后可以用这个密钥进行加密和解密。但是注意，这个密钥交换协议/算法只能用于密钥的交换，而不能进行消息的加密和解密。双方确定要用的密钥后，要使用其他对称密钥操作加密算法实际加密和解密消息。

## 原因及方案
Diffie-Hellman group长度过短，目前NSA已破解1024位Diffie-Hellman 
### 第一步：生成多位数Diffie-Hellman group
openssl dhparam -out dhparams.pem 2048
### 第二步：使用安全的密码套件 
这里举tomcat为例，因为我们用的就是tomcat，其他的服务器nginx，iis，apache等看第一篇参考资料
Apache Tomcat
server.xml (for JSSE) 
``` xml
<Connector
ciphers="TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_DSS_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_SHA256,TLS_ECDHE_RSA_WITH_AES_128_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_SHA,TLS_ECDHE_RSA_WITH_AES_256_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_SHA384,TLS_ECDHE_RSA_WITH_AES_256_SHA,TLS_ECDHE_ECDSA_WITH_AES_256_SHA,TLS_DHE_RSA_WITH_AES_128_SHA256,TLS_DHE_RSA_WITH_AES_128_SHA,TLS_DHE_DSS_WITH_AES_128_SHA256,TLS_DHE_RSA_WITH_AES_256_SHA256,TLS_DHE_DSS_WITH_AES_256_SHA,TLS_DHE_RSA_WITH_AES_256_SHA"
/>
```
因为我采用的是springboot内置tomcat方法，没有server.xml 去springboot官网 可以在application.properties配置
![这里写图片描述](http://img.blog.csdn.net/20170816185030266)

server.ssl.ciphers=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_DSS_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_SHA256,TLS_ECDHE_RSA_WITH_AES_128_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_SHA,TLS_ECDHE_RSA_WITH_AES_256_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_SHA384,TLS_ECDHE_RSA_WITH_AES_256_SHA,TLS_ECDHE_ECDSA_WITH_AES_256_SHA,TLS_DHE_RSA_WITH_AES_128_SHA256,TLS_DHE_RSA_WITH_AES_128_SHA,TLS_DHE_DSS_WITH_AES_128_SHA256,TLS_DHE_RSA_WITH_AES_256_SHA256,TLS_DHE_DSS_WITH_AES_256_SHA,TLS_DHE_RSA_WITH_AES_256_SHA

针对老版本jdk，为了能够使用256位AES密码，有必要安装JCE无限强度管理策略文件，具体地址在[这里](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
## 参考资料
https://weakdh.org/sysadmin.html  
https://baike.baidu.com/item/Diffie-Hellman/9827194?fr=aladdin

