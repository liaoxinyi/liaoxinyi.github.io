---
layout:     post
title:      "就决定是你了，Kafka-01"
subtitle:   "基础概念、Kafka 为何如此之快、消费者(@KafkaListener、ConsumerConfig的几个常量参数)/生产者(ProducerRecord)代码"
update-date:       2021-06-28
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka.png"
tags:
    - Java
    - 消息队列
    - Kafka
---
> 2021.6.28 更新kafka的一些核心思想，或者说设计理念  
> 部分资料来源于网络上各位前辈（诗心客、柳树的絮叨叨、芋道源码、 武哥漫谈IT等）

### 前言
emsp;emsp;决定开始整理Kafka相关的东西的时候，大概是在去年年初，前前后后拖了这么久。俗话说，好记性不如烂笔头，整理的过程也算是一个重新认识的过程吧，话不多说，直接开搞
### 概念和基本功能
##### Kafka是个啥？
emsp;emsp;首先看一下这几个主流MQ的发展情况：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka-01-07.jpg)  
<center>几个MQ的发展历史</center>

官方对于kafka的说明是这样的：

> Kafka is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.  
> Apache Kafka is an open-source distributed event streaming platform.

emsp;emsp;核心关注的东西：**这是一个实时数据处理系统，可以横向扩展、高可靠，而且还变态快**，在这之前，我的印象里只有在需要大吞吐的消息中间件的情况下才会使用kafka，所以这不得不涉及到Kafka的一个诞生历史问题：  
emsp;emsp;Kafka 最开始其实是 Linkedin 内部孵化的项目，在设计之初是被当做「数据管道」，用于处理以下两种场景：  
- 运营活动场景：记录用户的浏览、搜索、点击、活跃度等行为  
- 系统运维场景：监控服务器的 CPU、内存、请求耗时等性能指标

emsp;emsp;所以特点是：**数据实时生产，而且数据量很大**

emsp;emsp;Linkedin 最初也尝试过用 ActiveMQ 来解决数据传输问题，但是性能无法满足要求，然后才决定自研 Kafka。了解了这个背景后，就不难理解 Kafka 与流数据的关系了。那为什么 Kafka 还被官方定义成流处理平台呢？它不就提供了一个数据通道能力吗，怎么还和平台扯上关系了？这是因为 Kafka 从 0.8 版本开始，就已经在提供一些和数据处理有关的组件了，比如：  
- Kafka Streams：一个轻量化的流计算库，性质类似于 Spark、Flink  
- Kafka Connect：一个数据同步工具，能将 Kafka 中的数据导入到关系数据库、Hadoop、搜索引擎中  

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
高吞吐量，单机可达到百万级的TPS写入；低延迟，时效性高  
- 高可用  
kafka本身支持分布式，通过分区和备份策略，即使少数机器或节点宕机，也不会丢失数据，不会导致系统不可用  
- 消息有序性  
采用`pull方式`获取消息，因为消息存储是顺序存储，传输也是按顺序传输，因此通过控制可以保证消息被有序消费  
- 在大数据领域和日志领域比较成熟，与其他大数据组件容易集成  
kafka已成为大数据的重要组件，与zookeeper等组件配合，在大数据领域应用非常广泛，技术非常成熟  
- 事务消息的支持  
在kafka中使用`@Transactional`或`KafkaTemplate`的`executeInTransaction`方法很容易实现事务消息  

##### 思想
这是2021.6.28更新的模块内容，目的是想记录一下Kafka中的一些设计理念。  
- 消息持久化  
emsp;emsp;这是个什么意思呢？正常情况，对于消息中间件的Topic模式，如果采用每增加一个消费者就复制一个队列的模式，对于Kafka的这种高吞吐的情形其实是不适用的，所以干脆直接将消息持久化起来。谁要用，谁来自己取。一定程度保证了消息的可靠，同时也把问题分散到了消费端或者生产端

- 分片存储  
emsp;emsp;这种，其实很像数据库庞大之后的分库分表，或者Redis庞大之后的分片集群。如果所有的持久化数据都挤在一坨的话，存取都是个麻烦事情，所以干脆来个分片存储，也就是都知道的**Partition（分区）**

emsp;emsp;Partition 主要是为了解决 Kafka 存储上的水平扩展问题，如果一个 Topic 的所有消息都只存在一个 Broker，这个 Broker 必然会成为瓶颈。因此，将 Topic 内的数据分成多个 Partition，然后分布到整个集群是很自然的设计方式

emsp;emsp;**划重点，Partition一般都是分布在多个Broker中的**

- 稀疏索引思想  
emsp;emsp;首先，在数据存储领域，有两个 “极端” 发展方向：**追求极致的读速度和追求极致的写速度**：  
    1. 加快读：通过索引（ B+ 树、二份查找树等方式），提高查询速度，但是写入数据时要维护索引，因此会降低写入效率  
    2. 加快写：纯日志型，数据以 append 追加的方式顺序写入，不加索引，使得写入速度非常高（理论上可接近磁盘的写入速度），但是缺乏索引支持，因此查询性能低

emsp;emsp;根据这两种方向，又衍生出来了三类最具代表性的底层索引结构：  
    1. 哈希索引：通过哈希函数将 key 映射成数据的存储地址，适用于等值查询等简单场景，对于比较查询、范围查询等复杂场景无能为力  
    2. B/B+ Tree 索引：最常见的索引类型，重点考虑的是读性能，它是很多传统关系型数据库，比如 MySQL、Oracle 的底层结构  
    3. LSM Tree 索引：数据以 Append 方式追加写入日志文件，优化了写但是又没显著降低读性能，众多 NoSQL 存储系统比如 BigTable，HBase，Cassandra，RocksDB 的底层结构
    
##### 基本原理
emsp;emsp;写到这里，我开始想一个问题：**为什么需要消息中间件？**，答案是下面两个：  
- **解耦消息的生产和消费**  
- **缓冲**

emsp;emsp;前面一个比较好理解，这里说一下缓冲具体是什么意思。通过解耦，消费者在消费数据时更加的灵活，不必每次消息一产生就要马上去处理（虽然通常消费者侧也会有线程池等缓冲机制），可以等自己有空了的时候，再过来消息中间件这里取数据进行处理。这就是消息中间件带来的缓冲作用

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

具体的详细角色内容介绍以及kafka是如何进行数据查询的，可以参考下一篇内容:[就决定是你了，Kafka-02](https://www.threejinqiqi.fun/2021/01/13/java-message-kafka-02/)  

- 特征  
1. kafka支持消息持久化  
2. 消费端是主动拉取数据，消费状态和订阅关系由客户端负责维护  
3. **消息消费完后，不会立即删除，会保留历史消息**  
4. **topic中所有的分区的消费顺序是随机的但是，在单个分区内的消费顺序是固定的**

##### 角色理解
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka01.jpg)  
<center>kafka的角色示意</center>  
emsp;emsp;每一个topic可以创建多个分区，每一个分区包含单独的文件夹，并且是多副本机制，即topic的每一个分区会有Leader与Follower，并且Kafka内部有机制保证topic的某一个分区的Leader与follow不会存在在同一台机器，并且每一台broker会尽量均衡的承担各个分区的Leader，当然在运行过程中如果不均衡，可以执行命令进行手动重平衡。Leader节点承担一个分区的读写，follow节点只负责数据备份  
- **Leader、Follower**  
    - Kafka 的负载均衡主要依靠分区 Leader 节点的分布情况，分区的Leader节点负责读写，而Follower节点负责数据同步  
    - 如果Leader分区所在的Broker机器发生宕机，会触发主从节点的切换，会在剩下的follow节点中选举产生一个新的Leader节点（这里有**同步副本**、**非同步副本**、**不完全首领选举**概念）  
    - 如果某一个分区有三个副本因子，就算其中一个挂掉，那么只会剩下的两个中，选择一个leader。此时不会在其他的broker中另启动一个副本（因为在另一台启动的话，存在数据传递，只要在机器之间有数据传递，就会长时间占用网络IO）  
- **ack**  
    - `ack=0`代表不等broker端确认就直接返回，即客户端将消息发送到网络中就返回发送成功  
    - `ack=1`代表发送过去，等待首领副本确认消息，认为成功。这里就有点意思了，首领肯定收到了消息，写入了分区文件，但是不一定全部都落盘了  
    - `ack=all`代表发送过去之后，消息被写入所有同步副本之后才会认为成功  
- **partition、consumergroup**  
    - **partition数量决定了每个consumer group中并发消费者的最大数量**  
    - 分区数越多，同一时间可以有越多的消费者来进行消费，消费数据的速度就会越快，提高消费的性能  
- **partition replicas、ISR**  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka04.jpg)  
<center>分区副本示意图</center>  
    - 副本因子/副本数（replication-factor）：控制消息保存在几个broker（服务器）上，一般情况下副本数等于broker的个数  
    - 副本因子操作以分区为单位的。每个分区都有各自的主副本（Leader）和从副本（Follower）  
    - 处于同步状态的副本叫做`in-sync-replicas(ISR)`

### Kafka 为何如此之快？
##### 零拷贝
Kafka 实现了零拷贝原理（`Zero-Copy零拷贝技术`）来快速移动数据，避免了内核之间的切换（`采用的是Java底层FileTransferTo方法`），先看看如果不是“零拷贝”会是什么样子？  
- **非零拷贝**
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka-01-05.jpg)  
<center>非零拷贝示意图</center>  
上面的内容可以表示为：  
1、read() 调用引发了一次从用户模式到内核模式的上下文切换。在内部，发出 sys_read()（或等效内容）以从文件中读取数据。直接内存存取（direct memory access，DMA）引擎从磁盘中读取文件内容，然后将它们存储到一个内核地址空间缓存区中  
2、所需的数据被从内核缓冲区拷贝到用户缓冲区，随后read()调用的返回引发了核模式到用户模式的上下文切换（又一次上下文切换）。现在数据被储存在用户地址空间缓冲区  
3、send() 调用引发了从用户模式到内核模式的上下文切换。数据被第三次拷贝，并被再次放置在内核地址空间缓冲区。但是这一次放置的缓冲区不同，该缓冲区与目标套接字相关联  
4、随后send()调用的返回导致了第四次的上下文切换。DMA 引擎将数据从内核缓冲区传到协议引擎，第四次拷贝独立地、异步发生  
既然这四次拷贝的时间这么麻烦，那为啥还需要中间内核缓冲区呢？  
使用中间内核缓冲区（而不是直接将数据传输到用户缓冲区）看起来可能有点效率低下。但是之所以引入中间内核缓冲区的目的是想提高性能。在读取方面使用中间内核缓冲区，可以允许内核缓冲区在应用程序不需要内核缓冲区内的全部数据时，充当 “预读高速缓存（readahead cache）” 的角色。这在所需数据量小于内核缓冲区大小时极大地提高了性能。在写入方面的中间缓冲区则可以让写入过程异步完成  
不幸的是，如果所需数据量远大于内核缓冲区大小的话，这个方法本身可能成为一个性能瓶颈。数据在被最终传入到应用程序前，在磁盘、内核缓冲区和用户缓冲区中被拷贝了多次  
- **零拷贝**  
emsp;emsp;当使用“零拷贝”后，数据的拷贝从内存拷贝到kafka服务进程那块，又拷贝到socket缓存那块，整个过程耗费的时间比较高，kafka利用了Linux的sendFile技术（NIO），省去了进程切换和一次数据拷贝，让性能变得更好  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka-01-06.jpg)  
<center>零拷贝示意图</center>  
上面的内容可以表示为：  
    - `transferTo()` 方法引发 DMA 引擎将文件内容拷贝到内核缓冲区  
    - 数据未被拷贝到套接字缓冲区。取而代之的是，只有包含关于数据的位置和长度的信息的描述符被追加到了套接字缓冲区。DMA 引擎直接把数据从内核缓冲区传输到协议引擎，从而消除了剩下的最后一次 CPU 拷贝

##### 分批发送
- Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据  
- 批处理能够进行更有效的数据压缩并减少 I/O 延迟  

##### 顺序读写
Kafka **采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费**：  
- **一次磁盘 IO 的耗时主要取决于：寻道时间和盘片旋转时间，提高磁盘 IO 性能最有效的方法就是：减少随机 IO，增加顺序 IO**  
- 磁盘顺序写入速度可以达到`几百兆/s`，而随机写入速度只有`几百KB/s`，相差上千倍。此外，磁盘顺序 IO 访问甚至可以超过内存随机 IO 的性能
- 操作系统每次从磁盘读写数据的时候，需要先寻址，也就是先要找到数据在磁盘上的物理位置，然后再进行数据读写，如果是机械硬盘，寻址就需要较长的时间  
- kafka的设计中，数据其实是存储在磁盘上面，一般来说，会把数据存储在内存上面性能才会好。但是kafka用的是顺序写，追加数据是追加到末尾，磁盘顺序写的性能极高，在磁盘个数一定，转数达到一定的情况下，基本和内存速度一致  
- 随机写的话是在文件的某个位置修改数据，性能会较低

##### 消息压缩

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
    @Value("${kafka.consumer.max-poll-interval}")
    private int kafkaConsumerMaxPollInterval;
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
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, kafkaConsumerMaxPollInterval);
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
##### ConsumerConfig补充

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
    * 最大拉取记录数，对应了消费类中的Record的单次List大小，也即单次poll回来的数据大小
    * 配合max.poll.interval.ms = 300000， 也就说poll是在一直不停的拉去的，如果单次拉取没有数据的时候，在这个时间段超时后会主动返回，如果在这个时间段内达到了设定的条数也会主动返回
    * 默认值为500条
    * Maximum number of records returned in a single call to poll().
    */
   private Integer maxPollRecords;
}
```

##### Consumer心跳
- 关键参数：**session.timeout.ms和heartbeat.interval.ms**  
- **consumer心跳机制**  
consumer定期向coordinator发送心跳请求，以表明自己还在线；如果session.timeout.ms内未发送请求，coordinator认为其不可用，然后触发rebalance  
- 默认值  
`session.timeout.ms`：coordinator感知consumer崩溃所需时间，默认10秒  
`heartbeat.interval.ms`：consumer发送心跳请求间隔，默认3秒  
- **建议**  
`session.timeout.ms`一定要大于`heartbeat.interval.ms`，否则消费者组会一直处于rebalance状态  
`session.timeout.ms`最好几倍于`heartbeat.interval.ms`；这是因为如果因为某一时间段的网络延迟导致coordinator未感知到心跳请求，`session.timeout.ms`和`heartbeat.interval.ms`接近的话，会导致consumer组rebalance过于频繁，影响消费性能  

##### 消费类

```java
/**
 * 注解的释义
 * id：监听器的id：（1）消费者线程命名规则（2）会覆盖消费者工厂的消费组GroupId，可以配合idIsGroup = false来恢复工程中的groupId
 * containerFactory：监听容器工厂，也就是ConcurrentKafkaListenerContainerFactory，一般对应项目中配置的BeanName
 * topics：需要监听的Topic，可监听多个
 * topicPartitions：可配置更加详细的监听信息，必须监听某个Topic中的指定分区，或者从offset为200的偏移量开始监听
 * errorHandler：监听异常处理器，配置BeanName
 * groupId：消费组ID
 * idIsGroup：id是否为GroupId，值为true或false
 * clientIdPrefix：消费者Id前缀
 * beanRef：真实监听容器的BeanName，默认为"__listener"
 */
@KafkaListener(id = "consumer-xxxx",topics = "${xxxxx}",containerFactory = "batchListenerContainerFactory",topicPartitions = {
            @TopicPartition(topic = "topic.quick.batch.partition",partitions = {"1","3"}),
            @TopicPartition(topic = "topic.quick.batch.partition",partitions = {"0","4"},
                    partitionOffsets = @PartitionOffset(partition = "2",initialOffset = "100"))
        })
/**
 * 入参的释义
 * ConsumerRecord：具体消费数据类，包含Headers信息、分区信息、时间戳等，List集合的则是用作批量消费
 * Acknowledgment：用作Ack机制的接口
 * Consumer：消费者类，使用该类我们可以手动提交偏移量、控制消费速率等功能
 */
public void receiveQueue(List<ConsumerRecord<?, ?>> records,Acknowledgment ack, Consumer<K,V> consumer) {
    records.forEach(message->{
        //do something.根据生产者发送的消息体来进行消息获取，比如string时
        JSONObject jsonObject = JSONObject.parseObject((String)message.value());
        //ack的处理
        ack.acknowledge();
        //seek的处理：可以指定从哪个主题的哪个分区的哪个offset开始进行消费
        consumer.seek(new TopicPartition("自定义的topic",record.partition()),record.offset() );
    }
}

//除了从ConsumerRecord类中获取消息头等外，还可以根据注解方式获取消息头及消息体
@KafkaListener(id = "anno", topics = "topic.quick.anno")
/**
 * @Payload：获取的是消息的消息体，也就是发送内容
 * @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY)：获取发送消息的key
 * @Header(KafkaHeaders.RECEIVED_PARTITION_ID)：获取当前消息是从哪个分区中监听到的
 * @Header(KafkaHeaders.RECEIVED_TOPIC)：获取监听的TopicName
 * @Header(KafkaHeaders.RECEIVED_TIMESTAMP)：获取时间戳
 */
public void annoListener(@Payload List<String> data,
                         @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) Integer key,
                         @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                         @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                         @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long ts) {
    
}
```

### 实战-生产者（无情的send） 
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

