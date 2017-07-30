---
title: quartz2.x源码分析——启动过程
date: 2017-04-13 14:59:01
categories: quartz
tags: [quartz, 源码分析]
---
先简单介绍一下quartz，Quartz是一个功能丰富的开源作业调度库，可以集成到几乎任何Java应用程序中 - 从最小的独立应用程序到最大的电子商务系统。quartz可用于创建执行数十，数百甚至数十万个作业的简单或复杂的计划; 任务定义为标准Java组件的任务，可以执行任何可以对其进行编程的任何内容。Quartz Scheduler包含许多企业级功能，例如支持JTA事务和集群。
以上内容来自[quartz官网](http://www.quartz-scheduler.org/overview/)。

我们会针对quartz的几个主要类进行分析，看一下quartz是如何实现定时调度功能
### 1.1测试Demo
 来自官方实例的example1 
<!--more-->
```java
public class SimpleExample {

  public void run() throws Exception {
    Logger log = LoggerFactory.getLogger(SimpleExample.class);

    log.info("------- Initializing ----------------------");

    // First we must get a reference to a scheduler
    SchedulerFactory sf = new StdSchedulerFactory();
    Scheduler sched = sf.getScheduler();

    log.info("------- Initialization Complete -----------");

    // computer a time that is on the next round minute
    Date runTime = evenMinuteDate(new Date());

    log.info("------- Scheduling Job  -------------------");

    // define the job and tie it to our HelloJob class
    JobDetail job = newJob(HelloJob.class).withIdentity("job1", "group1").build();

    // Trigger the job to run on the next round minute
    Trigger trigger = newTrigger().withIdentity("trigger1", "group1").startAt(runTime).build();

    // Tell quartz to schedule the job using our trigger
    sched.scheduleJob(job, trigger);
    log.info(job.getKey() + " will run at: " + runTime);

    // Start up the scheduler (nothing can actually run until the
    // scheduler has been started)
    sched.start();

    log.info("------- Started Scheduler -----------------");

    // wait long enough so that the scheduler as an opportunity to
    // run the job!
    log.info("------- Waiting 65 seconds... -------------");
    try {
      // wait 65 seconds to show job
      Thread.sleep(65L * 1000L);
      // executing...
    } catch (Exception e) {
      //
    }

    // shut down the scheduler
    log.info("------- Shutting Down ---------------------");
    sched.shutdown(true);
    log.info("------- Shutdown Complete -----------------");
  }

  public static void main(String[] args) throws Exception {

    SimpleExample example = new SimpleExample();
    example.run();

  }

}

public class HelloJob implements Job {

    private static Logger _log = LoggerFactory.getLogger(HelloJob.class);
    public HelloJob() {
    }
    public void execute(JobExecutionContext context)
        throws JobExecutionException {
        _log.info("Hello World! - " + new Date());
    }

}
```
### 1.2 quartz关键API
从上述example中我们可以看到quartz主要接口和类
 * Scheduler - 进行作业调度的主要接口.
 * Job - 作业接口，编写自己的作业需要实现，如example中的HelloJob
 * JobDetail - 作业的详细信息，除了包含作业本身，还包含一些额外的数据。
 * Trigger - 作业计划的组件-作业何时执行，执行次数，频率等。
 * JobBuilder - 建造者模式创建 JobDetail实例.
 * TriggerBuilder - 建造者模式创建 Trigger 实例.
 * QuartzSchedulerThread 继承Thread 主要的执行任务线程
 
从上面的几个接口，可以看到quartz设计非常精妙，将作业和触发器分开设计，同时调度器完成对作业的调度。
了解了几个关键类和接口作用，下面我们来分析整个执行过程。
### 1.3 执行过程分析
#### 1.3.1 从StdSchedulerFactory获取scheduler
```java

    public Scheduler getScheduler() throws SchedulerException {
        if (cfg == null) {
            initialize();
        }

        SchedulerRepository schedRep = SchedulerRepository.getInstance();

        Scheduler sched = schedRep.lookup(getSchedulerName());

        if (sched != null) {
            if (sched.isShutdown()) {
                schedRep.remove(getSchedulerName());
            } else {
                return sched;
            }
        }

        sched = instantiate();

        return sched;
    }
```

1. cfg变量为PropertiesParser实例————是quartz的配置信息（主要是quartz.properties），如果为空，就初始化读取quartz的配置信息。
2. SchedulerRepository是一个HashMap,用于存储Scheduler。如果有重名的，判断是否已经停止，是从hashMap删掉,否直接返回已保存实例
3. SchedulerRepository未找到，实例化一个scheduler

```java 
private Scheduler instantiate() throws SchedulerException {
        if (cfg == null) {
            initialize();
        }

        if (initException != null) {
            throw initException;
        }

        .....
        .....

        SchedulerRepository schedRep = SchedulerRepository.getInstance();

        // Get Scheduler Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String schedName = cfg.getStringProperty(PROP_SCHED_INSTANCE_NAME,
                "QuartzScheduler");

        String threadName = cfg.getStringProperty(PROP_SCHED_THREAD_NAME,
                schedName + "_QuartzSchedulerThread");

        .....
        .....

        String managementRESTServiceHostAndPort = cfg.getStringProperty(MANAGEMENT_REST_SERVICE_HOST_PORT, "0.0.0.0:9889");

        Properties schedCtxtProps = cfg.getPropertyGroup(PROP_SCHED_CONTEXT_PREFIX, true);

        // If Proxying to remote scheduler, short-circuit here...
        // ~~~~~~~~~~~~~~~~~~
        if (rmiProxy) {

           ....
           ....

            schedRep.bind(remoteScheduler);

            return remoteScheduler;
        }


        // Create class load helper
        ClassLoadHelper loadHelper = null;
        try {
            loadHelper = (ClassLoadHelper) loadClass(classLoadHelperClass)
                    .newInstance();
        } catch (Exception e) {
            throw new SchedulerConfigException(
                    "Unable to instantiate class load helper class: "
                            + e.getMessage(), e);
        }
        loadHelper.initialize();

        // If Proxying to remote JMX scheduler, short-circuit here...
        // ~~~~~~~~~~~~~~~~~~
        if (jmxProxy) {
            if (autoId) {
                schedInstId = DEFAULT_INSTANCE_ID;
            }

           ....
           ....

            jmxScheduler.initialize();

            schedRep.bind(jmxScheduler);

            return jmxScheduler;
        }

        
        JobFactory jobFactory = null;
        if(jobFactoryClass != null) {
           ....
           ....
        }

        InstanceIdGenerator instanceIdGenerator = null;
        if(instanceIdGeneratorClass != null) {
           .....
           .....
        }

        // Get ThreadPool Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String tpClass = cfg.getStringProperty(PROP_THREAD_POOL_CLASS, SimpleThreadPool.class.getName());
        ....
        ....

        // Get JobStore Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String jsClass = cfg.getStringProperty(PROP_JOB_STORE_CLASS,
                RAMJobStore.class.getName());

        if (jsClass == null) {
        }
        ....
        ....
        ....

        // Set up any DataSources
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String[] dsNames = cfg.getPropertyGroups(PROP_DATASOURCE_PREFIX);
        for (int i = 0; i < dsNames.length; i++) {
            PropertiesParser pp = new PropertiesParser(cfg.getPropertyGroup(
                    PROP_DATASOURCE_PREFIX + "." + dsNames[i], true));

            String cpClass = pp.getStringProperty(PROP_CONNECTION_PROVIDER_CLASS, null);

            // custom connectionProvider...
           ....
           ....
           ....

        }

        // Set up any SchedulerPlugins
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String[] pluginNames = cfg.getPropertyGroups(PROP_PLUGIN_PREFIX);
        SchedulerPlugin[] plugins = new SchedulerPlugin[pluginNames.length];
        for (int i = 0; i < pluginNames.length; i++) {
            Properties pp = cfg.getPropertyGroup(PROP_PLUGIN_PREFIX + "."
                    + pluginNames[i], true);

            ....
            ....

            plugins[i] = plugin;
        }

        // Set up any JobListeners
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        Class<?>[] strArg = new Class[] { String.class };
        String[] jobListenerNames = cfg.getPropertyGroups(PROP_JOB_LISTENER_PREFIX);
        JobListener[] jobListeners = new JobListener[jobListenerNames.length];
        for (int i = 0; i < jobListenerNames.length; i++) {
            Properties lp = cfg.getPropertyGroup(PROP_JOB_LISTENER_PREFIX + "."
                    + jobListenerNames[i], true);

            .....
            .....

            jobListeners[i] = listener;
        }

        // Set up any TriggerListeners
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String[] triggerListenerNames = cfg.getPropertyGroups(PROP_TRIGGER_LISTENER_PREFIX);
        TriggerListener[] triggerListeners = new TriggerListener[triggerListenerNames.length];
        for (int i = 0; i < triggerListenerNames.length; i++) {
            Properties lp = cfg.getPropertyGroup(PROP_TRIGGER_LISTENER_PREFIX + "."
                    + triggerListenerNames[i], true);

            String listenerClass = lp.getProperty(PROP_LISTENER_CLASS, null);

           .....
           .....
           .....

            triggerListeners[i] = listener;
        }

        boolean tpInited = false;
        boolean qsInited = false;


        // Get ThreadExecutor Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        String threadExecutorClass = cfg.getStringProperty(PROP_THREAD_EXECUTOR_CLASS);
        if (threadExecutorClass != null) {
            tProps = cfg.getPropertyGroup(PROP_THREAD_EXECUTOR, true);
           .....
           .....
        }



        // Fire everything up
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
        try {
                
    
            JobRunShellFactory jrsf = null; // Create correct run-shell factory...
    
            ......
           ......
    
            schedRep.bind(scheduler);
            return scheduler;
        }
        catch(SchedulerException e) {
            shutdownFromInstantiateException(tp, qs, tpInited, qsInited);
            throw e;
        }
        catch(RuntimeException re) {
            shutdownFromInstantiateException(tp, qs, tpInited, qsInited);
            throw re;
        }
        catch(Error re) {
            shutdownFromInstantiateException(tp, qs, tpInited, qsInited);
            throw re;
        }
    }
```
instantiate()是一个比较重要的方法，主要从上一段代码介绍的PropertiesParser获取[Scheduler配置信息](http://www.quartz-scheduler.org/documentation/quartz-2.2.x/configuration/)
1. 获取Scheduler Properties
2. 如果是rmi代理sheduler,则创建RemoteScheduler，并通过scheRep.bind放入SchedulerRepository中，返回scheduler,结束
3. 创建class load helper 加载类提供帮助
4. 如果是jmx scheduler 则进行对应操作，并通过scheRep.bind放入SchedulerRepository中，返回scheduler,结束

  往下是本地调度，不是远程调度，因此需要获取和本地调度相关的信息

6.获取线程池配置
7.获取Job存储设置，（分为内存存储和数据库存储作业还有一个不太了解的方式,默认是内存存储）
8.设置数据库连接池
9.安装Scheduler插件
10.安装JobListener ，监听作业启动前，作业呗否决执行，作业已经执行，参见[JobListener.class]
11.安装TriggerListener,监听触发器被触发工作将要执行时，触发错过，否决工作执行，工作完成，参见[TriggerListener.class]
12.获取ThreadExecutor配置
13.初始化JobRunShellFactory，这个工厂类很重要，后面会介绍
14.创建标准StdScheduler，返回。

#### 1.3.2 创建JobDetail
JobDeatil包含jobDataMap和jobClass，以及一些描述，名称等等，采用Build模式建造，jobDataMap主要存储一些额外信息。
#### 1.3.3 创建Trigger
Trigger主要包含了一些计划信息，详细可参考接口

准备工作做完了，开始进行调度
#### 1.3.3 scheduler.scheduleJob()
```java
public Date scheduleJob(JobDetail jobDetail, Trigger trigger)
        throws SchedulerException {
        return sched.scheduleJob(jobDetail, trigger);
    }
```
sched是StdScheduler的一个成员，是QuartzScheduler的实例，在上面的instantiate()方法中实例化过并且用它构造StdScheduler;看一下QuartzScheduler的scheduleJob
```java
    public Date scheduleJob(JobDetail jobDetail,
            Trigger trigger) throws SchedulerException {
        validateState();

        if (jobDetail == null) {
            throw new SchedulerException("JobDetail cannot be null");
        }
        
        if (trigger == null) {
            throw new SchedulerException("Trigger cannot be null");
        }
        
        if (jobDetail.getKey() == null) {
            throw new SchedulerException("Job's key cannot be null");
        }

        if (jobDetail.getJobClass() == null) {
            throw new SchedulerException("Job's class cannot be null");
        }
        
        OperableTrigger trig = (OperableTrigger)trigger;

        if (trigger.getJobKey() == null) {
            trig.setJobKey(jobDetail.getKey());
        } else if (!trigger.getJobKey().equals(jobDetail.getKey())) {
            throw new SchedulerException(
                "Trigger does not reference given job!");
        }

        trig.validate();

        Calendar cal = null;
        if (trigger.getCalendarName() != null) {
            cal = resources.getJobStore().retrieveCalendar(trigger.getCalendarName());
        }
        Date ft = trig.computeFirstFireTime(cal);

        if (ft == null) {
            throw new SchedulerException(
                    "Based on configured schedule, the given trigger '" + trigger.getKey() + "' will never fire.");
        }

        resources.getJobStore().storeJobAndTrigger(jobDetail, trig);
        notifySchedulerListenersJobAdded(jobDetail);
        notifySchedulerThread(trigger.getNextFireTime().getTime());
        notifySchedulerListenersSchduled(trigger);

        return ft;
    }
```
让我们来看一下schedule()方法
1. 一些校验，jobDetail，trigger，jobClass判空
2. 计算第一次执行的日期，计算日期为空，则抛出异常，job永远不会被调度
3. 如果上述通过，通过jobStore存储jobDetail和trigger
4. 通知监听器程序：工作加入通知、

```java
   public void notifySchedulerListenersJobAdded(JobDetail jobDetail) {
        // build a list of all scheduler listeners that are to be notified...
        List<SchedulerListener> schedListeners = buildSchedulerListenerList();

        // notify all scheduler listeners
        for(SchedulerListener sl: schedListeners) {
            try {
                sl.jobAdded(jobDetail);
            } catch (Exception e) {
                getLog().error(
                        "Error while notifying SchedulerListener of JobAdded.",
                        e);
            }
        }
    }
```
5.通知监听器程序：通知正在休眠（工作可能都执行完，主线程sigLock.wait()）的主执行线程，有工作加入，唤醒主线程

```java
 protected void notifySchedulerThread(long candidateNewNextFireTime) {
        if (isSignalOnSchedulingChange()) {
            signaler.signalSchedulingChange(candidateNewNextFireTime);
        }
    }

  public void signalSchedulingChange(long candidateNewNextFireTime) {
        synchronized(sigLock) {
            signaled = true;
            signaledNextFireTime = candidateNewNextFireTime;
            sigLock.notifyAll();
        }
    }
```

 6.通知调度器job被调度了

```java
    public void notifySchedulerListenersSchduled(Trigger trigger) {
        // build a list of all scheduler listeners that are to be notified...
        List<SchedulerListener> schedListeners = buildSchedulerListenerList();

        // notify all scheduler listeners
        for(SchedulerListener sl: schedListeners) {
            try {
                sl.jobScheduled(trigger);
            } catch (Exception e) {
                getLog().error(
                        "Error while notifying SchedulerListener of scheduled job."
                                + "  Triger=" + trigger.getKey(), e);
            }
        }
    }
```

此时scheduler.scheduleJob()执行完毕

#### 1.3.4 scheduler.start() 未执行此函数，没有什么会真正执行，主线程loop循环一直被wait。
```java
public void start() throws SchedulerException {
        sched.start();
    }

//sched.start()
public void start() throws SchedulerException {

        if (shuttingDown|| closed) {
            throw new SchedulerException(
                    "The Scheduler cannot be restarted after shutdown() has been called.");
        }

        // QTZ-212 : calling new schedulerStarting() method on the listeners
        // right after entering start()
        notifySchedulerListenersStarting();

        if (initialStart == null) {
            initialStart = new Date();
            this.resources.getJobStore().schedulerStarted();            
            startPlugins();
        } else {
            resources.getJobStore().schedulerResumed();
        }

        schedThread.togglePause(false);

        getLog().info(
                "Scheduler " + resources.getUniqueIdentifier() + " started.");
        
        notifySchedulerListenersStarted();
    }

```
1. 判断调度器是否关掉或停止（因为可能开始之前已经将调度器停止或关闭），是则直接抛异常
2. 通知监听器，马上就要执行
3. 启动一些通知jobStore和一些插件开始执行，如果initial不为空，说明曾经启动过，则重新恢复
4. 切换线程开始执行togglePause(false)。此时开始执行， 下面的run()方法选取了一部分，未设置开始之前，while里面还有一个while (paused && !halted.get()) 一直等待变量改变，否则，就一直wait这也就是sche.scheduleJob()未真正启动工作的原因

```java
 void togglePause(boolean pause) {
        synchronized (sigLock) {
            paused = pause;

            if (paused) {
                signalSchedulingChange(0);
            } else {
                sigLock.notifyAll();
            }
        }
    }

  public void run() {
        boolean lastAcquireFailed = false;

        while (!halted.get()) {
            try {
                // check if we're supposed to pause...
                synchronized (sigLock) {
                    while (paused && !halted.get()) {
                        try {
                            // wait until togglePause(false) is called...
                            sigLock.wait(1000L);
                        } catch (InterruptedException ignore) {
                        }
                    }

                    if (halted.get()) {
                        break;
                    }
                }
}
```

 5.通知任务已经开始执行

```java
   public void notifySchedulerListenersStarted() {
        // build a list of all scheduler listeners that are to be notified...
        List<SchedulerListener> schedListeners = buildSchedulerListenerList();

        // notify all scheduler listeners
        for(SchedulerListener sl: schedListeners) {
            try {
                sl.schedulerStarted();
            } catch (Exception e) {
                getLog().error(
                        "Error while notifying SchedulerListener of startup.",
                        e);
            }
        }
    }
```
至此，整个流程大约完成了。
具体执行的（run）细节看看有空再讲一下。


##参考文档
* quartz官方文档 http://www.quartz-scheduler.org/documentation