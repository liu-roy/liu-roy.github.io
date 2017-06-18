---
title: 记一次亲身踩过的hibernate的bug
date: 2017-03-04 17:01:47
categories: hibernate
tags: [hiberate, hashcode]
---

在写实体类时，经常会对域增加校验，例如@NotNull表示哪个字段不能为空，昨天晚上调试代码，就遇到了问题，
```java
@Entity
public class ApplicationCategory implements Serializable {
    private static final long serialVersionUID = -8018302345969463947L;

    @Id
    @GeneratedValue
    private Integer id;

    @NotNull(message = "应用分类名称不能为空")
    private String name;

    private String remark;

    @Override
    public int hashCode() {
        int result = id.hashCode();
        result = 31 * result + name.hashCode();
        result = 31 * result + (remark != null ? remark.hashCode() : 0);
        return result;
    }
   .....
   .....
   //省略其他方法
}
```
程序启动，保存applicationCategory时，抛异常，错误如下：
```java
2017-05-23 18:51:43.195 ERROR 16876 --- [http-nio-65009-exec-4] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is javax.validation.ValidationException: HV000041: Call to TraversableResolver.isReachable() threw an exception.] with root cause

java.lang.NullPointerException: null
	at com.xxx.hhhh.cccc.machine.domain.ApplicationCategory.hashCode(ApplicationCategory.java:69)
	at org.hibernate.validator.internal.engine.resolver.CachingTraversableResolverForSingleValidation$TraversableHolder.buildHashCode(CachingTraversableResolverForSingleValidation.java:143)
	at org.hibernate.validator.internal.engine.resolver.CachingTraversableResolverForSingleValidation$TraversableHolder.<init>(CachingTraversableResolverForSingleValidation.java:104)
	at org.hibernate.validator.internal.engine.resolver.CachingTraversableResolverForSingleValidation$TraversableHolder.<init>(CachingTraversableResolverForSingleValidation.java:86)
	at org.hibernate.validator.internal.engine.resolver.CachingTraversableResolverForSingleValidation.isReachable(CachingTraversableResolverForSingleValidation.java:31)
	at org.hibernate.validator.internal.engine.ValidatorImpl.isReachable(ValidatorImpl.java:1612)
	at org.hibernate.validator.internal.engine.ValidatorImpl.isValidationRequired(ValidatorImpl.java:1597)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateMetaConstraint(ValidatorImpl.java:609)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateConstraint(ValidatorImpl.java:580)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateConstraintsForSingleDefaultGroupElement(ValidatorImpl.java:524)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateConstraintsForDefaultGroup(ValidatorImpl.java:492)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateConstraintsForCurrentGroup(ValidatorImpl.java:457)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validateInContext(ValidatorImpl.java:407)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validate(ValidatorImpl.java:205)
	at org.hibernate.cfg.beanvalidation.BeanValidationEventListener.validate(BeanValidationEventListener.java:114)
	at org.hibernate.cfg.beanvalidation.BeanValidationEventListener.onPreInsert(BeanValidationEventListener.java:78)
	at org.hibernate.action.internal.EntityIdentityInsertAction.preInsert(EntityIdentityInsertAction.java:197)
	at org.hibernate.action.internal.EntityIdentityInsertAction.execute(EntityIdentityInsertAction.java:75)
	at org.hibernate.engine.spi.ActionQueue.execute(ActionQueue.java:619)
	at org.hibernate.engine.spi.ActionQueue.addResolvedEntityInsertAction(ActionQueue.java:273)
	at org.hibernate.engine.spi.ActionQueue.addInsertAction(ActionQueue.java:254)
	at org.hibernate.engine.spi.ActionQueue.addAction(ActionQueue.java:299)
	at org.hibernate.event.internal.AbstractSaveEventListener.addInsertAction(AbstractSaveEventListener.java:317)
	at org.hibernate.event.internal.AbstractSaveEventListener.performSaveOrReplicate(AbstractSaveEventListener.java:272)
	at org.hibernate.event.internal.AbstractSaveEventListener.performSave(AbstractSaveEventListener.java:178)
	at org.hibernate.event.internal.AbstractSaveEventListener.saveWithGeneratedId(AbstractSaveEventListener.java:109)
	at org.hibernate.jpa.event.internal.core.JpaPersistEventListener.saveWithGeneratedId(JpaPersistEventListener.java:67)
	at org.hibernate.event.internal.DefaultPersistEventListener.entityIsTransient(DefaultPersistEventListener.java:189)
	at org.hibernate.event.internal.DefaultPersistEventListener.onPersist(DefaultPersistEventListener.java:132)
	at org.hibernate.event.internal.DefaultPersistEventListener.onPersist(DefaultPersistEventListener.java:58)
	at org.hibernate.internal.SessionImpl.firePersist(SessionImpl.java:775)
	at org.hibernate.internal.SessionImpl.persist(SessionImpl.java:748)
	at org.hibernate.internal.SessionImpl.persist(SessionImpl.java:753)
	at org.hibernate.jpa.spi.AbstractEntityManagerImpl.persist(AbstractEntityManagerImpl.java:1146)
	at sun.reflect.GeneratedMethodAccessor101.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.orm.jpa.ExtendedEntityManagerCreator$ExtendedEntityManagerInvocationHandler.invoke(ExtendedEntityManagerCreator.java:347)
	at com.sun.proxy.$Proxy113.persist(Unknown Source)
	at sun.reflect.GeneratedMethodAccessor101.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.orm.jpa.SharedEntityManagerCreator$SharedEntityManagerInvocationHandler.invoke(SharedEntityManagerCreator.java:298)
	at com.sun.proxy.$Proxy113.persist(Unknown Source)
	at org.springframework.data.jpa.repository.support.SimpleJpaRepository.save(SimpleJpaRepository.java:506)
	at sun.reflect.GeneratedMethodAccessor99.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.data.repository.core.support.RepositoryFactorySupport$QueryExecutorMethodInterceptor.executeMethodOn(RepositoryFactorySupport.java:503)
	at org.springframework.data.repository.core.support.RepositoryFactorySupport$QueryExecutorMethodInterceptor.doInvoke(RepositoryFactorySupport.java:488)
	at org.springframework.data.repository.core.support.RepositoryFactorySupport$QueryExecutorMethodInterceptor.invoke(RepositoryFactorySupport.java:460)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.data.projection.DefaultMethodInvokingMethodInterceptor.invoke(DefaultMethodInvokingMethodInterceptor.java:61)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:99)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:281)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.dao.support.PersistenceExceptionTranslationInterceptor.invoke(PersistenceExceptionTranslationInterceptor.java:136)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.data.jpa.repository.support.CrudMethodMetadataPostProcessor$CrudMethodMetadataPopulatingMethodInterceptor.invoke(CrudMethodMetadataPostProcessor.java:133)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:213)
	at com.sun.proxy.$Proxy126.save(Unknown Source)
	at com.hikvision.hummer.pandora.machine.service.impl.ApplicationCategoryServiceImpl.save(ApplicationCategoryServiceImpl.java:42)
	at com.hikvision.hummer.pandora.machine.service.impl.ApplicationCategoryServiceImpl$$FastClassBySpringCGLIB$$3748b453.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:720)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
	at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:99)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:281)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:655)
	at com.hikvision.hummer.pandora.machine.service.impl.ApplicationCategoryServiceImpl$$EnhancerBySpringCGLIB$$2a6d0df9.save(<generated>)
	at com.hikvision.hummer.pandora.controller.machine.MachineController.writeTestData1(MachineController.java:155)
	at com.hikvision.hummer.pandora.controller.machine.MachineController.doTest(MachineController.java:123)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:221)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:136)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:114)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:963)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:897)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:622)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:729)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:230)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:89)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:77)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:197)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:108)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:472)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:349)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:784)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:802)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1410)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
```
检查一下看到是validation报错，但是自己的变量的确已经赋值，并且提示hashCode代码有问题，但是我的代码都是IDE帮生成的，即使hashCode有问题，validation也不应该去调用此方法啊，@NotNull作用不是检查字段是否为空么，把@NotNull去掉或者换成@Column(nullable=false)就没问题。

在stackoverflow上找了答案[https://stackoverflow.com/questions/30333779/javax-validation-validationexception-hv000041-call-to-traversableresolver-isre](https://stackoverflow.com/questions/30333779/javax-validation-validationexception-hv000041-call-to-traversableresolver-isre)，发现正好有一位仁兄遇到相同的问题，有评论指出hashcode有问题，但是在save时，有的域对象确实为空，例如Integer Id，是数据库生成的，在save时肯定为null，但究竟是谁在save时调用了hashCode方法。

看到下面另外一条评论，也坚定了我的想法，这是hibernate的一个bug; 下面的又指出这是hibernate bug HV_1013，[https://hibernate.atlassian.net/browse/HV-1013](https://hibernate.atlassian.net/browse/HV-1013),在 5.3.0.CR1.修复.

看一下官方是怎么说的
```java
A couple of comments.
SingleThreadCachedTraversableResolver (or CachingTraversableResolverForSingleValidation as it is called in later
versions of Validator is there to speed up TraversableResolver calls by caching calls to this class during a single
validation call. How much this really improves the performance is a good question. This caching approach has been
around since Validator 4 and it might be worth checking the actual benefit of this implementation.

Secondly, I was about to suggest that you can always plugin your own implementation of a TraversableResolver in
case you want to bypass CachingTraversableResolverForSingleValidation, but I actually think we wrap any configured
TraversableResolver into this caching one. There I think it would be beneficial to provide a configuration 
property which allows to bypass the caching.

Last but not least, regardless of any potential changes in Validator, I suggest to change your hashCode 
implementation to take into account potential null values. What if you are going to use an instance of your class
in a Collection or Map or something similar. You will get exceptions in this case as well. Bean Validation occurs
at given life cycle events, for example in the case of JPA during pre-persist. This means that until you call 
persist followed by a explicit flush or transaction commit, your entity will be un-validated. What if your entity
is used in a one to many relation ship. Again you would get exceptions while still assembling your final object 
graph to persist. IMO hashCode must deal with potentially null values if it is possible that an instance can exist
in a state where the value is null (even if from a business point of view the field is mandatory).

```
三条观点，

为什么会调用hashCode 大体意思是好像加快什么，提升性能, 官方已承认是bug，在后面的版本进行了修改。

对TraversalbeResolver自己实现

最后一条，完善hashCode，因为自身带码本身就有潜在的空值

采用了最后一条，将hashCode重写 如下
```java
 @Override
    public int hashCode() {
        int result = id != null ? id.hashCode() : 0;
        result = 31 * result + (name != null ? name.hashCode() : 0);
        result = 31 * result + (remark != null ? remark.hashCode() : 0);
        return result;
    }
```