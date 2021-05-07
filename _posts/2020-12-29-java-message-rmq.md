---
layout:     post
title:      "这只兔子该怎么玩儿？"
subtitle:   "Channel的API、死信/临时/自动过期/延时队列、保证消息不丢失方案"
date:       2020-12-29
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/rmq-2021-1-18.jpg"
tags:
    - Java
    - 消息队列
    - RabbitMQ
---
> 资料来源于网络上各位前辈的总结

### 前言
终于能够有机会把这一个月的一些积累沉淀一下了，这个月在产品中主要是接触的都是消息中间件，rabbitMQ、activeMQ和kafka。前两个都在原来的基础之上，详细用了一些深入的特性，kafka暂时还是偏基础一点的使用。这里主要记录一下在使用rabbitMQ的时候一些想法  


### Channel的API  
这里主要采用的是Channel的方式，话不多说直接上代码  
##### 队列参数  

|参数名|功能|  
|----|----|  
|x-dead-letter-exchange|死信交换器|  
|x-dead-letter-routing-key|死信消息的可选路由键|  
|x-expires|队列在指定毫秒数后被删除|  
|x-ha-policy|创建HA队列|  
|x-ha-nodes|HA队列的分布节点|  
|x-max-length|队列的最大消息数|  
|x-message-ttl|毫秒为单位的消息过期时间，队列级别|  
|x-max-priority|最大优先值为255的队列优先排序功能|  

- x-expires说明  
 x-expires 参数控制 queue 被自动删除前可以处于未使用状态的时间。未使用的意思是 queue 上没有任何 consumer ，queue 没有被重新声明，并且在过期时间段内未调用过 basic.get 命令。该方式可用于，例如，RPC-style 的回复 queue, 其中许多 queue 会被创建出来，但是却从未被使用  
 参数值以毫秒为单位，如果该参数设置为 1000 ，则表示该 queue 如果在 1s之内未被使用则会被删除  

##### basicPublish
生产者发布一条消息。如果交换器不存在，会产生通道协议异常，并且关闭当前通道  
`void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, AMQP.BasicProperties props, byte[] body) throws IOException`  
Parameters:  
exchange - 消息要发布到的交换器，如果设为空字符串则会发布到默认交换器  
routingKey - 路由键  
mandatory - 如果设置了mandatory标签则为true。true：交换器无法路由到任何符合条件的队列，则会通过basic.return返还给生产者。false：出现以上情形直接丢弃该消息  
immediate - 如果设置了immediate标签则为true。true: 如果交换器路由消息的队列上没有消费者，这条消息不会放入队列；如果交换器路由到的所有队列都没有消费者，则会通过basic.return返还给生产者。 注意 RabbitMQ 3.0 版本开始去掉了此参数的支持  
props - 消息的基本属性集。成员有：contentType,contentEncoding,headers(Map<String,Object>),deliveryMode(消息投递模式),priority(消息优先级),correlationId(关联请求和其调用RPC之后的回复),replyTo(回调队列),expiration,messageId,timestamp,type,userId,appId,clusterId  
body - 消息体  

##### exchangeDeclare
声明交换器  
`AMQP.Exchange.DeclareOk exchangeDeclare(String exchange, BuiltinExchangeType type, boolean durable, boolean autoDelete, boolean internal, Map<String,Object> arguments) throws IOException`  
Parameters:  
exchange - 交换器名字  
type - 交换器类型（direct, fanout, headers, topic）  
durable - 是否持久化 (存储在磁盘，服务器重启时依旧存在)  
autoDelete - 自动删除的前提是至少有一个交换器或队列与之绑定，自动删除操作是当交换器所有绑定都解除了，就会被自动删除  
internal - 是否属于内部交换器（对于内部使用的交换器，不能由客户端直接发布）  
arguments - 可以传一些其他参数  
**exchangeDeclareNoWait**  
与exchangeDeclare一样，但是没有返回AMQP.Exchange.DeclareOk，表示该方法无需等待服务器返回创建结果  
**exchangeDeclarePassive**  
声明一个已经存在指定名字的交换器，如果交换器不存在会引发404通道异常  
`AMQP.Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException`  

##### exchangeDelete
删除交换器  
`AMQP.Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException`  
Parameters:  
exchange - 交换器名字  
ifUnused - 是否删除无客户端使用的交换器  
**exchangeDeleteNoWait**  
与exchangeDelete一样，但是没有返回AMQP.Exchange.DeleteOk，表示该方法无需等待服务器返回删除结果  

##### exchangeBind
绑定一个交换器到另一个交换器上  
`AMQP.Exchange.BindOk exchangeBind(String destination, String source, String routingKey, Map<String,Object> arguments) throws IOException`  
Parameters:  
destination - 目的交换器  
source - 来源交换器  
routingKey - 交换器绑定的路由键  
arguments - 一些其他的绑定参数  
**exchangeBindNoWait**  
与exchangeBind一样，但是没有返回AMQP.Exchange.BindOk，表示该方法无需等待服务器返回绑定结果  

##### exchangeUnbind
解绑交换器与交换器的绑定  
`AMQP.Exchange.UnbindOk exchangeUnbind(String destination, String source, String routingKey, Map<String,Object> arguments) throws IOException`  
Parameters: 参数与exchangeBind一致  
**exchangeUnbindNoWait**  
与exchangeUnbind一样，但是没有返回AMQP.Exchange.UnbindOk，表示该方法无需等待服务器返回解绑结果  

##### queueDeclare
声明一个队列  
`AMQP.Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String,Object> arguments) throws IOException`  
Parameters:  
queue - 队列名  
durable - 是否持久化 (服务器重启时依旧存在)  
exclusive - 是否为排他队列 (仅限于在声明它的当前连接的所有通道中使用)；排他队列即使声明为持久化的，一旦连接关闭或者所有客户端断开连接，也会被自动删除  
autoDelete - 自动删除的前提是至少有一个客户端与之绑定，自动删除操作是当队列所有绑定都解除了，就会被自动删除  
arguments - 一些其他的参数。有 x-massage-ttl(队列中的消息过期时间)，x-max-priority(指定了就代表队列里的消息是有优先级的，指定的数值代表最大优先级数，若队列中消息的优先级设置的值大于最大优先级数，一律按最大优先级数处理；若指定了队列的最大优先级数，则此后发送消息时都需要设置消息当前的优先级) 等。  
queueDeclareNoWait  
与queueDeclare一样，但是没有返回AMQP.Queue.DeclareOk，表示该方法无需等待服务器返回队列创建结果  
queueDeclarePassive  
声明一个已经存在指定名字的队列，如果队列不存在，或在另一个Connection中为排他队列，会引发IO异常  
  
##### queueDelete  
删除一个队列  
`AMQP.Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException`  
Parameters:  
queue - 队列名  
ifUnused - 是否删除无客户端使用的队列  
ifEmpty - 是否删除空的队列  
queueDeleteNoWait  
与queueDelete一样，但是没有返回AMQP.Queue.DeleteOk，表示该方法无需等待服务器返回队列删除结果  
  
##### queueBind  
绑定一个队列到交换器  
`AMQP.Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String,Object> arguments) throws IOException`  
Parameters:  
queue - 队列名  
exchange - 交换器的名字  
routingKey - 当前绑定的路由键  
arguments - 传递一些其他参数  
queueBindNoWait  
与queueBind一样，但是没有返回AMQP.Queue.BindOk，表示该方法无需等待服务器返回队列绑定结果  
  
##### queueUnbind  
解绑交换器的一个队列  
`AMQP.Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey, Map<String,Object> arguments) throws IOException`  
  
##### queuePurge  
清空给定队列内容  
`AMQP.Queue.PurgeOk queuePurge(String queue) throws IOException`  
  
##### basicGet  
获取队列中的一条消息  
`GetResponse basicGet(String queue, boolean autoAck) throws IOException`  
Parameters:  
queue - 队列名  
autoAck - 是否需要自动应答  
  
##### basicAck  
应答一条或多条消息。 AMQP.Basic.GetOk 或 AMQP.Basic.Deliver 的方法里提供了传递标签的值（deliveryTag），并且该方法中包含已被应答的消息  
`void basicAck(long deliveryTag, boolean multiple) throws IOException`  
Parameters:  
deliveryTag - 来自接收消息的AMQP.Basic.GetOk 或 AMQP.Basic.Deliver 的标签（tag）  
multiple - true: 应答所有消息，匹配给定传递标签的消息，false: 只应答匹配给定传递标签的消息  
  
##### basicNack  
拒绝一条或多条消息  
`void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException`  
Parameters:  
deliveryTag - 来自接收消息的AMQP.Basic.GetOk 或 AMQP.Basic.Deliver 的标签（tag）  
multiple - true: 拒绝所有未被确认的消息，也包括匹配给定传递标签的消息，false: 只拒绝匹配给定传递标签的消息。  
requeue - true: 被拒绝的消息会重新进入队列，而不是被丢弃或进入死信队列。如果队列只有一个消费者，消息会被尽可能放入原来的位置；如果队列有多个消费者，消息会被尽可能放入队头的位置  
  
##### basicReject  
拒绝一条消息  
`void basicReject(long deliveryTag, boolean requeue) throws IOException`  
Parameters:  
deliveryTag - 来自接收消息的AMQP.Basic.GetOk 或 AMQP.Basic.Deliver 的标签（tag）  
requeue - true: 被拒绝的消息会重新进入队列，而不是被丢弃或进入死信队列  
  
##### basicConsume  
启动一个消费者  
`String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String,Object> arguments, DeliverCallback deliverCallback, CancelCallback cancelCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException`  
Parameters:  
queue - 队列名  
autoAck - 消息是否会自动应答  
consumerTag - 消费者客户端自动生成，用于建立上下文的标签，区分多个消费者  
noLocal - true: 同一个Connection中的生产者和消费者不会互相传递消息。注意RabbitMQ 5.0 版本已不支持这个参数。  
exclusive - 是否为排他客户端（消费队列中只允许有当前一个消费者）  
arguments - 消费者传递的一些其他参数  
deliverCallback - 消费消息的推模式下，消费者接收消息投递的回调函数  
cancelCallback - 消费者取消的回调函数（消费者主动调用basicCancel或queueDelete）  
shutdownSignalCallback - 通道或连接中断时的回调函数  
  
实际使用时：  

```java
//构造队列参数
String exchangeName=groupExchangePrefix+groupId;
String queueName=clientQueueNamePrefix+userIdentification;
//1.建立通道
Channel channel= buildChannel();
//2.建立终端队列
//参数释义：1. queue： 队列的名称 ；2. durable： 是否持久化 ；3. exclusive： 是否排外的 ；4. autoDelete： 是否自动删除 ；5. arguments
//队列属性arguments设置：消息过期时间、对应的死信队列信息
Map<String, Object> arguments=new HashMap<>();
arguments.put(RMQ_QUEUE_PROPERTY_MSG_TTL, clientQueueMsgTtl*1000);
arguments.put(RMQ_QUEUE_PROPERTY_DEAD_EXCHANGE,deadExchangeName);
arguments.put(RMQ_QUEUE_PROPERTY_DEAD_EXCHANGE_ROUTING_KEY,deadExchangeRoutingKey);
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare(queueName,Boolean.TRUE,Boolean.FALSE,Boolean.FALSE,arguments);
//3.将队列Binding到对应路由上：queue、路由名、bindingKey
channel.queueBind(declareOk.getQueue(), exchangeName, userIdentification, new HashMap<>());
//4.监听
DefaultConsumer clientConsumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                               byte[] body) throws IOException {
        //消息处理逻辑
        
        //消息确认为成功获取，false表示不重新入队
        channel.basicAck(envelope.getDeliveryTag(), false);
    }
};
//消费者监听如上声明的队列，false表示需手动确认ack
channel.basicConsume(declareOk.getQueue(), Boolean.FALSE, queueName+"的消费者", clientConsumer);
```

##### basicCancel  
通过消费者标签，取消一个消费者  
`void basicCancel(String consumerTag) throws IOException`  
  
##### basicRecover  
请求Broker重新发送未应答的消息  
`AMQP.Basic.RecoverOk basicRecover(boolean requeue) throws IOException`  
Parameters:  
requeue - true：消息可能传递给不同的消费者  
  
##### basicQos  
限制客户端数据流量  
`void basicQos(int prefetchSize, int prefetchCount, boolean global) throws IOException`  
Parameters:  
prefetchSize - 消息最大长度，0为无限制  
prefetchCount - 消息最大数量，0为无限制  
global - true: 设置的参数应用到整个channel的consumer中，false: 应用到单个consumer  

##### 连接示意
```java
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public void buildDeadExchangeAndQueue() {
        Channel channel=null;
        try {
            //1.建立通道
            channel= buildChannel();
            //2.声明死信交换机 (交换机名, 交换机类型, 是否持久化, 是否自动删除, 是否是内部交换机, 交换机属性);
            channel.exchangeDeclare(deadExchangeName, BuiltinExchangeType.TOPIC, Boolean.TRUE, Boolean.FALSE, Boolean.FALSE,
                    new HashMap<>());
            //3.建立死信队列
            //参数释义：1. queue： 队列的名称 ；2. durable： 是否持久化 ；3. exclusive： 是否排外的 ；4. autoDelete： 是否自动删除 ；5. arguments
            AMQP.Queue.DeclareOk declareOk = channel.queueDeclare(deadQueueName,Boolean.TRUE,Boolean.FALSE,Boolean.FALSE,new HashMap<>());
            //将队列Binding到死信交换机上 (队列名, 交换机名, Routing key, 绑定属性);
            channel.queueBind(declareOk.getQueue(), deadExchangeName, deadExchangeRoutingKey, new HashMap<>());
        } catch (Exception e) {
            //todo somthing
        }finally {
            try {
                //4.关闭通道
                if (null!=channel) {
                    channel.close();
                }
            } catch (Exception e) {
                //todo somthing
            }
        }
    }
    
    public Channel buildChannel() {
        try {
            return RabbitConnectionUtil.buildRmqChannel(host,port,username,BiciiiUtil.transDataDecrypt(password));
        } catch (Exception e) {
            //todo somthing
        }
    }
    
    public static Channel buildRmqChannel(String host,int port,String username,String password) throws Exception {
        //如果rmq连接断开了，则先尝试建立连接
        if (null== RabbitConnectionUtil.RMQ_CONNECTION) {
            RabbitConnectionUtil.buildRmqConnection(host,port,username,password);
        }
        return RabbitConnectionUtil.RMQ_CONNECTION.createChannel();
    }
    
    //默认和rmq只建立一次连接
    public static Connection RMQ_CONNECTION;
    
    public static void buildRmqConnection(String host, int port, String username, String password)throws Exception {
        RMQ_CONNECTION=getRmqConnection(host,port,username,password);
    }
    
    public static Connection getRmqConnection(String host,int port,String username,String password) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(host);
        factory.setPort(port);
        factory.setUsername(username);
        factory.setPassword(password);
        return factory.newConnection();
    }
    
```

### 死信队列
##### 定义  
死信队列：DLX，Dead-Letter-Exchange  
当消息在一个队列中变成死信（dead message，就是没有任何消费者消费）之后，他能被重新publish到另一个Exchange，这个Exchange就是DLX 

##### 消息变死信条件  
- 消息被拒绝（basic.reject/basic.nack）,并且requeue参数为false（不重回队列）  
- 消息过期（TTL过期）。当消息过期时间设为0时，消息路由到的队列如果没有消费者，会立即变成死信，可以解决immediate参数为false时消息丢失的情况  
- 队列达到最大长度  

##### 使用
- 设置Exchange和Queue，然后进行绑定  
- Exchange: dlx.exchange(自定义的名字)  
- queue: dlx.queue（自定义的名字）  
- routingkey: #（#表示任何routingkey出现死信都会被路由过来）  
- 最后在然后正常的声明交换机、队列、绑定时，在队列的配置参数上加上一个参数：`arguments.put("x-dead-letter-exchange","dlx.exchange(自定义的名字)");`  

详细使用示例可参考上一篇内容[电梯直达](https://www.threejinqiqi.fun/2020/11/11/java-message-rmq/)


### 临时队列  
##### 自动删除队列  
自动删除队列和普通队列在使用上没有什么区别，唯一的区别是，当消费者断开连接时，队列将会被删除。自动删除队列允许的消费者没有限制，也就是说当这个队列上最后一个消费者断开连接才会执行删除  
自动删除队列只需要在声明队列时，设置属性auto-delete标识为true即可。系统声明的随机队列，缺省就是自动删除的  
##### 单消费者队列  
普通队列允许的消费者没有限制，多个消费者绑定到多个队列时，RabbitMQ会采用轮询进行投递。如果需要消费者独占队列，在队列创建的时候，设定属性exclusive为true  

### 自动过期队列  
指队列在超过一定时间没使用，会被从RabbitMQ中被删除。什么是没使用？一定时间内没有Get操作发生或者没有Consumer连接在队列上  
特别的：就算一直有消息进入队列，也不算队列在被使用  
通过声明队列时，设定x-expires参数即可，单位毫秒  

### 永久队列  
##### 队列的持久性
持久化队列和非持久化队列的区别是，持久化队列会被保存在磁盘中，固定并持久的存储，当Rabbit服务重启后，该队列会保持原来的状态在RabbitMQ中被管理，而非持久化队列不会被保存在磁盘中，Rabbit服务重启后队列就会消失  
非持久化比持久化的优势就是，由于非持久化不需要保存在磁盘中，所以使用速度就比持久化队列快。即是非持久化的性能要高于持久化。而持久化的优点就是会一直存在，不会随服务的重启或服务器的宕机而消失  
在声明队列时，将属性durable设置为'false'，则该队列为非持久化队列，设置成'true'时，该队列就为持久化队列  

##### 队列级别消息过期  
就是为每个队列设置消息的超时时间。只要给队列设置x-message-ttl 参数，就设定了该队列所有消息的存活时间，时间单位是毫秒。如果声明队列时指定了死信交换器，则过期消息会成为死信消息  
当队列消息的TTL 和消息TTL都被设置，时间短的TTL设置生效。如果将一个过期消息发送给RabbitMQ，该消息不会路由到任何队列，而是直接丢弃  
为消息设置TTL有一个问题：RabbitMQ只对处于队头的消息判断是否过期（即不会扫描队列），所以，很可能队列中已存在死消息，但是队列并不知情。这会影响队列统计数据的正确性，妨碍队列及时释放资源  

### 延时队列
顾名思义，延迟队列是指被延迟消费的队列。而一般的队列，消息一旦入队了之后就会被消费者马上消费  
RabbitMQ延迟队列，主要是借助消息的TTL（Time to Live）和死信exchange（Dead Letter Exchanges：DLX）来实现  

##### 什么时候会用到延时队列？
- 延迟消费  
在订单系统中，一个用户 下单之后通常有30分钟的时间进行支付，如果30分钟之内没有支付成功 ，那么这个订单将进行异常处理（如： 关闭订单 ），这时就可以使用延迟队列来处理这些订单  
- 延迟重试  
主要是处理一些异常信息，如：发送失败、消费失败，可能当时由于网络抖动的问题，暂时无法访问，需要延迟再进行重试（重新发送，重新消费）  

##### 消息TTL/队列TTL的区别  
- 队列TTL  
通过队列的属性来设置TTL ， 队列中的所有消息都有相同的过期时间  
一旦消息过期，就会立即从队列中抹去 （ 因为过期的消息肯定处于队列的头部 ）  
- 消息TTL   
对消息进行单独设置 ，每条消息的TTL可以设置不同的过期时间  
即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的  
- 分析  
第一种方法里，队列中已过期的消息肯定在队列头部 ，RabbitMQ只要定期从队头开始扫描是否有过期消息即可  
第二种方法里，每条消息的过期时间不同，如果要删除所有过期消息，势必要扫描整个队列，所以不如等到此消息即将被投递时再判定是否过期，如果过期，再进行删除  

##### 实践
- 延迟消费  
例如：生产者发送消息到Queue_10s，预先设置Queue_10s的队列TTL为10s，当消息过期后，经过DLX，进入Dead_Queue。消费者则消费Dead_Queue中的消息，此时的消息则是延迟消费  
- 延迟重试  
生产者发送消息到普通队列，消费者正常消费队列  
生产者1发送失败，发送消息到Queue_20s，消息过期后，经过DLX，进入Dead_Queue，生产者2订阅DLX，获取数据再次尝试发送
消费者1消费失败，发送消息到Queue_20s，消息过期后，经过DLX，进入Dead_Queue，消费者2订阅DLX，获取数据再次尝试消费

### 保证消息不丢失方案
- 消息生产者开启事务机制或者publisher confirm机制，以确保消息可以可靠传输到RabbitMQ中  
- 消息和队列都要持久化处理，以确保RabbitMQ服务器故障时不会丢失消息  
- 消费者要将autoAck设置为false，然后通过手动确认的方式去确认已经正确消费的消息，以避免在消费端引起不必要的消息丢失  
- 备份交换器  
通过声明交换器(channel.exchangeDeclare)时添加 alternate-exchange参数 来实现，也可通过Policy(策略)的方式实现，如果备份交换器和mandatory参数一同使用，则前者的优先级更高  
备份交换器可以在不设置mandatory参数的情况下，将未路由的消息存储在RabbitMQ中（未路由成功的消息通过备份交换器，路由到其绑定队列），再在需要的时候去处理这些消息  
如果没有设置mandatory参数，消息在未路由成功的情况下将会丢失；如果设置了mandatory参数，那么需要添加ReturnListener逻辑，生产者的代码将变复杂  