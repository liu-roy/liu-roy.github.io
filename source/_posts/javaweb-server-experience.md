---
title: 后端架构高可用可伸缩经验之谈
date: 2016-08-20 13:58:42
categories: 后端设计
tags: [高可用,伸缩性]
---

去年参加技术分享活动，七牛的一个技术简要的介绍了一些高可用可伸缩的一些经验之谈，听完之后受益匪浅，整理一下，主要分以下几个部分: 
- **入口层高可用**
- **业务层高可用**
- **缓存层高可用**
- **数据库高可用**
- **入口层可伸缩**
- **业务层可伸缩**
- **缓存层可伸缩**
- **数据库可伸缩**

![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/1.jpg)
下面来分层介绍实践方法。
<!--more-->
## 入口层高可用
nigix两个 keeplive保活 心跳做好。
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/2.jpg)

- 使用心跳技术：keeplive提供这个技术
- 比如机器A IP是1.2.3.4，机器B IP是1.2.3.5，那么再申请一个IP （1.2.3.6）我们称之为心跳IP，平时绑定再A上面，如果A宕机，那么IP会自动绑定到B上面
- DNS 层面绑定到心跳IP即可
- 两台机器必须在同一网段
- 服务监听必须监听所有IP，如果仅仅监听心跳IP，那么从机上的服务（不持有心跳IP的机器）会启动失败
- 服务器利用率下降（混合部署可以改善这一点）

考虑一个问题，两台机器，两个公网IP，DNS把域名同时定位到两个IP，这算高可用吗

不算，客户端（比如浏览器） 解析完后会随机选一个 IP访问 ， 而不是一个失败后就去另一个 。 所以如果一台机器当机 ，那么就有一半左右的用户无法访问 。 

## 业务层高可用
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/3.jpg)

 - 业务层不要有状态 ， 状态分散到缓存层和数据库层 。 只要没有状态，业务层的服务死掉后，前面的nginx会自动把流量打到剩下的服务 。 所以，业务层无状态是一个重点。
 - 友情提醒：不要因为想让服务无状态就直接用cookie session, 里边的坑有点大，考察清楚后再用比较好。比如重放攻击 。
 
## 缓存层高可用
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/4.jpg)
- 缓存层分得细一点，保证单台缓存宕机后数据库还能撑得住 。 
- 中小模下缓存层和业务层可以混合部署， 这样可以节省机器 
- 大型规模网站，业务层和缓存层分开部署。
- 缓存层高可用，缓存可以启用主从两台，主缓存活着的时候，主缓存读，主从缓存都写，主缓存宕机后，从变主，主恢复后， 变成新的从。这样可以保证数据完整性，实现高可用 

## 数据库高可用
- MySQL有主从模式， 还有主主模式都能满足你的需求
- MongoDB也有ReplicaSet的概念，基本都能满足大家的需求。 

##高可用小结
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/5.jpg)

##入口层可伸缩
- 入囗层如何提供伸缩性？直接铺机器 ？然后DNS加IP就可以了吧？ 
- 可以，但也有需要注意的地方
- 尽管一个域名解析到几十个IP没有问题，但是很浏览器客户端只会使用前面几个IP，部分域名供应商对此有优化（比如每次返回的IP顺序随机 ）， 但是这个优化效果不稳定。推荐的做法是使用少量的nginx机 器作为入囗，业务服务器隐藏在内网（HTTP类型的业务这种方式居多） 

##业务层可伸缩

- 跟应付高可用一样，保证无状态是很好的手段。加机器继续水平部署即可。 

##缓存层可伸缩
- 直接用 codis或者redis 3.0 即可 

- 如果低峰期间数据库能抗的住 ，那么直接下线存然后上新缓存就是最简单有效的办法 

缓存类型 

- 强一致性缓存： 无法接受从缓存中读取错误的数据（比如用户余额）
- 弱一致性缓存：可以接受一段时间内的错误数据，例如微博转发数
- 不变型缓存：缓存key对应的value不会变更
弱一致性和不变型缓存扩容很方便，用一致性HASH即可
###一致性HASH
在分布式集群中，对机器的添加删除，或者机器故障后自动脱离集群这些操作是分布式集群管理最基本的功能。如果采用常用的hash(object)%N算法，那么在有机器添加或者删除后，很多原有的数据就无法找到了，这样严重的违反了单调性原则
一致性HASH可以有效解决这个问题，一致性哈希算法在保持了单调性的同时，还是数据的迁移达到了最小，这样的算法对分布式集群来说是非常合适的，避免了大量数据迁移，减小了服务器的的压力。
举个例子
如果缓存从9台扩容到10台， 简单Hash况下90％的缓存会马上失效，而一致性Hash 情况下，只有10%的缓存会失效。

###强一致缓存问题
- 1.缓存客户端的配置更新时间会有微小的差异，在这个时间窗内有可能会拿到过期的数据。
- 2.如果在扩充节点之后裁撤节点，会拿到脏数据。比如a这个key之前在机器1， 扩容后在机器2，数据更新了， 但裁撤节点后key回到机器1 这样就会有脏数据的问题


问题二解决方法：要么保持永不减少节点，要么节点调整间隔大于数据有效时间。
问题一解决方法：
  - 两套hash配置都更新到客户端，但仍使用旧的配置
  - 两个个客户端改为只有两套hash结果一致的情况下会使用缓存，其余情况从数据库读，但写入缓存。
  -  逐个客户端通知使用新配置。




##数据库可伸缩
- 水平拆分
- 垂直拆分
- 定期滚动


![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/webserver-ha-experience/6.jpg)
