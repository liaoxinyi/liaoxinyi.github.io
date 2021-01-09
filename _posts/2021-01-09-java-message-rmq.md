---
layout:     post
title:      "这只兔子该怎么玩儿？"
subtitle:   "生产者的Confirm/Return、消费者的Auto/Manual"
date:       2021-01-09
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-rmq-bk.jpg"
tags:
    - Java
    - 消息队列
---
> 资料来源于网络上各位前辈（DatDreamer等）的总结

### 前言
终于能够有机会把这一个月的一些积累沉淀一下了，这个月在产品中主要是接触的都是消息中间件，rabbitMQ、activeMQ和kafka。前两个都在原来的基础之上，详细用了一些深入的特性，kafka暂时还是偏基础一点的使用。这里主要记录一下在使用rabbitMQ的时候一些想法  


### 生产者的Confirm/Return
- 配置启用生产者的发布确认和发布退回机制  
`spring.rabbitmq.publisher-confirms = true`  
`spring.rabbitmq.publisher-returns = true`  

- 分别实现RabbitTemplate中两个接口  

```java
/**
 * 确认消息到达Broker时返回ack=true，否则返回ack=true
 */
public interface ConfirmCallback {
    void confirm(CorrelationData correlationData, boolean ack, String cause);

}
/**
 * 需要把mandatory也设为true才会启用此回调，在未路由到任何队列时会调用
 */
public interface ReturnCallback {
    void returnedMessage(Message message, int replyCode, String replyText,
            String exchange, String routingKey);
}
```

- 将以上两个实现类注入RabbitTemplate实例中  

`rabbitTemplate.setReturnCallback(messageReturnCallback);`  
`rabbitTemplate.setConfirmCallback(messageConfirmCallback); `  

### 消费者的Auto/Manual
这里主要是指消费者在消费rmq消息发生异常时的auto/manual/none这三种ack的返回方式  
##### 说明
RabbitMQ对异常是一无所知的，它只根据收到的ack / nack / reject 以及 requeue 来处理该消息

异常是在消费端内部处理的

无论哪种模式，无论哪种异常，只要设置了重试，抛出异常，消费端就会进行重试

超出重试限制后，会根据当前的模式进行不同处理

##### auto自动确认
- 消息成功被消费，没有抛出异常，则自动确认，回复ack

不涉及`requeue`，毕竟已经成功了。`requeue`是对被拒绝的消息生效

- 当抛出`ImmediateAcknowledgeAmqpException`异常，则视为成功消费，确认该消息

- 当抛出`AmqpRejectAndDontRequeueException`异常的时候，则消息会被拒绝，且 requeue = false

AmqpRejectAndDontRequeueException异常除了可以代码主动抛出外，在一天rmq消费重试消费的次数超过限制后会自己抛出

- 其他的异常，则消息会被拒绝，且`requeue = true`

##### manual人工确认
- 无论消费消息的时候有没有异常，只看是否主动调用了`basicAck()`、`basicNack()`等方法，没有调用的则一直阻塞，该条消息就是uack的状态

- 即使是auto模式的那两个特殊的（`ImmediateAcknowledgeAmqpException`和`AmqpRejectAndDontRequeueException`）异常，在manual模式下都是一样的，没有调用`basicAck()`、`basicNack()`等方法时会一直阻塞

##### default-requeue-rejected属性
- 属性定义

**(若有重试)重试次数超过限制后**，是否将被拒绝的消息重新入队

注意：对象是被拒绝的消息。（这就是requeue的作用对象），所以不要期望能通过该属性解决manual模式的异常的问题，如果你根本没响应消息，就不符合"被拒绝"的概念

- 到底消息该不该重入队列？

由这几个配置项的优先级来决定：

1. 最高优先级

消息超出重试限制后抛出的`AmqpRejectAndDontRequeueException`异常，因为此时`requeue = false`（注意`ImmediateAcknowledgeAmqpException`异常其实是指消息消费成功了，所以不会重入队列的）

`basicNack()`的`boolean requeue`参数（注意该参数是调用该方法时必填的）

2. 次优先级

`default-requeue-rejected`配置的属性

3. 最低优先级

auto模式中，处理其他异常时，拒绝消息，且`requeue = true`

- 为什么会有这个配置？

设置为`false`时，可以在`auto`模式下，消息消费时发生异常（除开`AmqpRejectAndDontRequeueException`和`ImmediateAcknowledgeAmqpException`，因为二者默认消息不会回队）时的重入队至队首，防止死循环

##### 什么时候会产生消息死循环？
- 现象

一条消息一直不停被重复投递，其他后来的消息都在它后面堆积起来了

- 原因

无论哪种模式下，消息被拒绝且时，重回队列是指消息回到队列头部，而不会到队列尾部，比如

1. auto模式，没有设置重试，也没有设置`requeue = false`，因为默认`requeue = true` 

2. 消费过程中主动抛出异常（除开`AmqpRejectAndDontRequeueException`和`ImmediateAcknowledgeAmqpException`，因为二者默认消息不会回队）

3. maual模式，主动回执ack时，设置了`requeue = true`

- 解决

1. 可以选择不重回队列，转为重新发送该消息，此时会到队列尾部。但缺点是，该消息如果本身有问题，也会无限重复，只是不是死循环，一样是浪费资源的。需要通过限制处理次数，记录下来，人工解决

2. 配置回队属性，配合死信队列记录.(`auto`模式下通过`default-requeue-rejected`属性覆盖，`manual`模式下直接在basicNack中指定不回队列)










