---
layout:     post
title:      "Spring Cloud Stream-01-未完"
subtitle:   "基础"
update-date:       2021-06-10
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-redis-bk.png"
tags:
    - Java
    - SpringCloud
    - 消息队列

---
> 资料来源于：https://blog.csdn.net/zhuguang10/article/details/89183514

### 前言
之前在整合Kafka的时候，实践中大体上有三种方式：手动poll、Kafka注解以及Stream。但是一直都还没来得及去了解Stream的一些东西，除了“流”这一概念外，什么都没有，这里进行一下研究

`Spring Cloud Stream` 在 Spring Cloud 体系内用于构建**高度可扩展的基于事件驱动的**微服务，其目的是为了简化消息在 Spring Cloud 应用程序中的开发。Spring Cloud Stream (后面以 SCS 代替 Spring Cloud Stream) 本身内容很多，而且它还有很多外部的依赖，想要熟悉 SCS，必须要先了解 `Spring Messaging` 和 `Spring Integration` 这两个项目
### Spring Messaging
Spring Messaging 是 Spring Framework 中的一个模块，其作用就是统一消息的编程模型

##### Messaging
比如消息 Messaging 对应的模型就包括一个消息体 Payload 和消息头 Header：  
```java
package org.springframework.messaging; 
public interface Message {
    T getPayload();
    MessageHeaders getHeaders(); 
}
```
##### MessageChannel
消息通道 MessageChannel 用于接收消息，调用 send 方法可以将消息发送至该消息通道中

```java
@FunctionalInterface 
public interface MessageChannel {
    long INDEFINITE_TIMEOUT = -1;
    
    default boolean send(Message<?> message) {
        return send(message, INDEFINITE_TIMEOUT);
    }
    
    boolean send(Message<?> message, long timeout);
}
```

##### SubscribableChannel
消息通道里的消息如何被消费呢？

由消息通道的子接口可订阅的消息通道 SubscribableChannel 实现，被 MessageHandler 消息处理器所订阅

```java
public interface SubscribableChannel extends MessageChannel {
    boolean subscribe(MessageHandler handler);
    boolean unsubscribe(MessageHandler handler); 
}
//由MessageHandler 真正地消费/处理消息
@FunctionalInterface 
public interface MessageHandler {
    void handleMessage(Message<?> message) throws MessagingException; 
}
```

Spring Messaging 内部在消息模型的基础上衍生出了其它的一些功能，如：

- 消息接收参数及返回值处理  
    - 消息接收参数处理器 HandlerMethodArgumentResolver 配合 @Header, @Payload 等注解使用  
    - 消息接收后的返回值处理器 HandlerMethodReturnValueHandler 配合 @SendTo 注解使用  
- 消息体内容转换器 MessageConverter  
- 统一抽象的消息发送模板 AbstractMessageSendingTemplate


消息通道拦截器 ChannelInterceptor；
