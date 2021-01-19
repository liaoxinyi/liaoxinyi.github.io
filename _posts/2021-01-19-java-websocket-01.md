---
layout:     post
title:      "推呀推，推呀推，推到客户端"
subtitle:   "一些概念、SpringBoot中集成WebSocket、多线程下的问题、获取客户端真实IP"
date:       2021-01-19
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka.png"
tags:
    - WebSocket
    - Java
---
> 资料来源于网络上各位前辈（Moshow郑锴等）的总结

### 前言
从最开始的简单的把消息推送给客户端，到开始关注心跳机制，到获取客户端的真实ip......  
还是有必要把之前整理的一些东西记录一下，话不多说，直接开搞。

### 一些概念
##### WebSocket是个啥？
电梯：https://blog.csdn.net/moshowgame/article/details/80275084
WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器全双工(full-duplex)通信——允许服务器主动发送信息给客户端

##### 特点
既然已经有了HTTP协议，为什么还需要另一个协议？因为HTTP协议有一个缺陷：**通信只能由客户端发起，HTTP 协议做不到服务器主动向客户端推送信息**

举例来说，我们想要查询当前的排队情况，只能是页面轮询向服务器发出请求，服务器返回查询结果。轮询的效率低，非常浪费资源（因为必须不停连接，或者 HTTP 连接始终打开）。因此WebSocket 就是这样发明的

### SpringBoot中集成WebSocket
##### maven依赖
SpringBoot2.0就有对WebSocket的支持  

```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-websocket</artifactId>  
</dependency> 
```

##### 配置类
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * 开启WebSocket支持
 */
@Configuration  
public class WebSocketConfig {  
	
    @Bean  
    public ServerEndpointExporter serverEndpointExporter() {  
        return new ServerEndpointExporter();  
    }  
  
} 
```

##### 主体类代码
```java
import com.alibaba.fastjson.JSONObject;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Component;

import javax.websocket.OnClose;
import javax.websocket.OnError;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Calendar;
import java.util.concurrent.ConcurrentHashMap;

@ServerEndpoint(value = "/v1/webSocket/{userId}",configurator = WebsocketServerEndpointConfig.class)
@Component
public class WebSocketServer {

    private static ConcurrentHashMap<String,WebSocketServer> webSockets = new ConcurrentHashMap<>();

    private String userId;

    private String userIdentification;

    private Session session;

    @OnOpen
    public void onOpen(@PathParam("userId") String userId,Session session) {
        try {
            //获取ip，采用WebsocketFilter中配置的WebsocketClientIP来获取
            String ip =(String) session.getUserProperties().get("WebsocketClientIP");
            //生成本次终端标识
            String userIdentification = new StringBuilder(userId).append("@").append(ip).toString();
            //查找是否已经有连接，有链接则关闭之前的链接
            if (webSockets.containsKey(userIdentification)) {
                WebSocketServer webSocket = webSockets.get(userIdentification);
                try {
                    webSocket.session.getBasicRemote().sendText("This Client Has logged in elsewhere!");
                } catch (IOException e) {
                    TODO 处理异常
                }
                webSocket.onClose();
            }
            //建立本次链接
            this.session = session;
            this.userId = userId;
            this.userIdentification = userIdentification;
            webSockets.put(userIdentification,this);
            //返回客户端终端标识
            JSONObject userJsonObject = new JSONObject();
            userJsonObject.put("identity", userIdentification);
            try {
                this.session.getBasicRemote().sendText(userJsonObject.toJSONString());
            } catch (Exception e) {
                TODO 处理异常
            }
            TODO 保存用户心跳信息
            LOG.info("NEW CONNECTION JOINED.userIdentification=[{}]. CURRENT CONNECTION NUMS IS={}. ",
                    userIdentification, webSockets.size());
        }catch (Exception e){
            TODO 处理异常
        }
    }

    @OnClose
    public void onClose() {
        webSockets.remove(userIdentification);
        TODO 终端下线处理
        LOG.info("CLIENT CONNECTION CLOSED.userIdentification=[{}]. CURRENT CONNECTION NUMS IS={}.",
                userIdentification,webSockets.size());
    }

    @OnMessage
    public void onMessage(String message) {
        try {
            if(StringUtils.isNotBlank(message)){
                HeartbeatInfoDTO heartbeatInfoDTO = JSONObject.parseObject(message, HeartbeatInfoDTO.class);
                UserService userService = (UserService) SpringUtil.getBean("userServiceImpl");
                userService.Heartbeat(heartbeatInfoDTO);
            }
        } catch (Exception e) {
            TODO 处理异常
        }
    }

    @OnError
    public void onError(Session session, Throwable error) {
        webSockets.remove(userIdentification);
        TODO 终端下线处理
        TODO 处理异常
    }

    /**
     * 发送消息
     * @param message
     * @param userIdentification
     */
    public void sendMsg(String message, String userIdentification) {
        try {
            WebSocketServer webSocket = webSockets.get(userIdentification);
            if (null==webSocket||null==webSocket.session) {
                TODO 此时为可预见的异常
                LOG.info(CgjErrorCode.ERR_JUDGE_WEBSOCKET_SEND_MSG.getCode(), "THERE IS NO THIS CLIENT.userIdentification={}", userIdentification);
                return;
            }
            webSocket.session.getBasicRemote().sendText(message);
            LOG.info("SEND MESSAGE SUCCESSFULLY.userIdentification={},msg={}", userIdentification, message);
        } catch (IOException e) {
            TODO 处理异常
        }
    }
}

```
##### 代码说明
- 可以把WebSocketServer理解为平时http请求中的Controller层  
- 直接`@ServerEndpoint("/imserver/{userId}")` +`@Component`启用即可，然后在里面实现`@OnOpen`开启连接，`@onClose`关闭连接，`@onMessage`接收消息等方法  
- 通过调用`WebSocketServer.sendInfo();`推送新信息  
- 单部署时：新建一个Map用于接收当前userId的WebSocket，方便IM之间对userId进行推送消息  
- 集群时：需要借助mysql或者redis等进行处理，改造对应的sendMessage方法  
- 从以前的CopyOnWriteArraySet改为了ConcurrentHashMap，保证多线程安全同时方便利用map.get(userId)进行推送到指定端口。相比之前的Set，Set遍历是费事且麻烦的事情，而Map的get是简单便捷的，当WebSocket数量大的时候，这个小小的消耗就会聚少成多，影响体验，所以需要优化。在IM的场景下，指定userId进行推送消息更加方便  

### 多线程下的问题
在多线程的情况下，除了采用ConcurrentHashMap之外，还需要注意**Websocker注入Bean问题**  
##### 情形描述
因为websocket是原型模式，`@ServerEndpoint`每次建立双向通信的时候都会创建一个实例，可以理解为WebSocketServer这个主体类其实是多例的了，所以自然直接注入单例的实体类就会报错  
##### 解决方法
这里的解决方法有两种：
- 根据上下文获取bean，[获取bean的工具](https://www.threejinqiqi.fun/2021/01/15/tool-java-code/)  
- 将需要注入的类作为WebSocketServer的静态成员变量即可  

```java
@Autowired
private static XXXService XXXService;
```

### 获取客户端真实IP
在Websocket中如果服务端想要直接从session中获取到客户端的ip是没法的，这个可能也算是Websocket通信的一个缺陷  
但是方法总归还是有的。大体的实现原理就是通过filter过滤ws链接建立的请求时从request中获取ip然后再放到ws的session中去，这样在WebSocketServer中就可以获取到客户端的ip了  
**当然，这里面还需注意一个问题就是，如果客户端是通过代理来建立的链接，那么在获取真实ip的时候需要注意这个问题**

##### WebsocketFilter
```java
import org.apache.commons.lang3.StringUtils;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;


@WebFilter(urlPatterns="/webSocket/*")
public class WebsocketFilter implements Filter {

    private static final String UNKNOWN_IP_FLAG="unknown";
    private static final String X_FORWARDED_FOR="x-forwarded-for";
    private static final String PROXY_CLIENT_IP="Proxy-Client-IP";
    private static final String WL_PROXY_CLIENT_IP="WL-Proxy-Client-IP";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req= (HttpServletRequest) request;
        //代理时
        String element = req.getHeader(X_FORWARDED_FOR);
        if(StringUtils.isBlank(element) || UNKNOWN_IP_FLAG.equalsIgnoreCase(element)) {
            element = req.getHeader(PROXY_CLIENT_IP);
        }
        if(StringUtils.isBlank(element)|| UNKNOWN_IP_FLAG.equalsIgnoreCase(element)) {
            element = req.getHeader(WL_PROXY_CLIENT_IP);
        }
        if(StringUtils.isBlank(element)|| UNKNOWN_IP_FLAG.equalsIgnoreCase(element)) {
            element = req.getRemoteAddr();
        }
        if(StringUtils.isBlank(element)) {
            //可以记录此时没有获取到ip
            //无法获取时返回默认值：UnknownIp
            element = "UnknownIp";
        }else {
            //剔除多个代理时
            if (element.contains(",")) {
                String[] strings = element.split(",");
                for (String ip : strings) {
                    if (!UNKNOWN_IP_FLAG.equals(ip)) {
                        element=ip;
                        break;
                    }
                }
            }
        }
        //在实时获取ip的时候也通过WebsocketClientIP来获取
        req.getSession().setAttribute("WebsocketClientIP",element);
        chain.doFilter(request,response);
    }
}
```
##### 启动类添加@ServletComponentScan注解

##### WebsocketServerEndpointConfig
```java
import javax.servlet.http.HttpSession;
import javax.websocket.HandshakeResponse;
import javax.websocket.server.HandshakeRequest;
import javax.websocket.server.ServerEndpointConfig;
import javax.websocket.server.ServerEndpointConfig.Configurator;
import java.util.Enumeration;
import java.util.Map;

public class WebsocketServerEndpointConfig extends Configurator {

    @Override
    public void modifyHandshake(ServerEndpointConfig sec, HandshakeRequest request, HandshakeResponse response) {
        HttpSession httpSession = (HttpSession) request.getHttpSession();
        Map<String, Object> attributes = sec.getUserProperties();
        if (null != httpSession) {
            Enumeration<String> names = httpSession.getAttributeNames();
            while (names.hasMoreElements()) {
                String name = names.nextElement();
                attributes.put(name, httpSession.getAttribute(name));
            }
        }
    }
}
```

##### WebSocketServer中
`@ServerEndpoint(value = "/webSocket/{userId}",configurator = WebsocketServerEndpointConfig.class)`  
`String ip =(String) session.getUserProperties().get("WebsocketClientIP");`  
**其中，WebsocketClientIP是自定义的属性key值，详细参考上面的使用示例**

