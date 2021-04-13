---
layout:     post
title:      "就决定是你了，Kafka-01"
subtitle:   "概念、基本原理、消费者/生产者代码"
date:       2021-01-11
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka.png"
tags:
    - Java
    - 消息队列
---
> 部分资料来源于网络上各位前辈（诗心客、柳树的絮叨叨、芋道源码等）

### 前言
决定开始整理Kafka相关的东西的时候，大概是在去年年初，前前后后拖了这么久。俗话说，好记性不如烂笔头，整理的过程也算是一个重新认识的过程吧，话不多说，直接开搞
### 概念和基本功能
##### Kafka是个啥？

官方对于kafka的说明是这样的：

>Kafka is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.

核心关注的东西：**这是一个实时数据处理系统，可以横向扩展、高可靠，而且还变态快**，在这之前，我的印象里只有在大数据的情况下才会使用kafka。

Kafka由LinkedIn公司通过Scala语言开发，并捐献给Apache基金会了。我们常常听到的kafka其实更多的是它的消息队列的功能，其实kafka是一个分布式的流处理平台，处理和管理数据流向的平台(消息队列只是其中一个功能)。

##### 基本功能
- 消息系统  
有发布订阅功能的消息队列，此功能与ActiveMQ、RibbitMQ、RocketMQ类似  
- 流处理  
在实时系统中记录事件数据  
- 数据流存储  
在分布式、可复制备份的、可容错的集群中安全的存储数据流  

##### 使用场景  
- 活动跟踪  
Kafka 可以用来跟踪用户行为，比如我们经常回去淘宝购物，你打开淘宝的那一刻，你的登陆信息，登陆次数都会作为消息传输到 Kafka ，当你浏览购物的时候，你的浏览信息，你的搜索指数，你的购物爱好都会作为一个个消息传递给 Kafka ，这样就可以生成报告，可以做智能推荐，购买喜好等。  
- 传递消息  
Kafka 另外一个基本用途是传递消息，应用程序向用户发送通知就是通过传递消息来实现的，这些应用组件可以生成消息，而不需要关心消息的格式，也不需要关心消息是如何发送的。  
- 度量指标  
Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。  
- 日志记录  
Kafka 的基本概念来源于提交日志，比如我们可以把数据库的更新发送到 Kafka 上，用来记录数据库的更新时间，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。  
- 流式处理  
流式处理是有一个能够提供多种应用程序的领域。  
- 限流削峰  
Kafka 多用于互联网领域某一时刻请求特别多的情况下，可以把请求写入Kafka 中，避免直接请求后端程序导致服务崩溃。
  
##### 优势  
- 高性能  
高吞吐量，单机可达到百万级的TPS写入  
低延迟，时效性高  
- 高可用  
它天然支持分布式，通过分区和备份策略，即使少数机器或节点宕机，也不会丢失数据，不会导致系统不可用  
- 消息有序性  
采用pull方式获取消息，因为消息存储是顺序存储，传输也是按顺序传输，因此通过控制可以保证消息被有序消费  
- 在大数据领域和日志领域比较成熟，与其他大数据组件容易集成  
kafka已成为大数据的重要组件，与zookeeper等组件配合，在大数据领域应用非常广泛，技术非常成熟  
- 事务消息的支持  
在kafka中使用`@Transactional`或`KafkaTemplate`的`executeInTransaction`方法很容易实现事务消息  

### 基本原理
写到这里，我开始想一个问题：**为什么需要消息中间件？**

- **解耦消息的生产和消费**  
- **缓冲**

前面一个比较好理解，这里说一下缓冲具体是什么意思。通过解耦，消费者在消费数据时更加的灵活，不必每次消息一产生就要马上去处理（虽然通常消费者侧也会有线程池等缓冲机制），可以等自己有空了的时候，再过来消息中间件这里取数据进行处理。这就是消息中间件带来的缓冲作用

##### 角色说明
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka03.jpg)  
<center>kafka的角色示意图</center>

- 角色说明  
1. **broker**：kafka集群中包含一个或者多个服务实例（节点），这种服务实例被称为broker（一个broker就是一个节点/一个服务器）  
2. **topic**：每条发布到kafka集群的消息都属于某个类别，这个类别就叫做topic  
3. **partition**:partition是一个物理上的概念，每个topic包含一个或者多个partition，发布者发到某个topic的消息会根据指定的规则被均匀的分布到其下的多个partition。一个broker服务下，可以创建多个分区，broker数与分区数没有关系。在kafka中，每一个分区会有一个编号：编号从0开始。每一个分区内的数据是有序的，但全局的数据不能保证是有序的。（有序是指生产什么样顺序，消费时也是什么样的顺序）  
4. **segment**：一个partition当中存在多个segment文件段，每个segment分为两部分`.log`文件和`.index`文件  
5. **producer**：消息的生产者，负责发布消息到kafka的broker中  
6. **consumer**：消息的消费者，向kafka的broker中读取消息的客户端  
7. **consumergroup**：消费者组，每一个consumer属于一个特定的consumergroup（每个消费者组都有一个ID，即group ID，可以为每个consumer指定group ID）,**同一个组中的消费者对于同一条消息只消费一次，组内的所有消费者协调在一起来消费一个订阅主题( topic)的所有分区(partition)，`每个分区只能由同一个消费组内的一个消费者(consumer)来消费，但每个分区是可以由不同的消费组同时来消费的`**  
8. **.log**：存放生产者发送的数据文件  
9. **.index**：存放`.log`文件的数据索引值，用于加快数据的查询速度

具体的详细角色内容介绍，可以参考下一篇内容:[就决定是你了，Kafka-02](https://www.threejinqiqi.fun/2021/01/13/java-message-kafka/)  

- 特征  
1. kafka支持消息持久化  
2. 消费端是主动拉取数据，消费状态和订阅关系由客户端负责维护  
3. **消息消费完后，不会立即删除，会保留历史消息**  
4. **topic中所有的分区的消费顺序是随机的但是，在单个分区内的消费顺序是固定的**

##### 角色理解
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka01.jpg)  

以topic+分区进行组织，每一个topic可以创建多个分区，每一个分区包含单独的文件夹，并且是多副本机制，即topic的每一个分区会有Leader与Follower，并且Kafka内部有机制保证topic的某一个分区的Leader与follow不会存在在同一台机器，并且每一台broker会尽量均衡的承担各个分区的Leader，当然在运行过程中如果不均衡，可以执行命令进行手动重平衡。Leader节点承担一个分区的读写，follow节点只负责数据备份。

- **Leader与Follower**  
1. Kafka 的负载均衡主要依靠分区 Leader 节点的分布情况，分区的Leader节点负责读写，而Follower节点负责数据同步

2. 如果Leader分区所在的Broker机器发生宕机，会触发主从节点的切换，会在剩下的follow节点中选举产生一个新的Leader节点

3. 如果某一个分区有三个副本因子，就算其中一个挂掉，那么只会剩下的两个中，选择一个leader。此时不会在其他的broker中另启动一个副本（因为在另一台启动的话，存在数据传递，只要在机器之间有数据传递，就会长时间占用网络IO），这里涉及到**同步副本**、**非同步副本**、**不完全首领选举**

4. ack的作用：①ack=0代表不等broker端确认就直接返回，即客户端将消息发送到网络中就返回发送成功②ack=1代表Leader节点接受并存储后向客户端返回成功③ack=-1代表Leader节点和所有的Follower节点接受并成功存储再向客户端返回成功

- **partition与consumergroup**  
1. **partition数量决定了每个consumer group中并发消费者的最大数量**

2. 某一个主题下的分区数，对于消费该主题的同一个消费组下的消费者数量，应该小于等于该主题下的分区数

3. 分区数越多，同一时间可以有越多的消费者来进行消费，消费数据的速度就会越快，提高消费的性能

- **partition replicas与ISR**

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka04.jpg) 

1. 副本因子/副本数（replication-factor）：控制消息保存在几个broker（服务器）上，一般情况下副本数等于broker的个数

2. 副本因子操作以分区为单位的。每个分区都有各自的主副本（Leader）和从副本（Follower）

3. 处于同步状态的副本（当前可用的副本）叫做in-sync-replicas(ISR)

##### 消息写入时发生了什么？
- 正常过程  
1. kafka以topic来进行消息管理，每个topic包含多个partition，每个partition对应一个逻辑log，由多个segment组成

2. 每个segment中存储多条消息，消息id由其逻辑位置决定，即从消息id可直接定位到消息的存储位置，避免id到位置的额外映射

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka02.jpg) 

3. 每个part在内存中对应一个index，记录每个segment中的第一条消息偏移

4. 发布者发到某个topic的消息会被均匀的分布到多个partition上（或根据用户指定的路由规则进行分布），broker收到发布消息后会往对应partition的最后一个segment上添加该消息，当某个segment上的消息条数达到配置值或消息发布时间超过阈值时，segment上的消息会被flush到磁盘，只有flush到磁盘上的消息订阅者才能订阅到，segment达到一定的大小后将不会再往该segment写数据，broker会创建新的segment

- 负载均衡  
1. producer可以自定义发送到哪个partition的路由规则。默认路由规则：hash(key)%numPartitions，如果key为null则随机选择一个partition

2. 自定义路由：如果key是一个user id，可以把同一个user的消息发送到同一个partition，这时consumer就可以从同一个partition读取同一个user的消息

##### Kafka 为何如此之快？
Kafka 实现了零拷贝原理来快速移动数据，避免了内核之间的切换。Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据。  
批处理能够进行更有效的数据压缩并减少 I/O 延迟，Kafka **采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费**  
总结起来就是：**顺序读写、零拷贝、消息压缩、分批发送**

### 实战-消费者  
理论是认识一个技术的基础，实战才是硬道理，这里以SpringBoot继承kafka为例  
##### 依赖
```xml
<dependency>
	<groupId>org.springframework.kafka</groupId>
	<artifactId>spring-kafka</artifactId>
	<!--<version>2.2.5.RELEASE</version>-->
</dependency>
```
##### 配置类
```java
import org.apache.commons.lang3.StringUtils;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.RoundRobinAssignor;
import org.apache.kafka.common.serialization.IntegerDeserializer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.util.CollectionUtils;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Value("${kafka.consumer.group-id}")
    private String kafkaConsumerGroupId;
    @Value("${kafka.consumer.auto-offset-reset}")
    private String kafkaConsumerAutoOffsetReset;
    @Value("${kafka.consumer.enable-auto-commit}")
    private boolean kafkaConsumerEnableAutoCommit;
    @Value("${kafka.consumer.session-timeout-ms-config}")
    private String kafkaConsumerSessionTimeoutMsConfig;
    @Value("${kafka.consumer.heartbeat-interval-ms-config}")
    private String kafkaConsumerHeartbeatIntervalMsConfig;
    @Value("${kafka.consumer.max-poll-records-config}")
    private int kafkaConsumerMaxPollRecordsConfig;
    @Value("${kafka.consumer.auto-commit-interval}")
    private int kafkaConsumerAutoCommitInterval;
    @Value("${kafka.listener.concurrency}")
    private int kafkaListenerConcurrency;

    /**
     * 不使用spring boot默认方式创建的DefaultKafkaConsumerFactory，重新定义创建方式
     * @return
     */
    @Bean("defaultConsumerFactory")
    public DefaultKafkaConsumerFactory defaultConsumerFactory(){
        try {
            return new DefaultKafkaConsumerFactory(consumerProperties());
        } catch (BusinessException e) {
            throw e;
        } catch (Exception e) {
            //do something
        }
    }

    @Bean("batchListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory BatchListenerContainerFactory(DefaultKafkaConsumerFactory consumerFactory) {
        ConcurrentKafkaListenerContainerFactory factory = new ConcurrentKafkaListenerContainerFactory();
        //指定使用DefaultKafkaConsumerFactory
        factory.setConsumerFactory(consumerFactory);
        // 连接池中消费者数量
        factory.setConcurrency(kafkaListenerConcurrency);
        // 开启并发消费
        factory.setBatchListener(Boolean.TRUE);
        log.info("Kafka batchListenerContainerFactory build successfully!!");
        return factory;
    }

    /**
     * 构造消费者属性map
     * @return
     */
    private Map<String, Object> consumerProperties(){
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.GROUP_ID_CONFIG, kafkaConsumerGroupId);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, kafkaConsumerEnableAutoCommit);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, kafkaConsumerAutoOffsetReset);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, kafkaConsumerAutoCommitInterval);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, kafkaConsumerSessionTimeoutMsConfig);
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, kafkaConsumerHeartbeatIntervalMsConfig);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, RoundRobinAssignor.class.getName());
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, kafkaConsumerMaxPollRecordsConfig);
        //建议动态获取集群的地址
        List<String> bootstrapServers=XXXX;
        if (CollectionUtils.isEmpty(bootstrapServers)) {
            //do something
        }
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return props;
    }
}
```
##### 配置类补充
- consumer参数的一些配置

```java
public static class Consumer {

   private final Ssl ssl = new Ssl();

   /**
    * 自动提交间隔时间 设为自动提交时有效
    * 默认 5000 ms
    * Frequency with which the consumer offsets are auto-committed to Kafka if
    * 'enable.auto.commit' is set to true.
    */
   private Duration autoCommitInterval;

   /**
    * 当没有初始偏移量时从何处读取
    * 默认 latest
    * earliest:从分区开始位置读取
​​​​​​​    * latest：从分区末尾开始读数据
    * What to do when there is no initial offset in Kafka or if the current offset no
    * longer exists on the server.
    */
   private String autoOffsetReset;

   /**
    * kafka集群地址，多个用逗号隔开
    * Comma-delimited list of host:port pairs to use for establishing the initial
    * connection to the Kafka cluster.
    */
   private List<String> bootstrapServers;

   /** 
    * 请求服务器时带的clientId
    * 通常用于服务器端日志
    * 默认 ""
    * ID to pass to the server when making requests. Used for server-side logging.
    */
   private String clientId;

   /**
    * 是否允许自动提交
    * 默认：true
    * Whether the consumer's offset is periodically committed in the background.
    */
   private Boolean enableAutoCommit;

   /**
    * 如果没有足够的数据立即满足“fetch.min.bytes”的要求，则服务器在响应fetch请求之前阻塞的最长时间
    * 默认 500 ms，即就算无法达到要求最小数据量 也要在500ms过后返回
    * Maximum amount of time the server blocks before answering the fetch request if
    * there isn't sufficient data to immediately satisfy the requirement given by
    * "fetch.min.bytes".
    */
   private Duration fetchMaxWait;

   /**
    * 一次fetch请求最小拉取数据量
    * 默认 1 byte
    * Minimum amount of data, in bytes, the server should return for a fetch request.
    */
   private Integer fetchMinSize;

   /**
    * group id
    * Unique string that identifies the consumer group to which this consumer
    * belongs.
    */
   private String groupId;

   /**
    * 预期的心跳间隔时间
    * 默认 3000 ms
    * Expected time between heartbeats to the consumer coordinator.
    */
   private Duration heartbeatInterval;

   /**
    * 反序列化的key
    * Deserializer class for keys.
    */
   private Class<?> keyDeserializer = StringDeserializer.class;

   /**
    * Deserializer class for values.
    */
   private Class<?> valueDeserializer = StringDeserializer.class;

   /**
    * 最大拉取记录数
    * 默认 500
    * Maximum number of records returned in a single call to poll().
    */
   private Integer maxPollRecords;
}
```

- **session.timeout.ms和heartbeat.interval.ms**

1. **consumer心跳机制**  
consumer定期向coordinator发送心跳请求，以表明自己还在线；如果session.timeout.ms内未发送请求，coordinator认为其不可用，然后触发rebalance  
2. 默认值  
session.timeout.ms：coordinator感知consumer崩溃所需时间，默认10秒  
heartbeat.interval.ms：consumer发送心跳请求间隔，默认3秒  
3. **建议**  
session.timeout.ms一定要大于heartbeat.interval.ms，否则消费者组会一直处于rebalance状态  
session.timeout.ms最好几倍于heartbeat.interval.ms；这是因为如果因为某一时间段的网络延迟导致coordinator未感知到心跳请求，session.timeout.ms和heartbeat.interval.ms接近的话，会导致consumer组rebalance过于频繁，影响消费性能  

##### 消费类

```java
@KafkaListener(id = "consumer-xxxx",topics = "${xxxxx}",containerFactory = "batchListenerContainerFactory")
public void receiveQueue(List<ConsumerRecord<?, ?>> records) {
    records.forEach(message->{
        //do something.根据生产者发送的消息体来进行消息获取，比如string时
        JSONObject jsonObject = JSONObject.parseObject((String)message.value());
    }
}
```

### 实战-生产者 
##### 依赖
```xml
<dependency>
	<groupId>org.springframework.kafka</groupId>
	<artifactId>spring-kafka</artifactId>
	<!--<version>2.2.5.RELEASE</version>-->
</dependency>
```
##### 配置类
```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.IntegerSerializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;


@Configuration
public class KafkaConfig {

    @Value("${kafka.bootstrap.servers}")
    private String bootstrapServersConfig;
    @Value("${kafka.producer.retries}")
    private String kafkaProducerRetries;
    @Value("${kafka.producer.linger.ms}")
    private String kafkaProducerLingerMs;
    @Value("${kafka.producer.batch.size}")
    private String kafkaProducerBatchSize;
    @Value("${kafka.producer.acks}")
    private String kafkaProducerAcks;

    //生产者个性化配置
    public Producer<String ,String> producerConfigs(){
        Map<String, Object> props = new HashMap<>();
        //建议动态获取集群的地址
        List<String> bootstrapServers=xxxxx;
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);//kafka集群地址
        props.put(ProducerConfig.ACKS_CONFIG, kafkaProducerAcks);//有0,1，all三种形式
        props.put(ProducerConfig.RETRIES_CONFIG, kafkaProducerRetries);
        props.put(ProducerConfig.LINGER_MS_CONFIG,kafkaProducerLingerMs);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG,kafkaProducerBatchSize);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class);// 发送消息的key，假如类型为String,就使用String类型的序列化器
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);//发送消息的value，假如类型为String,就使用String类型的序列化器
        return new KafkaProducer<String, String>(props);
    }
}
```
##### 生产类

```java
@Component
public class KafkaProductor {

    @Value("${xxxxx}")
    private String kafkaProducerTopic;

    @Autowired
    private KafkaConfig kafkaConfig;
    //Producer<String, String>需要与配置文件中的设置相对应，即KEY_SERIALIZER_CLASS_CONFIG和VALUE_SERIALIZER_CLASS_CONFIG
    private Producer<String, String> producer = null;

    public boolean send(String message) {
        if (producer == null) {
            this.producer = kafkaConfig.producerConfigs();
        }
        //ProducerRecord<String, String>需要与配置文件中的设置相对应，即KEY_SERIALIZER_CLASS_CONFIG和VALUE_SERIALIZER_CLASS_CONFIG
        producer.send(new ProducerRecord<String, String>(this.kafkaProducerTopic, message));
        return true;
    }
}
```

