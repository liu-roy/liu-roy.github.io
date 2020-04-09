---
title: quartz源码分析——执行引擎和线程模型
date: 2017-09-09 23:14:48
categories: quartz
tags: [quartz, 源码分析]
---
---
[TOC]

---

>软件版本：quartz-2.2.3

## 序
上一篇介绍了quartz的[启动过程](http://royliu.me/2017/04/13/quartz2-x-code-analyse-startprocess/)，这篇主要介绍quartz的执线程模型，众所周知，quartz并没有采用定时器去完成定时任务，而是通过线程去完成。为了简化对quartz线程模型的理解，就暂用下理解方式吧

| 类名 | 名词解释 |
|-
| SimpleThreadPool| 工头儿 |
| WorkThread| 工人 |
| QuartzScheduleThread| 老板 |
| JobRunShell | 工作，实现了Runnable|
<!--more-->
## 从配置说起
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/1.jpg)

从上述配置文件可以看出quartz配置了一个线程池,实现名称为SimpleThreadPool, 这个线程池作用是什么呢，我把注释写在代码中。
##  SimpleThreadPool——quartz里的工头儿
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/2.jpg)

以上是这个类的成员变量，从上面的成员变量可以看出，这个线程池用LinkedList存储执行所有job的工人（Worker），来管理了所有的工人（Worker），那么我们就叫SimpleThreadPool为工头儿吧，老板要分派任务，肯定会找工头儿，工头在找空闲的工人来处理工作。
那工头对老板提供的接口是什么呢,继续往下看

![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/3.jpg)

上面的runInThread 就是工头对老板提供的对外接口，Runnable就是老板安排的工作，流程是这样的:

![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/4.jpg)

## WorkerThread——quartz里的工人
介绍了工头，再来介绍一下工人，工头儿通过调用work.run方法，工人就开始工作了
开一下代码
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/5.jpg)

![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/6.jpg)

工头把任务交给工人，工人线程此时阻塞，当runnable被赋值时，工作线程被唤醒。流程图如下：
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/7.jpg)


## QuartzSchedulerThread——Quartz里面的老板
QuartzSchedulerThread是quartz里真正负责时间调度的类，这个线程的run方法也是最外层的loop。主要负责任务触发，工作包装，任务批处理的控制，这个方法是本章最难的一个方法了，看一下主loop
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/8.jpg)
boss线程涉及的细节非常多，看一下流程图
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/9.jpg)

![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/10.jpg)


上面的流程介绍的差不多了，建议对着代码看流程，有助于理解。

## 线程模型图
一图以概之
![](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/quartz2-x-code-anylse-core-threadModel/11.jpg)


以上是自己的一家之言，若有错误之处，请不吝赐教，共同提高。

## 参考文档
* quartz官方文档 http://www.quartz-scheduler.org/documentation
