---
layout:     post
title:      "就决定是你了，Kafka-03"
subtitle:   "如何保证消息不丢失？如何保证消息顺序？并发消费和批量消费"
date:       2021-04-17
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka.png"
tags:
    - Java
    - 消息队列
    - Kafka
---
> 部分资料来源于网络上各位前辈（cxuan、liudashuang2017、织田下总介等）

### 前言
这一篇主要想记录一下在kafka中如何保证消息的消费顺序以及如何保证消息不丢失。
### 为啥会丢失呢？  
##### Broker的原因   
- kafka批量入盘  
Broker丢失消息是由于Kafka本身的原因造成的，kafka为了得到更高的性能和吞吐量，将数据异步批量的存储在磁盘中。消息的刷盘过程，为了提高性能，减少刷盘次数，kafka采用了批量刷盘的做法。即按照一定的消息量，和时间间隔进行刷盘  
- linux操作系统批量入盘  
将数据存储到linux操作系统种，会先存储到页缓存`Page cache`中，按照时间/`fsync命令`或者其他条件再从页缓存刷入`file`。如果数据在page cache中时系统挂掉，数据会丢失  
- 不完全首领选举  
    - leader副本和follower副本   
    这个问题涉及到kafka的复制机制，kafka每个topic有多个分区，分区存储在磁盘上，kafka可以保证分区的数据是有序的，每个分区可以有多个副本。同时，副本按照是否是首领，可以分为首领副本和跟随者副本（这里对应的就是kafka集群中的leader和follower）  
    - 同步副本和非同步副本  
    除开leader副本外，其他副本按照和leader副本的同步的状态可以分为同步副本和非同步副本    
    **小细节：kafka 如何判断一个follower副本是不是同步副本？**  
    满足两个个条件：1.在过去`replica.lag.time.max`参数毫秒内从首领获取过消息，并且是最新消息。 2.过去6秒内 和 zk直接发送过心跳。   
    - 选举过程  
    如果leader宕机，这时候系统需要选举一个follower来作为首领，kafka优先选择同步副本作为首领，当系统没有同步副本的时候。kafka只有选择非同步副本作为首领，则会丢失一部分数据，（这一部分数据就是非同步副本无法及时从首领副本更新的消息）。kafka如果不选择非同步副本作为首领，则此时kafka集群不可用  

##### Producer的原因
为了提升效率，减少IO，producer在发送数据时可以将多个请求进行合并后发送。被合并的请求咋发送一线缓存在本地buffer中。缓存的方式和前文提到的刷盘类似，producer可以将请求打包成“块”或者按照时间间隔，将buffer中的数据发出  
在正常情况下，客户端的异步调用可以通过callback来处理消息发送失败或者超时的情况，但是：  
- producer被非法的停止了，那么buffer中的数据将丢失，broker将无法收到该部分数据  
- producer客户端内存不够时，如果采取的策略是丢弃消息（另一种策略是block阻塞），消息也会被丢失  
- producer的异步send()过快，导致挂起线程过多，内存不足，程序崩溃，消息丢失
    
##### Consumer的原因  
常见的就是多线程消费时选择了自动提交偏移量  
### 怎么保证消息不丢失
##### kafka服务保证  
- 避免不完全首领选举step1  
kafka 选择非同步副本作为首领副本的行为叫做**不完全首领选举**，顾名思义，非同步副本此时的数据肯定会有丢失的，那么如何控制kafka在leader宕机时，同步副本不可用时不要选择非同步作为首领？通过kafka的另外一个参数`unclean.leader.election`来控制的，如果是true 则会发生不完全首领选举。  
- 避免不完全首领选举step2  
除了谁定开关之外，还需要保证kafka系统中至少有两个同步副本。一个肯定是首领副本，另外一个是从的副本。否则，kafka也会巧妇难为无米之炊，那么怎么来保证呢？  
通过设置kafka的另外一个参数：**最小同步副本数**`min.insync.replicas`。这个参数的意思是 kafka收到生产者消息之后，至少几个同步副本完成同步之后，才给客户端消息确认。只有保证kafka收到生产者的消息且至少有 “最小同步副本数“ 的副本收到消息，才能保证在主宕机时消息不丢失。但是，数量多能保证高可用，效率也就低了。

##### 生产者保证
- 选择合适的ack  
这个简单，通过配置ack的属性来控制。0的话不用说，消息很容易丢失。如果是1，那么代表发送过去，等待首领副本确认消息，认为成功。这里就有点意思了，首领肯定收到了消息，写入了分区文件，但是不一定全部都落盘了。如果是all,那么代表发送过去之后，消息被写入所有同步副本之后才会认为成功。  
注意这里是**所有同步副本**，不是所有副本。具体是多少同步副本，还要取决于上面设置的最小同步副本数和集群当前的同步副本数。选择这种配置会可靠，但是牺牲效率，不过，这个效率可以通过使用异步模式来提高。  
- 使用消息发送回调函数保证消息  
- 配置消息重发，减少因为网络波动的影响  
- **<font color=red>总结，如果对于消息丢失要求十分严格，可以采用ack=all+合适的最小同步副本数、集群当前的同步副本数+异步send+回调函数+消息重发</font>**

##### 消费者保证
- 根据具体的业务来选择消息队列的消费方式，一种是发布订阅模式，一种是队列模式，尤其是group-id要配对  
- 设定好没有偏移量可以提交的时候，系统从哪里开始消费。`auto.offset.reset`的选择  
- 关闭自动提交，确认消费完成后再提交偏移量  
如果开启了自动提交，那么系统会自动进行提交offset。可能会引起，并未消费掉，就提交了offset.引起数据的丢失。与自动提交相关的是自动提交的间隔时间`auto.commit.interval.ms` 默认是5秒钟提交一次。  
但是这种方法可能引起消息的重复消费  
- 通过`ConsumerRebalanceListener`监听接口的实现类+`seek()`特定偏移量处开始处理消息，来保证重平衡后的offset提交准确性  

### 具体消费者该怎么保证？
##### 设置合适的Consumer心跳
`session.timeout.ms`一定要大于`heartbeat.interval.ms`，最好几倍于`heartbeat.interval.ms`，否则消费者组会一直处于rebalance状态  
##### subscribe()和assign()
对于消费者而言，有这两个方法，区别在于  
- **subscribe()方法**  
    - 设定好主题信息后，kafka会帮我们处理分区的分配问题，而你不需要处理分区的分配问题  
    - **一般会使用`subscribe(Collection<String> topics, ConsumerRebalanceListener var2)`**
- assign()方法(一般不建议使用)  
这个方法是采用手动的方式，指定消费者读取哪个主题分区，**注意此时不能再指定消费者组了**，当你需要精确地控制消息处理的负载，也能确定哪个分区有哪些消息时，这种手动的方式会很有用。但这时Kafka也无法提供rebalancing的功能了。而且在使用手动的方式时，你可以不指定消费者组，group.id为空

##### 自动提交（不建议）  
最简单的方式就是让消费者自动提交偏移量。  
- 如果 `enable.auto.commit` 被设置为true，那么默认每过 5s，消费者会自动把从 `poll()` 方法轮询到的最大偏移量提交上去  
- 提交时间间隔由 `auto.commit.interval.ms` 控制，默认是 5s。与消费者里的其他东西一样，自动提交也是在轮询中进行的。消费者在每次轮询中会检查是否提交该偏移量了，如果是，那么就会提交从上一次轮询中返回的偏移量

##### 手动同步提交commitSync()
- `commitSync()` 将会提交由 `poll()` 返回的最新偏移量，如果处理完所有记录后要确保调用了该方法，否则还是会有丢失消息的风险  
- 只要没有发生不可恢复的错误，`commitSync()`方法会阻塞，会一直尝试直至提交成功，如果失败，也只能记录异常日志  
- 同步提交有重复消费的风险，如果发生了重平衡，从最近一批消息到发生在重平衡之间的所有消息都将被重复处理  
    
```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

/**
 *手动同步提交commitSync()
 **/
public class CommitSyncConsumer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.42.111:9092");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "CommitSync");
        //取消自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        try {
            consumer.subscribe(Collections.singletonList("自定topic"));
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("主题：" + record.topic() + ";分区：" + record.partition() +
                            ";偏移量：" + record.offset() +";key:" + record.key() + ";value:" + record.value());
                    //do your work
                }
                //同步提交（这个方法会阻塞）
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
}
```    

##### 手动异步提交commitAsync()  
- **异步提交不会进行重试**  
- 异步提交支持回调`OffsetCommitCallback()`，在 `broker` 作出响应时会执行回调。回调经常被用于记录提交错误或生成度量指标。如果要用它来进行重试，则一定要注意提交的顺序（可使用一个单调递增的序列号维护异步提交顺序）

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Map;
import java.util.Properties;

/**
 *手动异步提交commitAsync()
 **/
public class CommitAsyncConsumer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.42.111:9092");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "CommitAsync");
        //取消自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        try {
            consumer.subscribe(Collections.singletonList("自定topic"));
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("主题：" + record.topic() + ";分区：" + record.partition() +
                            ";偏移量：" + record.offset() +";key:" + record.key() + ";value:" + record.value());
                    //do your work
                }
                //允许执行回调，对提交失败的消息进行处理
                consumer.commitAsync(new OffsetCommitCallback() {
                    @Override
                    public void onComplete(Map<TopicPartition, OffsetAndMetadata> map, Exception e) {
                        if (null != e) {
                            //do somthing
                        }
                    }
                });
            }
        } finally {
            consumer.close();
        }
    }
}
```
##### 同步和异步组合提交  
一般情况下，针对偶尔出现的提交失败，不进行重试不会有太大的问题，因为如果提交失败是因为临时问题导致的，那么后续的提交总会有成功的。但是如果在关闭消费者或再均衡前的最后一次提交，就要确保提交成功  
**因此，在消费者关闭之前一般会组合使用commitAsync和commitSync提交偏移量**

```java
package org.example.commit;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

/**
 *同步和异步组合
 **/
public class SyncAndAsync {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.42.111:9092");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "SyncAndAsync");
        //取消自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);

        try {
            consumer.subscribe(Collections.singletonList("自定topic"));
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("主题：" + record.topic() + ";分区：" + record.partition() +
                            ";偏移量：" + record.offset() +";key:" + record.key() + ";value:" + record.value());
                    //do your work
                }
                //异步提交
                consumer.commitAsync(new OffsetCommitCallback() {
                    @Override
                    public void onComplete(Map<TopicPartition, OffsetAndMetadata> map, Exception e) {
                        if (null != e) {
                            //do somthing
                        }
                    }
                });
            }
        } catch (Exception e) {
            //do something
        } finally {
            try {
                //同步提交
                consumer.commitSync();
            } finally {
                consumer.close();
            }
        }
    }
}
```

##### 提交特定的偏移量  
一般提交偏移量的频率和处理消息批次的频率是一样的。如果`poll()` 方法返回一大批数据，为了避免再均衡引发的重复处理整批消息，消费者 API 允许调用 commitSync() 和 commitAsync() 方法时传入希望提交的分区和偏移量的 map。不过因为消费者可能不只读取一个分区，你需要跟踪所有分区的偏移量，所以特定偏移量的提交会使得代码更加复杂  

```java
package org.example.commit;

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
 *特定提交
 **/
public class CommitSpecial {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.42.111:9092");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "CommitSpecial");
        //取消自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        Map<TopicPartition, OffsetAndMetadata> currOffsets = new HashMap<>();
        int count = 0;
        try {
            consumer.subscribe(Collections.singletonList("自定topic"));
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println("主题：" + record.topic() + ";分区：" + record.partition() +
                            ";偏移量：" + record.offset() +";key:" + record.key() + ";value:" + record.value());
                    //do your work
                    currOffsets.put(new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1, "no meta"));
                    if (count % 11 == 0) {
                        consumer.commitAsync(currOffsets,null);
                    }
                    count++;
                }
            }
        } finally {
            consumer.commitSync();
            consumer.close();
        }
    }
}
```

##### 终极大法ConsumerRebalanceListener接口
说白了，问题的关键还是在于怎么处理好发生`rebalance`时的偏移量提交问题。kafka官方提供了一个`ConsumerRebalanceListener`接口，这个有两个需要实现的方法：  
- `public void onPartitionsRevoked ( Collection<TopicPartition> partitions )`   
该方法会在消费者停止读取消息之后和再均衡开始之前被调用。如果在这里提交偏移量，下一个接管分区的消费者就会知道从哪里开始读取消息  
- `public void onPartionsAssigned ( Collection<TopicPartition> partitions )`   
该方法会在重新分配分区之后和消费者开始读取消息之前被调用，所以也可以通过记录每次的提交量，然后在开始消费前来指定从哪一个提交量开始进行消费  
- 使用方法  
在调用 ` KafkaConsumer#subscribe()` 方法时传入一个 `ConsumerRebalanceListener` 实例

```java
/**
 * rebalance触发的处理器
 */
public class HandleRebalance implements ConsumerRebalanceListener {

    // rebalance之前触发最后一次提交
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        //发生Rebalance时,在这里提交偏移量：策略就是先异步+回调，然后同步作为保底
        try {
            consumer.commitAsync(new OffsetCommitCallback() {
            @Override
            public void onComplete(Map<TopicPartition, OffsetAndMetadata> map, Exception e) {
                if (e != null) {
                    //do something
                }
            }
        });
        } finally {
            //同步提交
            consumer.commitSync();
        }
    }

    // rebalance后调用,consumer抓取数据之前触发
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        //指定从哪个主题的哪个分区的哪个offset开始进行消费
        consumer.seek(new TopicPartition("自定义的topic",record.partition()),record.offset() );
    }
}
```

### 如何保证消息顺序？
我们都知道kafka里面，单个分区的消息是有序的（因为内部有offset，当然排除重复消费和消息丢失的异常情况），但是针对多个分区的时候，其实是无序的，这个也叫做全局无序。所以，一般我们说保证kafka的消息有序的话，大多数是指当有多个分区的时候，怎么保证消息有序？
##### 问题
比如说我们建了一个 topic，有三个 partition。生产者在写的时候，其实可以指定一个 key，比如说我们指定了某个订单 id 作为 key，那么这个订单相关的数据，一定会被分发到同一个 partition 中去，而且这个 partition 中的数据一定是有顺序的  
消费者从 partition 中取出来数据的时候，也一定是有顺序的。到这里，顺序还是 ok 的，没有错乱。接着，我们在消费者里可能会搞多个线程来并发处理消息。因为如果消费者是单线程消费处理，而处理比较耗时的话，比如处理一条消息耗时几十 ms，那么 1 秒钟只能处理几十条消息，这吞吐量太低了。而多个线程并发跑的话，顺序可能就乱掉了  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-kafka-03-01.jpg)  
<center>问题场景</center>   

##### 解决方案
- 一个 topic，一个 partition，一个 consumer，内部单线程消费，单线程吞吐量太低，一般不会用这个  
- 写 N 个内存 queue，具有相同 key 的数据都到同一个内存 queue；然后对于 N 个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-kafka-03-02.jpg)  
<center>queue保证消息顺序</center>   

### 并发消费和批量消费
之所以提到并发消费和批量消费，是因为在回顾之前的kafka内容的时候，发现在消费端有两个配置项：`factory.setConcurrency(X)`和`factory.setBatchListener(true)`
##### factory.setConcurrency(X)
顾名思义，并发消费。`factory.setConcurrency(X)`方法也等价于在`application.properties`中添加`spring.kafka.listener.concurrency=X`，然后使用`@KafkaListener`并发消费  
- spring的线程封闭策略  
这里有一个很有意思的事情，在spring中实现消息的并发消费采用的是**线程封闭**的策略，具体实现是在创建监听器容器时，会根据配置的`concurrency`来创建多个`KafkaMessageListenerContainer`，在该类中又有内部类ListenerConsumer，在该内部类中封闭创建了consumer对象。以此来实现主题消息的并发消费  
- 特点  
注意，以这种方式进行并发消费时，**实际的并发度受到了主题分区数的限制，当消费线程数大于分区数时，会使多出来的消费者线程一直处于空闲状态。**对此，spring在创建`KafkaMessageListenerContainer`前，对用户配置的`concurrency`值进行了校验，当该值超出主题分区数时，将值设置为实际的分区数  
同时，以线程封闭的方式实现并发消费，每个消费者线程都需要保持一个TCP连接，如果分区数很大，则会带来很大的系统开销。但是，以该方式实现并发消费，可以保证每个分区消息的顺序消费

##### factory.setBatchListener(true)
批量消费往往和`ConsumerConfig.MAX_POLL_RECORDS_CONFIG`的属性配置是分不开的，**一个设启用批量消费，一个设置批量消费每次最多消费多少条消息记录**  
- **ConsumerConfig.MAX_POLL_RECORDS_CONFIG**  
重点说明一下，设置的`MAX_POLL_RECORDS_CONFIG`，并不是说如果没有达到设定的消息条数就一直等待。官方的解释是：  
"The maximum number of records returned in a single call to poll()."  
也就是`MAX_POLL_RECORDS_CONFIG`表示的是一次poll最多返回的记录数，因为每间隔`max.poll.interval.ms`消费者也会自动就调用一次poll。每次poll最多返回`MAX_POLL_RECORDS_CONFIG`条记录  
- factory.setBatchListener(true)  
启用批量消费  
- 对应的其实就是批量的`ConsumerRecord`

##### 单分区的并发消费
通过上面的分析可以发现，单个分区的时候，`factory.setConcurrency(X)`已经没有什么意义了，因为再多的消费者也会是空闲的。这个时候怎么办呢？可以采取**将消息拉取动作和处理动作分开**  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-kafka-03-03.jpg)  
<center>单个分区的并发消费</center>   
以上图示，将主题对应的消费者组进行池化，每个`group`对应一个`consumer`线程池，池中线程数为主题的分区数。  
每个消费者就是个线程，提交到`consumer`线程池后，不断从服务器拉取消息，同时在消费者线程中，又有一个用来实际处理消息的`MessageHandler`线程池，在获取的消息后，根据每批次的消息创建MessagedoHandle线程，提交到MessagedoHandle线程池进行消息的实际处理