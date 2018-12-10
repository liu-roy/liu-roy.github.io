
---
title: RabbitMq核心概念和术语
date: 2018-12-3 15:47:44
categories: RabbitMq
tags: [RabbitMq]
---

## 简介
越来越多的消息中间件很容易让人产生混淆，在学习一种消息中间件的时候，最好先了解他的几种抽象概念，方便你理解，明白了这些概念，你学习起来的时候也就得心应手,同时也是使用好RabbitMQ的基础。

## 核心概念
- Producer
- Message
- Consumer
- AMQP
- Queue
- Message acknowledgment
- Message durability
- Prefetch count
- Exchange
- RoutingKey
- Binding
- Binding key
- Exchange Types
<!--more-->
## AMQP
 Producer Message Consumer 过于简单，我就不介绍了。AMQP，即Advanced Message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。
AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。
RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写。用于在分布式系统中存储转发消息
##  Message acknowledgment
message ackonwledgment 即消息回执，消费者在消费完消息后发送一个回执给rabbitmq，rabbtiMq收到回执后才将消息从队列中删除，如果没有收到回执并检测到消费者的RabbitMQ连接断开，那么会将该消息发送给其他消费者进行处理。这里不存在timeout概念，一个消费者处理消息时间再长也不会导致该消息被发送给其他消费者，除非它的RabbitMQ连接断开。
##  Message durability
消息持久化，即RabbitMQ服务重启的情况下，也不会丢失消息，我们可以将Queue与Message都设置为（durable），这样可以保证绝大部分情况下我们的RabbitMQ消息不会丢失。小概率丢失事件无法避免（比如RabbitMQ服务器已经接收到生产者的消息，但还没来得及持久化该消息时RabbitMQ服务器就断电了），如果我们需要对这种情况也要处理，那么我们要用到事务。
## Prefetch count
消费者在开启acknowledge的情况下，对接收到的消息可以根据业务的需要异步对消息进行确认，prefetch允许为每个consumer指定最大的unacked messages数目。例如：设置prefetchCount=10，则Queue每次给每个消费者（假设一共2个AB）发送10条消息，消费者无需马上回应，消费者将10条消息缓存本地客户端；当一个消费者处理完1条消息后Queue会再给该消费者发送一条消息。如果两个消费者都没有回复任何一条ack，则queue就不会继续发送消息。
## Exchange
在RabbitMQ中生产者不会直接将消息发送队列。实际的情况是，生产者将消息发送到Exchange，由Exchange将消息路由到一个或多个Queue中（或者丢弃）。
![](http://images.royliu.me/rabbitmq-base-concept/1.jpg)

## RoutingKey
routingKey针对生产者而言，发送消息时一般需要指定routingkey，而这个routing key需要与Exchange Type及binding key联合使用才能最终生效。

## Binding
Binding将Exchange与Queue关联起来，这样RabbitMQ就知道如何正确地将消息路由到指定的Queue了。
![](http://images.royliu.me/rabbitmq-base-concept/2.jpg)
## BindingKey
在绑定Exchange与Queue的同时，一般会指定一个binding key；生产者将消息发送给Exchange时，一般会指定一个routing key；当binding key与routing key一致，或者符合模式匹配，消息就会被路由到对应的Queue中。在绑定多个Queue到同一个Exchange的时候，这些Binding允许使用相同的binding key。binding key 并不是在所有情况下都生效，它依赖于Exchange Type，比如fanout类型的Exchange就会无视binding key，而是将消息路由到所有绑定到该Exchange的Queue。
## Exchange Types
RabbitMQ常用的Exchange Type有fanout、direct、topic、headers这四种

## 参考文献
[http://www.rabbitmq.com/getstarted.html]
[https://www.jianshu.com/p/4d043d3045ca]
