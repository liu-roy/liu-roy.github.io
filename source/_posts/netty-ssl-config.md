---
title: Netty SSL安全配置
date: 2017-08-26 14:51:43
categories: 网络安全
tags: [netty,ssl]
---

# Netty SSL安全配置

---
[TOC]

---
## 摘要
在研发平台的过程中，涉及到平台网关和前置agent的通信加密，虽然目前软件在内网中，但是由于平台和agent的特殊性，一旦被控制，部署的软件就会受到很大威胁，平台网关采用Netty开发，下面主要介绍一下netty的ssl配置和安全软件扫出的Diffie-Hellman弱密码问题解决方法

## 主要名词解释
| 英文名称或缩写 | 名词解释 |
|-
| Netty | 高性能服务器端编程框架 |
| OpenSSL | 安全套接字层密码库 | 
| KeyTool | 密钥和证书管理工具 |
| Diffie-Hellman | 密钥交换算法 |
 
## SSL常用认证方式介绍
-  单向认证
-  双向认证
-  CA认证
### SSL单向认证
单向认证只需客户端验证服务端，即客户端只需要认证服务端的合法性，服务端不需要。这种认证方式适用Web应用，因为web应用的用户数目广泛，且无需在通讯层对用户身份进行验证，一般都在应用逻辑层来保证用户的合法登入。但如果是企业应用对接，情况就不一样，可能会要求对客户端(相对而言)做身份验证。这时就需要做SSL双向认证。
### SSL双向认证
双向认证顾名思义，服务端也需要认证客户端的合法性，这就意味着客户端的自签名证书需要导入服务端的数字证书仓库。
**采用这种方式会不太便利，一但客户端或者服务端修改了秘钥和证书，就需要重新进行证书交换，对于调试和维护工作量非常大，并且由于agent数目不确定，动态增加agent的时候需要平台和agent双发互相加入相应各自的证书。**
### CA认证
CA认证的好处是只要服务端和客户端只需要将CA证书导入各自的keystore，客户端和服务端只需判断这些证书是CA签名过的即可，这也是蜂鸟平台内部采用的认证方式
## 生成证书
openSSL的安装不是本经验案例的重点，这里不介绍
### 根证书生成
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout root.key –out root.crt –subj /C=CN/ST=ZheJiang/L=HangZhou/O=MyCompany/OU=GA/CN=GA –config openssl.cnf
```
### 服务器端证书生成
#### 服务器端秘钥对生成
```
keytool -genkey -alias server -keypass **** -validity 1825 -keyalg RSA -keystore gateway.keystore -keysize 2048 -storepass **** -dname "CN=GA, OU=GA, O=MyCompany, L=HangZhou, ST=ZheJiang, C=CN"
```
keypass： 指定生成秘钥的密码
keystore：指定存储文件的密码，再次打开需要此密码
#### 生成证书签名请求
```
keytool -certreq -alias server -keystore gateway.keystore -validity 1825 -file gateway.csr -storepass ****
```
#### 用根证书私钥进行签名
```
openssl x509 -req -in gateway.csr -CA root.crt -CAkey root.key -CAcreateserial -out gateway.pem -days 1825 -extensions SAN -extfile san1.cnf
```
### 导入根证书
```
keytool -keystore gateway.keystore -importcert -alias CA -file root.crt -storepass **** -noprompt
```

### 导入服务端证书
```
keytool -keystore gateway.keystore -importcert -alias server -file gateway.pem -storepass **** 
```

客户端证书生成方法与服务端基本相同，此处不再赘述，需要注意一点的是签名根证书必须是同一个

## Netty SSL配置 
### 获取SSLContext
```
public class SslContextFactory {

    private static final String	PROTOCOL = "TLS";

    private static volatile SSLContext SERVER_CONTEXT = null;

    private static final String DEFAULT_PROPERTIES = "application.properties";

    private static final String KEYSTORE_TYPE = "server.ssl.key-store-type";

    private static final String KEYSTORE_PASSWORD = "gateway.ssl.key-store-password";

    private static final String GATEWAY_KEYSTORE = "gateway.ssl.key-store";

    private SslContextFactory() {
    }

    private static void init(){
        Properties properties = null;
        InputStream gatewayKeyStore = null;
        InputStream gatewayTrustStore = null;
        try {
            properties = PropertiesTool.getInstance().getProperties(DEFAULT_PROPERTIES, false);
            //初始化keyManagerFactory
            KeyStore ks = KeyStore.getInstance(properties.getProperty(KEYSTORE_TYPE));
            gatewayKeyStore = new FileInputStream(properties.getProperty(GATEWAY_KEYSTORE));
            ks.load(gatewayKeyStore, properties.getProperty(KEYSTORE_PASSWORD).toCharArray());
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(ks, properties.getProperty(KEYSTORE_PASSWORD).toCharArray());
			 //初始化TrustManagerFacotry
            KeyStore ts = KeyStore.getInstance(properties.getProperty(KEYSTORE_TYPE));
            gatewayTrustStore = new FileInputStream(properties.getProperty(GATEWAY_KEYSTORE));
            ts.load(gatewayTrustStore, properties.getProperty(KEYSTORE_PASSWORD).toCharArray());
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(ts);
			//生成SSLContext
            SERVER_CONTEXT = SSLContext.getInstance(PROTOCOL);
            SERVER_CONTEXT.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);

        } catch (IOException e) {
            throw new GatewayException(e.getMessage(), e);
        } catch (Exception  e) {
            throw new GatewayException(e.getMessage(), e);
        } finally {
            if (null != gatewayKeyStore) {
                try {
                    gatewayKeyStore.close();
                } catch (IOException e) {
                }
            }
            if (null != gatewayTrustStore) {
                try {
                    gatewayTrustStore.close();
                } catch (IOException e) {
                }
            }
        }
    }

    public static SSLContext getServerContext() {
        if(SERVER_CONTEXT == null){
            synchronized (SslContextFactory.class) {
                if (SERVER_CONTEXT == null) {
                    init();
                }
            }
        }
        return SERVER_CONTEXT;
    }
}
```
### 加入NettyHandler
Netty 提供了一个SslHandler，主要用于加密和解密
在大多数情况下,SslHandler 将成为 ChannelPipeline 中的第一个 ChannelHandler 。这将确保所有其他 ChannelHandler 应用他们的逻辑到数据后加密后才发生,从而确保他们的变化是安全的。
![这里写图片描述](http://img.blog.csdn.net/20170826142915291?)
图片来自网络
```
		SSLContext sslCtx = SslContextFactory.getServerContext();
        SSLEngine sslEngine = sslCtx.createSSLEngine();
        //设置加密套件
        sslEngine.setEnabledCipherSuites(Constants.CIPHER_ARRAY);
        sslEngine.setUseClientMode(false);
        sslEngine.setNeedClientAuth(true);
        pipeline.addLast("SslEstablish",new SslHandler(sslEngine));
```
###  Diffie-Hellman 密码过弱问题
研究人员Alex Halderman和Nadia Heninger提出NSA已经能够通过攻击1024位素数的Diffie-Hellman密钥交换算法解密大量HTTPS、SSH和VPN连接。
提供安全的加密算法
```
 public static final String[] CIPHER_ARRAY = {"TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
 "TLS_DHE_RSA_WITH_AES_128_GCM_SHA256",
 "TLS_DHE_DSS_WITH_AES_128_GCM_SHA256"};
 ```
## 参考资料
Netty权威指南
https://weakdh.org/sysadmin.html

