---
title:spring-session设计解密
date: 2020-04-8 15:47:44
categories: 
tags: [spring-session]
---

[TOC]

## 目的
学习一下spring-session中包含的一些设计，理解其设计思想，其次是了解内部源码，逻辑。

## 工程结构
<img src="http://roy-markdown.oss-cn-qingdao.aliyuncs.com/spring-session-design/1.png" width = "400" height = "400" div align=right />

## 来自spring-session的思考

首先思考一下spring-session要解决什么问题，其次达到什么样的设计要求，
我们首先来正向推导，然后在结合代码逆向推导，他达到了一些什么要求

<!--more-->

### 基本要求

- 原业务无感知（重要）
- 支持多种存储介质
- 支持多种servlet容器（重要）
- 性能
- 稳定性、可靠性

要想做到第1、3条，基本限定必须要基于标准servlet协议 

## 基础知识
HttpSession (javax.servlet.http) 接口

Session (Spring-Session 接口)  MapSession RedisSession JdbcSession

ServletRequest->HttpServletRequest (javax.servlet;)

ServletRequestWrapper(类)->HttpServletRequestWrapper(类)(javax.servlet;)

SessionRepository(Spring Session接口) 一个管理Session实例的仓库

SessionRepositoryFilter（将spring-session 里面的Session转换 Httpsession 的实现）

SessionRepositoryRequestWrapper、SessionRepositoryResponseWrapper (spring-session)


<img src="http://roy-markdown.oss-cn-qingdao.aliyuncs.com/spring-session-design/6.jpg" width = "1200" height = "400" div align=right />


2种设计模式
- 适配器模式
- 包装着模式


思考 为什么要用适配器模式？spring为什么要另起一个Session的接口

1、通过调整HttpSessionAdapter 就可以屏蔽两种接口之间的差异
2、不仅仅能支持和满足servlet规范，还能方便扩展其他规范


## 关键类

| spring-session-core |  spring-session-data-redis|
|  ----                |     ----                  |  
| SessionRepositoryFilter  |               |
| Session              |            RedisSession RedisOperationsSessionRepositoryn内部类       |
| SessionRepository    |        RedisOperationsSessionRepository            |
| SpringHttpSessionConfiguration    |        RedisHttpSessionConfiguration            |

各个主要配置类作用


@EnableRedisHttpSession位于spring-session-data-redis module 中
并@Import RedisHttpSessionConfiguration.class
RedisHttpSessionConfiguration继承spring-session-core中的SpringHttpSessionConfiguration
其中SpringHttpSessionConfiguration只关注filter,cookie解析，sessionId解析
RedisHttpSessionConfiguration 主要作用构建SessionRepository （创建redis 序列化， redis连接工厂，命名空间，缓存有效期）
redisMessageListenerContainer 缓存的一些监听器


## 整体架构
<img src="http://roy-markdown.oss-cn-qingdao.aliyuncs.com/spring-session-design/2.png" width = "800" height = "400" div align=right/>


## 获取session
SessionRepositoryRequestWrapper getSession
```java
	@Override
		public HttpSessionWrapper getSession(boolean create) {
	     	//先获取request上下文中的，为什么，因为一次请求可能在业务层已经多次获取了
	     	//先放在本地request的ConcurrentHashMap中，不必每次去redis取
			HttpSessionWrapper currentSession = getCurrentSession();
			if (currentSession != null) {
				return currentSession;
			}
			//假如请求已经返回，第二次来请求，就获取当前request的sessionId
	       //从sessionRepository 拿出session 
	       // 两种情况，一是拿到了，二是没拿到
	       // 拿到了 就把放入当前request的ConcurrentHashMap 这里面涉及多线程
	       // 没拿到，说明session过期，或者非法的sessionId
			S requestedSession = getRequestedSession();
			if (requestedSession != null) {
				if (getAttribute(INVALID_SESSION_ID_ATTR) == null) {
					requestedSession.setLastAccessedTime(Instant.now());
					this.requestedSessionIdValid = true;
					currentSession = new HttpSessionWrapper(requestedSession, getServletContext());
					currentSession.setNew(false);
					setCurrentSession(currentSession);
					return currentSession;
				}
			}
			else {
				// This is an invalid session id. No need to ask again if
				// request.getSession is invoked for the duration of this request
				if (SESSION_LOGGER.isDebugEnabled()) {
					SESSION_LOGGER.debug(
							"No session found by id: Caching result for getSession(false) for this HttpServletRequest.");
				}
				setAttribute(INVALID_SESSION_ID_ATTR, "true");
			}
			if (!create) {
				return null;
			}
			if (SESSION_LOGGER.isDebugEnabled()) {
				SESSION_LOGGER.debug(
						"A new session was created. To help you troubleshoot where the session was created we provided a StackTrace (this is not an error). You can prevent this from appearing by disabling DEBUG logging for "
								+ SESSION_LOGGER_NAME,
						new RuntimeException(
								"For debugging purposes only (not an error)"));
			}
			S session = SessionRepositoryFilter.this.sessionRepository.createSession();
			session.setLastAccessedTime(Instant.now());
			currentSession = new HttpSessionWrapper(session, getServletContext());
			setCurrentSession(currentSession);
			return currentSession;
		}

```
## session创建
RedisOperationSessionsRepository.java
```java
	public RedisSession createSession() {
		Duration maxInactiveInterval = Duration
				.ofSeconds((this.defaultMaxInactiveInterval != null)
						? this.defaultMaxInactiveInterval
						: MapSession.DEFAULT_MAX_INACTIVE_INTERVAL_SECONDS);
		RedisSession session = new RedisSession(maxInactiveInterval);
		//看配置是否立即提交到session
		session.flushImmediateIfNecessary();
		return session;
	}
```


## session提交

session提交一共干了如下几件事

```java

 * HMSET spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe creationTime 1404360000000 maxInactiveInterval 1800 lastAccessedTime 1404360000000 sessionAttr:attrName someAttrValue sessionAttr2:attrName someAttrValue2
 * EXPIRE spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe 2100
 * APPEND spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe ""
 * EXPIRE spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe 1800
 * SADD spring:session:expirations:1439245080000 expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
 * EXPIRE spring:session:expirations1439245080000 2100
 ```
 
 
 RedisOperationsSessionRepository.java
 ```java
        private void save() {
            //servlet3.1规范，防止会话固定攻击
			saveChangeSessionId();
			saveDelta();
		}

		/**
		 * Saves any attributes that have been changed and updates the expiration of this
		 * session.
		 */
		private void saveDelta() {
			if (this.delta.isEmpty()) {
				return;
			}
			String sessionId = getId();
			//持久化session属性
			getSessionBoundHashOperations(sessionId).putAll(this.delta);
			String principalSessionKey = getSessionAttrNameKey(
					FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME);
			String securityPrincipalSessionKey = getSessionAttrNameKey(
					SPRING_SECURITY_CONTEXT);
            //和具体的登录或者安全框架有关，此处只考虑了spring_security_context的
			if (this.delta.containsKey(principalSessionKey)
					|| this.delta.containsKey(securityPrincipalSessionKey)) {
				if (this.originalPrincipalName != null) {
					String originalPrincipalRedisKey = getPrincipalKey(
							this.originalPrincipalName);
					RedisOperationsSessionRepository.this.sessionRedisOperations
							.boundSetOps(originalPrincipalRedisKey).remove(sessionId);
				}
				String principal = PRINCIPAL_NAME_RESOLVER.resolvePrincipal(this);
				this.originalPrincipalName = principal;
				if (principal != null) {
					String principalRedisKey = getPrincipalKey(principal);
					RedisOperationsSessionRepository.this.sessionRedisOperations
							.boundSetOps(principalRedisKey).add(sessionId);
				}
			}

            //清空session属性
			this.delta = new HashMap<>(this.delta.size());

            //更新即将过期时间
			Long originalExpiration = (this.originalLastAccessTime != null)
					? this.originalLastAccessTime.plus(getMaxInactiveInterval())
							.toEpochMilli()
					: null;
			RedisOperationsSessionRepository.this.expirationPolicy
					.onExpirationUpdated(originalExpiration, this);
		}

		private void saveChangeSessionId() {
			String sessionId = getId();
			//判断是否变换了sessionId
			if (sessionId.equals(this.originalSessionId)) {
				return;
			}
			//并且不是新的session，需要更改原来sessionId的值
			if (!isNew()) {
				String originalSessionIdKey = getSessionKey(this.originalSessionId);
				String sessionIdKey = getSessionKey(sessionId);
				//更改主session的key值
				try {
					RedisOperationsSessionRepository.this.sessionRedisOperations
							.rename(originalSessionIdKey, sessionIdKey);
				}
				catch (NonTransientDataAccessException ex) {
					handleErrNoSuchKeyError(ex);
				}
				String originalExpiredKey = getExpiredKey(this.originalSessionId);
				String expiredKey = getExpiredKey(sessionId);
				try {
				//更改过期session键 sessionId的值
					RedisOperationsSessionRepository.this.sessionRedisOperations
							.rename(originalExpiredKey, expiredKey);
				}
				catch (NonTransientDataAccessException ex) {
					handleErrNoSuchKeyError(ex);
				}
			}
			this.originalSessionId = sessionId;
		}

 ```


为何要这样设计呢
假设一下 
- 解决过期Session不能被及时清除的问题 （定时任务每隔一个钟去访问redis，触发清除）
- 为了不遍历全空间数据，将一分钟过期的数据放到同一个set下面，每分钟的定时任务只去清除这个set下的数据
- 即使数据过期，也不要立即删除当前,还有过期的事件处理




## session过期
需要监听session过期事件，并且进行触发


RedisHttpSessionConfiguration.java

```java
    //定时任务扫描
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
		taskRegistrar.addCronTask(() -> sessionRepository().cleanupExpiredSessions(),
				this.cleanupCron);
	}

```

```java
public void cleanExpiredSessions() {
		long now = System.currentTimeMillis();
		long prevMin = roundDownMinute(now);

		if (logger.isDebugEnabled()) {
			logger.debug("Cleaning up sessions expiring at " + new Date(prevMin));
		}

        //获取过期key
        //spring:session:expirations:1439245080000
		String expirationKey = getExpirationKey(prevMin);

		//expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
		Set<Object> sessionsToExpire = this.redis.boundSetOps(expirationKey).members();
		this.redis.delete(expirationKey);
		//分别touch过期key
	
		for (Object session : sessionsToExpire) {
		//	spring:session:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
			String sessionKey = getSessionKey((String) session);
			//触发删除，过期事件
			touch(sessionKey);
		}
	}

```

RedisOperationsSessionRepository.java
```java
public void onMessage(Message message, byte[] pattern) {
		byte[] messageChannel = message.getChannel();
		byte[] messageBody = message.getBody();

		String channel = new String(messageChannel);

		if (channel.startsWith(this.sessionCreatedChannelPrefix)) {
			// TODO: is this thread safe?
			Map<Object, Object> loaded = (Map<Object, Object>) this.defaultSerializer
					.deserialize(message.getBody());
			handleCreated(loaded, channel);
			return;
		}

		String body = new String(messageBody);
		if (!body.startsWith(getExpiredKeyPrefix())) {
			return;
		}

		boolean isDeleted = channel.equals(this.sessionDeletedChannel);
		if (isDeleted || channel.equals(this.sessionExpiredChannel)) {
			int beginIndex = body.lastIndexOf(":") + 1;
			int endIndex = body.length();
			String sessionId = body.substring(beginIndex, endIndex);

            //还是能取到session的 因为过期时间晚了5分钟,而且删除的是
            //spring:session:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
			RedisSession session = getSession(sessionId, true);

			if (session == null) {
				logger.warn("Unable to publish SessionDestroyedEvent for session "
						+ sessionId);
				return;
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Publishing SessionDestroyedEvent for session " + sessionId);
			}

			cleanupPrincipalIndex(session);

			if (isDeleted) {
			    // 给一些session事件监听器处理
				handleDeleted(session);
			}
			else {
				handleExpired(session);
			}
		}
	}

	private void cleanupPrincipalIndex(RedisSession session) {
		String sessionId = session.getId();
		String principal = PRINCIPAL_NAME_RESOLVER.resolvePrincipal(session);
		if (principal != null) {
			this.sessionRedisOperations.boundSetOps(getPrincipalKey(principal))
					.remove(sessionId);
		}
	}

	private void handleCreated(Map<Object, Object> loaded, String channel) {
		String id = channel.substring(channel.lastIndexOf(":") + 1);
		Session session = loadSession(id, loaded);
		publishEvent(new SessionCreatedEvent(this, session));
	}



```









## 参考文献
[https://github.com/spring-projects/spring-session/issues/92](https://github.com/spring-projects/spring-session/issues/92)

[https://github.com/spring-projects/spring-session/issues/93](https://github.com/spring-projects/spring-session/issues/93)

[https://www.cnblogs.com/lxyit/p/9672097.html](https://www.cnblogs.com/lxyit/p/9672097.html)

