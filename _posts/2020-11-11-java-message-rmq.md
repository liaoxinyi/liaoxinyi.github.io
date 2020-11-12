---
layout:     post
title:      "这只兔子该怎么玩儿？"
subtitle:   "RabbitMQ消息中间件的积累-待更新RMQ的事务"
date:       2020-11-11
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-rmq-bk.jpg"
tags:
    - Java
    - Message
---
> 资料来源于网络上各位前辈的总结，比如大佬【高广超】、【纯洁的微笑】等

### 前言
之前在线上环境中遇到消息堆积的问题，排查出来原因是消费者抛异常导致AKS未反馈。当时遇到这个问题的时候，才想起来，其实还没有真正梳理一下有关RabbitMQ的一些东西，所以一直计划着写这一篇文章。后面遇到有关的问题，也会持续在这篇文章基础上进行更新  
### 基本介绍介绍
##### 消息中间件
依旧是老样子，一个产品的产生必定是为了解决某一个场景问题。对于一个大型的软件系统来说，它会有很多的组件或者说模块或者说子系统或者（subsystem or Component or submodule）。那么这些模块的如何通信？这和传统的IPC有很大的区别。传统的IPC很多都是在单一系统上的，模块耦合性很大，不适合扩展（Scalability）；如果使用socket那么不同的模块的确可以部署到不同的机器上，但是还是有很多问题需要解决。比如信息的发送者和接收者如何维持这个连接，如果一方的连接中断，这期间的数据如何方式丢失？如何降低发送者和接收者的耦合度？如何做到load balance？有效均衡接收者的负载？等等  
消息中间件最主要的作用是解耦，中间件最标准的用法是生产者生产消息传送到队列，消费者从队列中拿取消息并处理，生产者不用关心是谁来消费，消费者不用关心谁在生产消息，从而达到解耦的目的。在分布式的系统中，消息队列也会被用在很多其它的方面，比如：分布式事务的支持，RPC的调用等等  
##### AMQP
虽然同步消息通讯有很多公开标准，如 COBAR的 IIOP ，或者是 SOAP 等。但是在异步消息处理中却不是这样，只有大企业有一些商业实现（如微软的 MSMQ ，IBM 的 Websphere MQ 等），因此，在 2006 年的 6 月，Cisco 、Redhat、iMatix 等联合制定了 AMQP 的公开标准。AMQP，即Advanced Message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全  
下面是以RMQ为例的AMQP的大致模型：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-amqp-structure.jpg)
<center>RabbitMQ实现的AMQP</center>  
- **RabbitMQ Server**  
也叫broker server，**它不是运送食物的卡车，而是一种传输服务**。原话是RabbitMQ isn’t a food truck, it’s a delivery service。他的角色就是维护一条从Producer到Consumer的路线，保证数据能够按照指定的方式进行传输。但是这个保证也不是100%的保证，但是对于普通的应用来说这已经足够了。当然对于商业系统来说，可以再做一层数据一致性的guard，就可以彻底保证系统的一致性了  
- **Client P**  
也叫Producer，数据的发送方。createmessages and publish (send) them to a broker server (RabbitMQ).一个Message有两个部分：payload（有效载荷）和label（标签）。payload顾名思义就是传输的数据。label是exchange的名字或者说是一个tag，它描述了payload，而且RabbitMQ也是通过这个label来决定把这个Message发给哪个Consumer。**AMQP仅仅描述了label，而RabbitMQ决定了如何使用这个label的规则**  
- **Client C**  
也叫Consumer，数据的接收方。Consumersattach to a broker server (RabbitMQ) and subscribe to a queue。把queue比作是一个有名字的邮箱。当有Message到达某个邮箱后，RabbitMQ把它发送给它的某个订阅者即Consumer。当然可能会把同一个Message发送给很多的Consumer。在这个Message中，只有payload，label已经被删掉了。对于Consumer来说，它是不知道谁发送的这个信息的。就是协议本身不支持。但是当然了如果Producer发送的payload包含了Producer的信息就另当别论了  

##### RabbitMQ
RabbitMQ是一个开源的AMQP实现，最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗  
一般消息队列都离不开三个东西：发消息者、队列和收消息者。RabbitMQ在此基础之上又继续增加了一个交换机（有些时候也被称为路由器）角色，这样发消息者和队列就没有直接联系, 转而变成发消息者把消息给交换机, 交换机根据调度策略再把消息再给队列。因此在RabbitMQ里面，有四个东西需要值得注意：**虚拟主机（Virtual Host），交换机（Exchange），队列（Queue），和绑定（Binding）**  
当然，有一些文章里面也把整个消息体分得比较详细，具体如下图：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-rmq-structure.jpg)
<center>RabbitMQ的消费模型</center>
- **虚拟主机（Virtual Host）**  
一个虚拟主机持有一组交换机、队列和绑定。为什么需要多个虚拟主机呢？很简单，RabbitMQ当中，用户只能在虚拟主机的粒度进行权限控制。 因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机，每一个RabbitMQ服务器都有一个默认的虚拟主机。每个virtual host本质上都是一个RabbitMQ Server，拥有它自己的queue，exchagne，和bings rule等等。这保证了你可以在多个不同的application中使用RabbitMQ    
- **交换机（Exchange）和队列（Queue）**  
交换机只会用于转发消息，不会做存储，如果没有队列绑定到这个交换机的话，它会直接丢弃掉生产者发送过来的消息，在启用ack模式后，交换机找不到队列会返回错误    
- **绑定（Binding）**  
既然说到了绑定，那是靠什么绑定的呢？靠的就是**路由键**，消息到交换机的时候，交换机会转发到对应的队列中，那么究竟转发到哪个队列，就要根据该路由键。至于绑定，就是交换机需要和队列相绑定，**注意是多对多关系，不一定只会一对多**  
1. binding key：在绑定（Binding）Exchange与Queue的同时，一般会指定一个binding key。在绑定多个Queue到同一个Exchange的时候，这些Binding允许使用相同的binding key  
2. routing key：这个就是我们说得最多的路由键，消费者将消息发送给Exchange时，一般会指定一个routing key  
  
- **连接（Connection）**
就是一个TCP的连接。Producer和Consumer都是通过TCP连接到RabbitMQ Server的。以后我们可以看到，程序的起始处就是建立这个TCP连接 
- **通道（Channel）**
虚拟连接。它建立在上述的TCP连接中。数据流动都是在Channel中进行的。也就是说，一般情况是程序起始建立TCP连接，第二步就是建立这个Channel。代码层次中，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等  
**那么，为什么使用Channel，而不是直接使用TCP连接？**  
对于OS来说，建立和关闭TCP连接是有代价的，频繁的建立关闭TCP连接对于系统的性能有很大的影响，而且TCP的连接数也有限制，这也限制了系统处理高并发的能力。但是，在TCP连接中建立Channel是没有上述代价的。对于Producer或者Consumer来说，可以并发的使用多个Channel进行Publish或者Receive  
- **队列（Queue）**  
这里有一个很有意思的点，多个消费者可以订阅同一个Queue，这时Queue中的消息会被平均分摊给多个消费者进行处理，而**不是每个消费者都收到所有的消息并处理**   

### 交换机类型
##### Direct
该类型是默认的交换机模式，其行为是”先匹配, 再投送”，即在绑定时设定一个routing_key, 消息的routing_key**全文匹配**时, 才会被交换器投送到绑定的队列中去。Direct Exchange可以使用默认的Exchange且默认的Exchange会绑定所有的队列，所以Direct可以直接使用Queue名（作为routing key ）绑定  
##### Topic
topic类型的Exchange在匹配规则上进行了扩展，它与direct类型的Exchage相似，也是将消息路由到binding key与routing key相匹配的Queue中，但这里的匹配规则有些不同，它约定：  
- binding key：用英文格式的句号`.`隔开的字符串，被句点号`.`分隔开的每一段独立的字符串称为一个单词  
- routing key：用英文格式的句号`.`隔开的字符串，除此之外，还可以存在两种特殊字符`*`与`#`，用于做模糊匹配，其中`*`用于匹配一个单词，`#`用于匹配多个单词（可以是零个）   

##### Headers  
headers 也是根据规则匹配, 相较于 direct 和 topic 固定地使用 routing_key , headers 则是一个自定义匹配规则的类型  
在队列与交换器绑定时, 会设定一组键值对规则, 消息中也包括一组键值对( headers 属性), 当这些键值对有一对, 或全部匹配时, 消息被投送到对应队列  
##### Fanout  
Fanout Exchange 消息广播的模式，不管路由键或者是路由模式，会把消息发给绑定给它的全部队列，如果配置了routing_key会被忽略  

**AMQP规范里还提到两种Exchange Type，分别为system与自定义**

### 队列和消息的那些事
##### TTL队列/消息
TTL（time to live）指生存时间  
- 支持消息的过期时间，在消息发送时可以指定  
- 支持队列过期时间，在消息入队列开始计算时间，只要超过了队列的超时时间配置，那么消息就会自动的清除  

##### 死信队列
死信队列：DLX，Dead-Letter-Exchange，利用DLX，当消息在一个队列中变成死信（dead message，就是没有任何消费者消费）之后，他能被重新publish到另一个Exchange，这个Exchange就是DLX  
- 消息变为死信的几种情况    
1. 消息被拒绝（basic.reject/basic.nack）同时requeue=false（不重回队列）  
2. TTL过期  
3. 队列达到最大长度  
- 死信队列的设置  
1. 设置Exchange和Queue，然后进行绑定  
2. Exchange: dlx.exchange(自定义的名字)  
3. queue: dlx.queue（自定义的名字）  
4. routingkey: #（#表示任何routingkey出现死信都会被路由过来）  
5. 最后在然后正常的声明交换机、队列、绑定时，在队列的配置参数上加上一个参数：`arguments.put("x-dead-letter-exchange","dlx.exchange(自定义的名字)");`  

DLX也是一个正常的Exchange，和一般的Exchange没有任何的区别，他能在任何的队列上被指定，实际上就是设置某个队列的属性。当这个队列出现死信的时候，RabbitMQ就会自动将这条消息重新发布到Exchange上去，进而被路由到另一个队列。可以监听这个队列中的消息作相应的处理，这个特性可以弥补rabbitMQ以前支持的immediate参数的功能  

##### 消息回执（Message acknowledgment）
在实际应用中，可能会发生消费者收到Queue中的消息，但没有处理完成就宕机（或出现其他意外）的情况，这种情况下就可能会导致消息丢失。为了避免这种情况发生，我们可以要求消费者在消费完消息后发送一个回执给RabbitMQ，RabbitMQ收到消息回执（Message acknowledgment）后才将该消息从Queue中移除；如果RabbitMQ没有收到回执并检测到消费者的RabbitMQ连接断开，则RabbitMQ会将该消息发送给其他消费者（如果存在多个消费者）进行处理。这里不存在timeout概念，一个消费者处理消息时间再长也不会导致该消息被发送给其他消费者，除非它的RabbitMQ连接断开  
##### 消息持久化（Message durability）
如果我们希望即使在RabbitMQ服务重启的情况下，也不会丢失消息，我们可以将Queue与Message都设置为可持久化的（durable），这样可以保证绝大部分情况下我们的RabbitMQ消息不会丢失。但依然解决不了小概率丢失事件的发生（比如RabbitMQ服务器已经接收到生产者的消息，但还没来得及持久化该消息时RabbitMQ服务器就断电了），如果需要把这种小概率事件也要管理起来，那么就要要用到事务了  
##### Prefetch count  
如果有多个消费者同时订阅同一个Queue中的消息，Queue中的消息会被平摊给多个消费者。这时如果每个消息的处理时间不同，就有可能会导致某些消费者一直在忙，而另外一些消费者很快就处理完手头工作并一直空闲的情况。我们可以通过设置prefetchCount来限制Queue每次发送给每个消费者的消息数，比如设置prefetchCount=1，则Queue每次给每个消费者发送一条消息；消费者处理完这条消息后Queue会再给该消费者发送一条消息  

### 其他
##### RPC（Remote Procedure Call）
MQ本身是基于异步的消息处理，前面的示例中所有的生产者（P）将消息发送到RabbitMQ后不会知道消费者（C）处理成功或者失败（甚至连有没有消费者来处理这条消息都不知道）。但实际的应用场景中，很可能需要一些同步处理，需要同步等待服务端将消息中间件的消息处理完成后再进行下一步处理。这相当于RPC（Remote Procedure Call，远程过程调用）  
RabbitMQ中实现RPC的机制是：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-rmq-rpc.jpg)
<center>RabbitMQ的RPC机制</center>  
- 客户端发送请求（消息）时，在消息的属性（MessageProperties，在AMQP协议中定义了14中properties，这些属性会随着消息一起发送）中设置两个值replyTo（一个Queue名称，用于告诉服务器处理完成后将通知我的消息发送到这个Queue中）和correlationId（此次请求的标识号，服务器处理完成后需要将此属性返还，客户端将根据这个id了解哪条请求被成功执行了或执行失败）  
- 服务器端收到消息并处理  
- 服务器端处理完消息后，将生成一条应答消息到replyTo指定的Queue，同时带上correlationId属性  
- 客户端之前已订阅replyTo指定的Queue，从中收到服务器的应答消息后，根据其中的correlationId属性分析哪条请求被执行了，根据执行结果进行后续业务处理  

### RabbitMQ使用-消费者

**关于队列的声明，如果使用同一套参数进行声明了，就不能再使用其他参数来声明，要么删除该队列重新删除，可以使用命令行删除也可以在RabbitMQ Management上删除，要么给队列重新起一个名字**  

###### springframework.amqp.rabbit方式
1. 自定义RabbitMQ的配置类  
```java
@Configuration
public class RabbitMqConfig {
    //1.自定义消息队列
    @Bean
    public Queue lsqQueue() {
        Map<String, Object> arguments=new HashMap<>();
        //设置消息过期时间，单位为毫秒(消息直接在列队头部，定期扫描然后移除)
        arguments.put("x-message-ttl",647);
        /*其他Arguments参数：
        expiration：消息在消费的时候判定是不是过期了然后在删除
        x-expires：队列在多长时间没有被使用后删除
        x-max-length：加入queue中消息的条数。先进先出原则，超过10条后面的消息会顶替前面的消息*/
        //队列的其他属性：是否持久化等可参考Queue的构造函数，这里使用默认值
        return new Queue("自定义队列的名称",true, false, false,arguments);
    }
    //2.需要订阅的exchange
    @Bean
    DirectExchange lsqExchange() {
        //根据需要订阅的exchange的类型来选择DirectExchange/TopicExchange
        return new DirectExchange("需要订阅的exchange");
    }
    //3.绑定路由和routingKey
    @Bean
    Binding lsqBinding() {
        return BindingBuilder.bind(lsqQueue()).to(lsqExchange()).with("订阅的exchange的RoutingKey");
    }
}
```

2. 消息接收类和方法  
@RabbitListener方式的消费方式很简单，只需要在一个被Spring容器管理的类中，在消费的方法上添加@RabbitListener注解即可：  
```java
@RabbitListener(queues = "上面配置类中已经定义好的自定义队列名称")
    //默认使用Message去消费
    public void receiveQueueFromLsq(Message message) {
        try {
            //处理逻辑
        } catch (Exception e) {
            //抛出ImmediateAcknowledgeAmqpException后，生产者也会受到本次消费的AKS，这样不会导致消息堆积
            //或者也可以直接记录日志，不抛出异常也可以
            throw new ImmediateAcknowledgeAmqpException("错误码", e);
        }
    }
```

###### Spring Cloud Stream方式
- 步骤  
maven坐标：  
```xml
<dependency> 
    <groupId>org.springframework.cloud</groupId> 
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    <version>X.X.X.RELEASE</version><!--版本号需自己指定--> 
</dependency>
```
1. 配置application.properties  

```txt
spring.cloud.stream.bindings.自定义的标识.destination=需要订阅的exchange
spring.cloud.stream.bindings.自定义的标识.group=消费者的队列
#消息的类型：application/json，缺失则为默认值null
#spring.cloud.stream.bindings.自定义的标识.contentType=application/json
spring.cloud.stream.rabbit.bindings.自定义的标识.consumer.bindingRoutingKey=订阅的exchange的RoutingKey
#exchange的类型
spring.cloud.stream.rabbit.bindings.自定义的标识.consumer.exchangeType=direct
#持久化消息的最大存活时间，单位毫秒
spring.cloud.stream.rabbit.bindings.自定义的标识.consumer.ttl=647
```

2. 使用@Input完成通道配置  
```java
public interface RmqInputChannel {
    //名称与配置文件中【自定义的标识】保持一致
    String AA_BB = "自定义的标识";
   
    @Input(AA_BB)
    SubscribableChannel lsqMessageInput();
}
```

3. 消息接收类和方法  
```java
//绑定之前完成了配置的通道接口
@EnableBinding(RmqInputChannel.class)
public class RmqMsgService {
    //通过通道标识，对应识别去消费哪一个队列里面的数据，建议使用JSONObject来进行消费，也可以使用String
    @StreamListener(RmqInputChannel.AA_BB)
    public void receiveQueueFromAb(JSONObject message){
        try {
            //处理逻辑
        } catch (Exception e) {
            //抛出ImmediateAcknowledgeAmqpException后，生产者也会受到本次消费的AKS，这样不会导致消息堆积
            //或者也可以直接记录日志，不抛出异常也可以
            throw new ImmediateAcknowledgeAmqpException("错误码", e);
        }
    }
}
```

### RabbitMQ使用-生产者
###### Spring Cloud Stream方式
1. 配置application.properties  

```txt
spring.cloud.stream.bindings.自定义的标识.destination=需要绑定的exchange 
spring.cloud.stream.rabbit.bindings.自定义的标识.producer.routing-key-expression=其他人绑定这个exchange时需要的routing-key    
#exchange的类型，不指定默认是topic 
spring.cloud.stream.rabbit.bindings.自定义的标识.producer.exchangeType=direct
```

2. 使用@Output完成通道配置  
```java
public interface RmqOutPutChannel {
    //名称与配置文件中【自定义的标识】保持一致
    String AA_BB = "自定义的标识";
   
    @Output(AA_BB)
    MessageChannel lsqMessageOutPut();
}
```

3. 消息接收类和方法  
```java
//消费发送实现类
@EnableBinding(RmqOutPutChannel.class)
public class RmqSendMsgService {
    @Autowired
    RmqOutPutChannel source;

    public void send(DeviceMessage msg){
        JSONObject json = new JSONObject();
        json.put("message",msg);
        Message<JSONObject> message = MessageBuilder.withPayload(json).build();
        source.output().send(message);
    }
}
//消息发送
@Autowired
private RmqSendMsgService sender;
//...
sender.send(deviceMessage);
```