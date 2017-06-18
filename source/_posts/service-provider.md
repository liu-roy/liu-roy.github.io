---
title: 服务提供者框架
date: 2016-05-10 21:00:09
categories: 设计模式
tags: [服务提供者]
---

## 概述
服务提供者框架有三个重要组件
>* 服务接口，这是提供者需要实现的
>* 提供者注册接口，这是系统用来注册实现，让客户端访问他们的
>* 服务访问接口，是客户端用来获取服务的市里的

这些构成了了服务提供者框架的基础。
下面举一个简单的例子

```java
/**
 * 邮件服务，抽象了传输邮件的方法
 */
public interface MailService{

    void transTheMail();
}
```

```java
/**
 * 邮件服务提供者接口，服务提供者需要实现此接口，并在系统中注册
 */
public interface MailServiceProvider{
    MailService getMailSerice();
}
```
```java
/**
 * 电子邮件服务提供者，实现MailServiceProvider，并提供自己的运输邮件方法
 */
public class EmailProvider implements MailServiceProvider{
    static{
        MailserviceManager.registerProvider("Email", new EmailProvider());
    }

    @Override
    public MailService getMailSerice() {
        return  new Email();
    }

    class Email implements MailService{
        @Override
        public void transTheMail(){
            System.out.println("电子邮件发送");
        }
    }

}
```
```java
/**
 * 快递邮件服务提供者，实现MailServiceProvider，并提供自己的运输邮件方法
 */
public class UpsExpressProvider implements MailServiceProvider{
    static{
        MailserviceManager.registerProvider("UpsExpress", new UpsExpressProvider());
    }

    @Override
    public MailService getMailSerice() {
        return  new UpsExpress();
    }
    class UpsExpress implements MailService{
        @Override
        public void transTheMail(){
            System.out.println("实体运输");
        }
    }

}
```
```java

/**
 * 框架管理者需要提供注册方法，让服务提供者能够在框架中注册
 */

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class MailserviceManager {
    private static final Map<String,MailServiceProvider> providers = new ConcurrentHashMap<>();
    public static void  registerProvider(String name, MailServiceProvider p){
        providers.put(name, p);
    }
    public static MailService getMailService(String name){
        MailServiceProvider p = providers.get(name);
        if(null == p)
            throw new IllegalArgumentException(
                    "No provider redistered with name : "+ name);
        return p.getMailSerice();
    }
}

```

```java
/**
 * 测试类
 */
public class Test {

    public static void main(String[] args) throws ClassNotFoundException {
		提供者注册
        Class.forName("com.xxx.xxx.EmailProvider");
        MailService mailService =  MailserviceManager.getMailService("Email");
        mailService.transTheMail();

    }

}
```

有没有眼熟，和jdbc提供服务有异曲同工之妙，jdbc Connection 就是他的服务接口，DriverManager.registerDriver是提供者注册API，DriverManager.get Connetction是服务访问API，Driver就是服务提供者接口。