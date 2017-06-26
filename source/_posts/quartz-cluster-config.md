---
title: quartz集群配置
date: 2017-06-26 15:09:35
categories: quartz
tags: [quartz]
---

## Quartz 应用集群的好处

集群的Quartz 应用比起非集群环境，能提供两个关键的好处

>*高可用性

>*伸缩性

>*负载均衡

### 高可用性
高可用性的应用是指能以高百分率的时间服务于客户。在某些情况下，可能就是一周 7 天，一天 24 小时。对于其他的应用，可能只是“尽量多的时间”。
### 伸缩性 

伸缩性意味着为增进应用自身能力，可以动态增加新的资源(如硬件) 到应用环境中的能力。在一个可伸缩的应用中，为达到增进能力的做法不涉及到改变代码或是设计。
获得伸缩性可不是变魔术那样。一个程序必须从一开始就要有合理的设计；支撑额外的能力通常要耗费管理上的努力来增加新的硬件(如内存) 或是启动程序更多的实例。
<!--more-->
### 负载均衡 

当前，Quartz 使用的是随机算法来提供最低限度负载均衡的能力。集群中的每个 Scheduler 实例尝试 Scheduler 所允许尽量快的速度触发已布署的 Trigger。Scheduler 实例通过触发自己的 Trigger 来竞争(使用数据库锁) 得到执行 Job 的权利。当某个 Job 的 Trigger 被触发时，别的 Scheduler 实例就不再试图触发这一 Trigger 了，直到下一次触发时间的到来。这种机制可能比你从它的简单性所推断的还要工作的更好。这是因为 Scheduler "多数时候是忙的"，作为至少的一个都有希望找到下次要触发的 Job。因此，这样就可能获得近乎完全合理的负载均衡性。


## 配置 

#Default Properties file for use by StdSchedulerFactory  
#to create a Quartz Scheduler Instance, if a different  
#properties file is not explicitly specified.

#default config

org.quartz.scheduler.instanceName = DefaultQuartzScheduler  
org.quartz.scheduler.rmi.export = false
org.quartz.scheduler.rmi.proxy = false  
org.quartz.scheduler.wrapJobExecutionInUserTransaction = false  

org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool  
org.quartz.threadPool.threadCount = 10  
org.quartz.threadPool.threadPriority = 5  
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread = true  
org.quartz.jobStore.misfireThreshold = 60000

#JobStore configure  
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX  
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate  
org.quartz.jobStore.useProperties =true  
org.quartz.jobStore.tablePrefix =QRTZ_  
org.quartz.jobStore.dataSource =myDs
org.quartz.jobStore.isClustered = true

#DataSource configure  
org.quartz.dataSource.myDs.driver =org.h2.Driver  
org.quartz.dataSource.myDs.URL =jdbc:h2:file:./h2/test;MVCC=true  
org.quartz.dataSource.myDs.user =sa
org.quartz.dataSource.myDs.password =  
org.quartz.dataSource.myDs.maxConnections =10  
