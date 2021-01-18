---
layout:     post
title:      "这只兔子该怎么玩儿？"
subtitle:   "生产者的Confirm/Return、消费者的Auto/Manual、队列的监控"
date:       2021-01-09
update-date:  2021-01-18
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

### 队列的监控
在实际的应用场景中，可能会存在需要监控某一个队列的消息情况，比如有多少消息处于`Ready`状态？有多少消息处于`Unacked`状态？通过查询资料和实际操作之后，目前记录有下面两种方法：
##### Channel的queueDeclarePassive方法
- 使用   
这个方法主要是通过Channel的API方法，声明一个已经存在指定名字的队列，如果队列不存在，或在另一个Connection中为排他队列，会引发IO异常  
```java
 //获取队列中的消息个数
DeclareOk declareOk = channel.queueDeclarePassive(queue);
int num = declareOk.getMessageCount();
```

- 特点  
这种方法适合rmq连接和队列都是程序自己主动维护的情形，比如动态创建队列的时候  
但是，这个方法获取的队列消息没有详细的分类，比如只有消息总数，具体里面有多少是`Ready`状态，有多少是`Unacked`状态，都无法知道。而且，考虑到一般监控队列的时候都会是定时调用的，所以Channel的创建、管理和销毁也是一个问题  

##### http-API方法
- 使用   
RabbitMQ提供了HTTP API手册，其中就有获取队列情况的API  
一般流程是在请求头部中加入权限验证信息，然后http调用即可  
常用的API接口有：  

1. 获取单个队列信息  
`http://host:15672/api/queues/vhost/name`   

**host为RabbitMQ部署地址，vhost为队列所在的虚拟主机名，name为队列名。若队列所在默认虚拟主机，即主机名为"/"，请求时需将"/"url转码后（"%2f"）请求，例如http://localhost:15672/api/queues/%2f/TestSendMsg2**  

```java

import com.alibaba.fastjson.JSONObject;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import sun.misc.BASE64Encoder;

import java.net.URI;

String auth = username + ":" + password;
BASE64Encoder enc = new BASE64Encoder();
String encoding = enc.encode(auth.getBytes());
HttpHeaders headers = new HttpHeaders();
headers.add("Authorization","Basic " + encoding);
HttpEntity<String> requestEntity = new HttpEntity<>(null, headers);
String url = "http://"+host+":"+ webPort+"/api/queues/%2f/"+xxxxx（想要查询的队列名称）;
//如果不用url包裹一下，%2f就无法被识别出来，所以还是建议一开始就使用自己独立的虚拟主机
URI uri = new URI(url);
ResponseEntity<JSONObject> responseEntity = restTemplate.exchange(uri, HttpMethod.GET, requestEntity,
        JSONObject.class);
if (HttpStatus.OK != responseEntity.getStatusCode()||null==responseEntity.getBody()) {
    return;
}
JSONObject result = responseEntity.getBody();
  ready的消息数 = (int) result.get("messages_ready");
  unacked的消息数 = (int) result.get("messages_unacknowledged");
 消息总数= (int) result.get("messages");

```

获取单个队列信息的返回示例

```java
{
    "consumer_details": [],//消费者信息
    "incoming": [],
    "deliveries": [],
    "messages_details": {
        "rate": 0.0
    },
    "messages": 3,//对应消息数量
    "messages_unacknowledged_details": {
        "rate": 0.0
    },
    "messages_unacknowledged": 0,//对应消息数量
    "messages_ready_details": {
        "rate": 0.0
    },
    "messages_ready": 3,//对应消息数量
    "reductions_details": {
        "rate": 0.0
    },
    "reductions": 7189,
    "message_stats": {
        "publish_details": {
            "rate": 0.0
        },
        "publish": 3
    },
    "node": "rabbit@ahy-PC",
    "arguments": {},
    "exclusive": false,
    "auto_delete": false,
    "durable": false,
    "vhost": "/",
    "name": "TestSendMsg2",
    "message_bytes_paged_out": 0,
    "messages_paged_out": 0,
    "backing_queue_status": {
        "avg_ack_egress_rate": 0.0,
        "avg_ack_ingress_rate": 0.0,
        "avg_egress_rate": 0.0,
        "avg_ingress_rate": 0.004635025593395823,
        "delta": [
            "delta",
            "undefined",
            0,
            0,
            "undefined"
        ],
        "len": 3,
        "mode": "default",
        "next_seq_id": 3,
        "q1": 0,
        "q2": 0,
        "q3": 0,
        "q4": 3,
        "target_ram_count": "infinity"
    },
    "head_message_timestamp": null,
    "message_bytes_persistent": 0,
    "message_bytes_ram": 381,
    "message_bytes_unacknowledged": 0,
    "message_bytes_ready": 381,
    "message_bytes": 381,
    "messages_persistent": 0,
    "messages_unacknowledged_ram": 0,
    "messages_ready_ram": 3,
    "messages_ram": 3,
    "garbage_collection": {
        "minor_gcs": 11,
        "fullsweep_after": 65535,
        "min_heap_size": 233,
        "min_bin_vheap_size": 46422,
        "max_heap_size": 0
    },
    "state": "running",
    "recoverable_slaves": null,
    "consumers": 0,
    "exclusive_consumer_tag": null,
    "effective_policy_definition": [],
    "operator_policy": null,
    "policy": null,
    "consumer_utilisation": null,
    "idle_since": "2019-10-12 2:43:46",
    "memory": 12720
}
```

2. 获取所有队列信息  
`http://host:15672/api/queues`  

- 特点  
这种方法返回的信息特别详细，但是其中会包含一些敏感信息，所以应该考虑怎么处理这个问题，因为是Http的调用，所以也不存在通道的频繁建立销毁问题  
