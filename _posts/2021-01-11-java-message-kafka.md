---
layout:     post
title:      "就决定是你了，Kafka"
subtitle:   "一些概念、消费者/生产者代码"
date:       2021-01-11
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-message-rmq-bk.jpg"
tags:
    - Java
    - 消息队列
---
> 资料来源于网络上各位前辈（诗心客等）的总结

### 前言
决定开始整理Kafka相关的东西的时候，大概是在去年年初，前前后后拖了这么久。俗话说，好记性不如烂笔头，整理的过程也算是一个重新认识的过程吧，话不多说，直接开搞。

### 概念和基本功能
##### Kafka是个啥？
Kafka由LinkedIn公司通过Scala语言开发，并捐献给Apache基金会了。我们常常听到的kafka其实更多的是它的消息队列的功能，其实kafka是一个分布式的流处理平台，处理和管理数据流向的平台(消息队列只是其中一个功能)。

##### 基本功能
- 消息系统  
有发布订阅功能的消息队列，此功能与ActiveMQ、RibbitMQ、RocketMQ类似  
- 流处理  
在实时系统中记录事件数据  
- 数据流存储  
在分布式、可复制备份的、可容错的集群中安全的存储数据流  

##### 使用场景  
- 消息系统  
作为一个消息队列系统，提供消息订阅与发布  
- 网站用户行为追踪  
用户操作记录与追踪  
- 数据指标监控  
将分布式应用程序中的统计数据聚合到一起，形成一个统计中心  
- 日志收集聚合  
将分布式系统中的日志数据聚合到一起，以便统一管理、监控  
- 数据流处理  
在处理由多个阶段组成的管道时处理数据，其中原始输入数据从Kafka主题中消费，然后聚合、扩展或以其他方式转换为新主题以供进一步消费或后续处理  
- 事件状态追踪  
记录事件源的状态变更  
- 分布式系统日志提交  
作为分布式系统的一种外部提交日志，该日志有助于在节点之间复制数据，并充当故障节点恢复其数据的重新同步机制  
  
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

