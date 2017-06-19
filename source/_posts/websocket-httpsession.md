---
title: webscoket实战之利用httpsession定向推送
date: 2016-12-8 20:45:14
categories: websocket
tags: [websocket]
---

## 开发框架
springboot

## 场景
在利用websocket主动推送信息给客户端的过程中，经常会遇到一个普遍需求，就是推送的消息要定向推送给不同的用户，或者解释的再普通一点，不同的消息推送给不同的session。例如一个用户admin，可以在多台设备登录，此时就有多个session，当一个设备向后台发起一个请求，处理时间较长（未采用客户端进行ajax轮训或者long poll时），采用websocket协议时，要针对不同session的操作定向返回操作结果。
下面来具体看看如何实现上述需求。

## 实例
springboot对websocket支持很友好，只需要继承webSocketHandler类，重写几个方法就可以了
```java
public class ResultHandler extends TextWebSocketHandler {
    private static final Logger logger = LoggerFactory.getLogger(ResultHandler.class);

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
    }

    @Override
    public void afterConnectionEstablished(WebSocketSession webSocketSession) throws Exception { 
    }

    @Override
    public void afterConnectionClosed(WebSocketSession webSocketsession, CloseStatus status) throws Exception {
       
    }
}

```
然后写一个简单的配置类
```java
@EnableWebSocket
@Configuration
public class WebSocketConfig implements WebSocketConfigurer{

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        webSocketHandlerRegistry.addHandler(resultHandler (), "/getResult").withSockJS();
    }

    @Bean
    public ResultHandler resultHandler(){
        return new ResultHandler();
    }

}
```
<!--more-->
这样一个websocket 服务端就写好了

我们希望能够把websocketSession和httpsession对应起来，这样就能根据当前不同的session，定向对websocketSession进行数据返回
如何解决这个问题呢，考虑到websocket是在http的基础上建立起来的（具体websocket建立，自行百度），能不能在websocket握手的时候，把当前的sessionId放入websocketSession的属性中呢？

在查询资料之后，发现spring中有一个拦截器接口，HandshakeInterceptor，可以实现这个接口，来拦截握手过程，向其中添加属性
```java
public class WebSocketHandshakeInterceptor implements HandshakeInterceptor{

    private static final Logger logger = LoggerFactory.getLogger(WebSocketHandshakeInterceptor.class);
    @Override
    public boolean beforeHandshake(ServerHttpRequest serverHttpRequest,ServerHttpResponse serverHttpResponse,
		 WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
        if(serverHttpRequest instanceof ServletServerHttpRequest){
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) serverHttpRequest;
             HttpSession httpSession = servletRequest.getServletRequest().getSession(true);
            if(null != httpSession){
                map.put(Constants.SESSION_ID,httpSession.getId());
            }
        }
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest serverHttpRequest, 
		ServerHttpResponse serverHttpResponse, WebSocketHandler webSocketHandler, Exception e) {

    }
}
```
beforeHandShake方法中的Map参数 就是对应websocketSession里的属性，向里面添加内容后，可以在上面的resultHandler里面利用websocketSession参数将其取出来  String sessionId = (String)webSocketsession.getAttributes().get(Constants.SESSION_ID);

参考下面代码
```java
public class ResultHandler extends TextWebSocketHandler {
    private static final Logger logger = LoggerFactory.getLogger(ResultHandler.class);

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
		
    }

    @Override
    public void afterConnectionEstablished(WebSocketSession webSocketSession) throws Exception { 
	if(null != webSocketSession ){
            String sessionId  = (String)webSocketSession.getAttributes().get(Constants.SESSION_ID);
            addConnectionCount();
            logger.info("WebSocket connection established, sessionId={} ConnectCount={}", sessionId, getConnectionCount());
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession webSocketsession, CloseStatus status) throws Exception {
        if(null != webSocketsession ){
            String sessionId = (String)webSocketsession.getAttributes().get(Constants.SESSION_ID);
            SocketSessionManager.socketSessionMap.remove(sessionId);
            subConnectionCount();
            logger.info("WebSocket connection close, Status={}, connectionCount={}", status, getConnectionCount());
        }
    }
    public static synchronized  int getConnectionCount (){
        return ScriptTaskResultHandler.connectionCount;
    }

    private static synchronized void addConnectionCount(){
        ScriptTaskResultHandler.connectionCount++;
    }

    private static synchronized void subConnectionCount(){
        ScriptTaskResultHandler.connectionCount--;
    }
	
}
```
还可以自己写一个webSocketSession管理类，当连接建立好之后，保存在这个管理类中，用sessionId当做Key，webSocketSession当做value，当结果处理完成后，根据sessionId找到相应的webSocketSession，进行数据推送。

拦截器加入的方法
```java
@EnableWebSocket
@Configuration
public class WebSocketConfig implements WebSocketConfigurer{

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        webSocketHandlerRegistry.addHandler(resultHandler (), "/getResult").
        addInterceptors(new WebSocketHandshakeInterceptor()).withSockJS();
    }

    @Bean
    public ResultHandler resultHandler(){
        return new ResultHandler();
    }

}
```