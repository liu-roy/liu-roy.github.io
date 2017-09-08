---
title: quartz源码分析——执行引擎和线程模型
date: 2017-09-09 23:14:48
categories: quartz
tags: [quartz, 源码分析]
---
---
[TOC]

---

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
![这里写图片描述](http://img.blog.csdn.net/20170908113301143)
从上述配置文件可以看出quartz配置了一个线程池,实现名称为SimpleThreadPool, 这个线程池作用是什么呢，那么我们先看一下这个类
##  SimpleThreadPool——quartz里的工头儿
```java
public class SimpleThreadPool implements ThreadPool {
    //初始化线程数，对应配置文件的threadCount
    private int count = -1;
    //线程优先级，对应配置文件的threadPriority
    private int prio = Thread.NORM_PRIORITY;
    //线程池是否停止
    private boolean isShutdown = false;
    //切换挂起
    private boolean handoffPending = false;
    //继承加载器，对应配置文件的threadsInheritContextClassLoader\
    //OfInitializingThread
    private boolean inheritLoader = false;
    //继承分组
    private boolean inheritGroup = true;
    //是否守护线程
    private boolean makeThreadsDaemons = false;
    //线程组 由于安全性和功能羸弱，effective java已不推荐使用
    private ThreadGroup threadGroup;
    //锁
    private final Object nextRunnableLock = new Object();
    //工作线程 WorkerThread是一个内部类
    private List<WorkerThread> workers;
    //可用线程
    private LinkedList<WorkerThread> availWorkers = new LinkedList<WorkerThread>();
    //忙碌线程
    private LinkedList<WorkerThread> busyWorkers = new LinkedList<WorkerThread>();
    private String threadNamePrefix;
    private final Logger log = LoggerFactory.getLogger(getClass());
    //调度器实例名称，对应配置文件scheduler.instanceName
    private String schedulerInstanceName;
```
以上是这个类的成员变量，从上面的成员变量可以看出，这个线程池用LinkedList存储执行所有job的工人（Worker），来管理了所有的工人（Worker），那么我们就叫SimpleThreadPool为工头儿吧，老板要分派任务，肯定会找工头儿，工头在找空闲的工人来处理工作。
那工头对老板提供的接口是什么呢,继续往下看
```java
public boolean runInThread(Runnable runnable) {
        if (runnable == null) {
            return false;
        }
        synchronized (nextRunnableLock) {
            handoffPending = true;

            // Wait until a worker thread is available
            while ((availWorkers.size() < 1) && !isShutdown) {
                try {
                    nextRunnableLock.wait(500);
                } catch (InterruptedException ignore) {
                }
            }
            if (!isShutdown) {
                WorkerThread wt = (WorkerThread)availWorkers.removeFirst();
                busyWorkers.add(wt);
                wt.run(runnable);
            } else {
                // If the thread pool is going down, execute the Runnable
                // within a new additional worker thread (no thread from the pool).
                WorkerThread wt = new WorkerThread(this, threadGroup,
                        "WorkerThread-LastJob", prio, isMakeThreadsDaemons(), runnable);
                busyWorkers.add(wt);
                workers.add(wt);
                wt.start();
            }
            nextRunnableLock.notifyAll();
            handoffPending = false;
        }
        return true;
    }
```
上面的runInThread 就是工头对老板提供的对外接口，Runnable就是老板安排的工作，流程是这样的:

![这里写图片描述](http://img.blog.csdn.net/20170908163046572?)
## WorkerThread——quartz里的工人
介绍了工头，再来介绍一下工人，工头儿通过调用work.run方法，工人就开始工作了
开一下代码
```java
public void run(Runnable newRunnable) {
            synchronized(lock) {
                if(runnable != null) {
                    throw new IllegalStateException("Already running a Runnable!");
                }

                runnable = newRunnable;
                lock.notifyAll();
            }
        }

```
```java        
 @Override
        public void run() {
            boolean ran = false;
            while (run.get()) {
                try {
                    synchronized(lock) {
                        while (runnable == null && run.get()) {
                            lock.wait(500);
                        }

                        if (runnable != null) {
                            ran = true;
                            runnable.run();
                        }
                    }
                } catch (InterruptedException unblock) {
                    // do nothing (loop will terminate if shutdown() was called
                    try {
                        getLog().error("Worker thread was interrupt()'ed.", unblock);
                    } catch(Exception e) {
                        // ignore to help with a tomcat glitch
                    }
                } catch (Throwable exceptionInRunnable) {
                    try {
                        getLog().error("Error while executing the Runnable: ",
                            exceptionInRunnable);
                    } catch(Exception e) {
                        // ignore to help with a tomcat glitch
                    }
                } finally {
                    synchronized(lock) {
                        runnable = null;
                    }
                    // repair the thread in case the runnable mucked it up...
                    if(getPriority() != tp.getThreadPriority()) {
                        setPriority(tp.getThreadPriority());
                    }

                    if (runOnce) {
                           run.set(false);
                        clearFromBusyWorkersList(this);
                    } else if(ran) {
                        ran = false;
                        makeAvailable(this);
                    }
                }
            }

            //if (log.isDebugEnabled())
            try {
                getLog().debug("WorkerThread is shut down.");
            } catch(Exception e) {
                // ignore to help with a tomcat glitch
            }
        }
    }
```
工头把任务交给工人，工人线程此时阻塞，当runnable被赋值时，工作线程被唤醒。流程图如下：
![这里写图片描述](http://img.blog.csdn.net/20170908163011710)

