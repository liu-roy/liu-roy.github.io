---
title: websocket多线程问题
date: 2017-06-28 11:21:24
categories: websocket
tags: [websocket]
---
## 开发框架
- springMVC
- tomcat8

## 问题描述
后端建立websocket 前端连接上来，后台会主动推送agent脚本执行信息，由于采用netty框架，保证并发性，执行的结果是多线程处理的，通过websocket返回前端居然报错了，很是费解。症状见下图。
![](http://d17znh8lvwja9e.cloudfront.net/websocket-multithread-problem/1.jpg)
![](http://d17znh8lvwja9e.cloudfront.net/websocket-multithread-problem/2.jpg)
<!--more-->
## 排查解决过程
从图中可以看出，远端处于【TEXT_PARTIAL_WRITING】状态，就这这个关键字google（不得不说就英文搜索而言，google强太多），
最后发现，tomcat的websocket没有相关的多线程处理，有人在apache上提bug，可以看一下开发者给的回复：
[tomcat8 websocker bug](https://bz.apache.org/bugzilla/show_bug.cgi?id=56026)

```shell
I have some sympathy with this view. Given that the Javadoc for RemoteEndpoint.Basic explicitly states that  
concurrent attempts to send messages are not allowed and the Javadoc for RemoteEndpoint.Async doesn't, you'd be  
forgiven for thinking that meant that RemoteEndpoint.Async permitted concurrent attempts to send messages.  
Unfortunately, the (not very well documented) intention of the WebSocket EG was not to allow concurrent attempts to send messages. [1]  

I did take a quick look at adding an option to relax (i.e. ignoring) this but the change required to do this with the current Tomcat code would be far from trivial.  

I'm marking this as INVALID since the intention of the EG was to not allow this but WONTFIX is almost as appropriate if this is viewed as an enhancement request.
As an aside, one reason not to implement this particular enhancement is that apps that depended on it would not be portable between containers.
```

### 提bug的人提出了一个很常见的需求： 

一个简单的聊天应用程序与3个用户聊天将导致并发调用Async.sendText（）当A和B都发送一个必须传递到C的消息，所以用例是很常见的。  
应用程序担心同步或排队并发调用Async.sendText（）会对执行并发写入的应用程序造成巨大的负担，我相信这不是JSR 356的意图（尽管规范并没有说明并发保证） 。

### tomcat 开发者大致认为
javaDoc，RemoteEndpoint.Basic明确指出不允许并发发送消息，但是RemoteEndpoint.Async没有，并且websocket eg 的意图是不允许并发，最终结论将此bug定位为无效bug

## 解决方法
看下面的评论也是很有意思，哈哈，大部分人认为容器应该为应用程序开发者提供便利，提供容器内部的缓冲，，而不是程序开发者自己管理websocket Session缓冲区。有一个方法是利用同步来降低这种问题的出现频率，与此同时，Tyrus 1.5是安全的，貌似只有tomcat存在这个问题。

## 随之而来的新的问题 
采用同步操作之后，由于agent会有很多中间结果，当第一条消息到来把锁抢占之后，后面的结果已经到来，当锁释放之后，其他线程抢占锁，尼玛，原先有时间顺序的结果变得混乱无序.  难道真的要自己实现一套缓冲队列吗。左思右想，最终决定还是有公平锁好了 

公平锁是个好东西，自己内部实现了一套等待队列，虽然效率有点低，但是对业务来说可以满足需求

```java
 private static ReentrantLock lock = new ReentrantLock(true);
```
![](http://d17znh8lvwja9e.cloudfront.net/websocket-multithread-problem/3.jpg)
