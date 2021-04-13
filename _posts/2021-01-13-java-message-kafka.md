---
layout:     post
title:      "就决定是你了，Kafka-02"
subtitle:   "生产者的深入、分区策略、Kafka压缩、消费者的深入、消息不丢失"
date:       2021-01-13
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka.png"
tags:
    - Java
    - 消息队列
---
> 部分资料来源于网络上各位前辈（cxuan、liudashuang2017、织田下总介等）

### 前言
前面基本上算是对kafka的一些基本情况进行了总结，这里对一些深入一点的东西进行记录。
### 生产者的那些事儿
##### ProducerRecord的流转  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka05.jpg)  
<center>生产者发送数据大致流程</center>  
`ProducerRecord` 是 Kafka 中的一个核心类，可以理解为消息的载体，包含：`key/value 键值对`，`目标topic`、`分区号Partition Number`以及`时间戳`（可选）

- **封装ProducerRecord并序列化数据**：  
    - ProducerRecord 还有关联的时间戳，如果用户没有提供时间戳，那么生产者将会在记录中使用当前的时间作为时间戳。Kafka 最终使用的时间戳取决于 topic 主题配置的时间戳类型  
        1. 如果将主题配置为使用`CreateTime`，则生产者记录中的时间戳将由 broker 使用   
        2. 如果将主题配置为使用`LogAppendTime`，则生产者记录中的时间戳在将消息添加到其日志中时，将由 broker 重写   
    - 序列化时要**将键值对对象由序列化器转换为字节数组**，这样它们才能够在网络上传输  
- **分区器（Partioner）确认目标分区**：然后，消息到达了分区器准备分发  
    - 如果发送过程中指定了有效的分区号，那么在发送记录时将使用该分区  
    - 如果发送过程中未指定分区，则将使用`key 的 hash 函数`映射指定一个分区  
    - 如果发送的过程中既没有分区号也没有key，则将以循环的方式分配到所有分区。选好分区后，生产者就知道向哪个主题和分区发送数据了  
- **进入记录批次**：以上都完成后，这条消息被存放在一个记录批次里，这个批次里的所有消息会被发送到相同的主题和分区上。由一个独立的线程负责把它们发到 Kafka Broker 上。**也就是说，消息是先被写入分区中的缓冲区中，然后分批次发送给 Kafka Broker**  
- **接下来就是副本同步机制**  
注：  
- **Kafka Broker 在收到消息时会返回一个响应，如果写入成功，会返回一个 `RecordMetaData` 对象，它包含了主题和分区信息，以及记录在分区里的偏移量，上面两种的时间戳类型也会返回给用户**。具体来说：发送成功后，send() 方法会返回一个 `Future(java.util.concurrent)` 对象，Future 对象的类型是 RecordMetadata类型。在消息发送之前，生产者还可能发生其他的异常。这些异常有可能是 `SerializationException`(序列化失败)，`BufferedExhaustedException` 或 `TimeoutException`(说明缓冲区已满)，又或是 `InterruptedException`(说明发送线程被中断)    
- 如果写入失败，会返回一个错误。生产者在收到错误之后会尝试重新发送消息，几次之后如果还是失败的话，就返回错误消息。  

##### 绝情：Producer#send()
这是我之前最为常用的模式，现在才知道这其实是发送之后便不再理会发送结果，算是比较绝情了，什么问题都没管。官方叫这种为：Fire and Fogret，挺形象的

**kafka的Producer是线程安全的，用户可以非常非常放心的在多线程中使用**  

根据上一篇中的实战代码，kafka发送消息的时候，主要还是靠Producer的send方法，在这之前需要对一些基础属性进行设置：  
- **bootstrap.servers**  
该属性指定`broker`的地址清单，地址的格式为 `host:port`，比如"xx.xx.xx.xx:9090"。**清单里其实不需要包含所有的 broker 地址，生产者会从给定的 broker 里查找到其他的 broker 信息。不过建议至少要提供两个 broker 信息，一旦其中一个宕机，生产者仍然能够连接到集群上**  
- **key.serializer**  
broker 需要接收到序列化之后的 key/value值，所以生产者发送的消息需要经过序列化之后才传递给 Kafka Broker。生产者需要知道采用何种方式把 Java 对象转换为字节数组。`key.serializer` 必须被设置为一个实现了`org.apache.kafka.common.serialization.Serializer` 接口的类，生产者会使用这个类把键对象序列化为字节数组。  
java代码：`props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class);props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);`  
Serializer 是一个接口，它表示类将会采用何种方式序列化，它的作用是把对象转换为字节，实现了 Serializer 接口的类主要有 **`ByteArraySerializer`（Kafka 默认使用的序列化器）**、`StringSerializer`、`IntegerSerializer` 等，要注意的一点：**key.serializer 是必须要设置的，即使发送消息的时候没有指定key只发送value**。  
- **acks**  
    - acks 参数指定了要有多少个分区副本接收消息，生产者才认为消息是写入成功的。**此参数对消息丢失的影响较大**  
    - `acks = 0`：就表示生产者也不知道自己产生的消息是否被服务器接收了，它才知道它写成功了。如果发送的途中产生了错误，生产者也不知道，它也比较懵逼，因为没有返回任何消息。这就类似于 UDP 的运输层协议，只管发，服务器接受不接受它也不关心。  
    - `acks = 1`：只要集群的 Leader 接收到消息，就会给生产者返回一条消息，告诉它写入成功。如果发送途中造成了网络异常或者 Leader 还没选举出来等其他情况导致消息写入失败，生产者会受到错误消息，这时候生产者往往会再次重发数据。因为消息的发送也分为 同步 和 异步，Kafka 为了保证消息的高效传输会决定是同步发送还是异步发送。如果让客户端等待服务器的响应（比如通过调用 Future 中的 get() 方法可以同步等待），显然会增加延迟，所以建议客户端使用回调，那就会解决这个问题。  
    - `acks = all`：这种情况下是只有当所有参与复制的节点都收到消息时，生产者才会接收到一个来自服务器的消息。不过，它的延迟比 acks =1 时更高，因为此时往往需要等待不只一个服务器节点接收消息。  
- **retries**  
    - `代表生产者可以重发的消息次数`，生产者从服务器收到的错误有可能是临时性的错误（比如分区找不到首领），如果达到参数设置的这个次数，生产者会放弃重试并返回错误。  
    - 默认情况下，生产者在每次重试之间等待 **100ms**，这个等待参数可以通过 `retry.backoff.ms` 进行修改  
- **batch.size**  
`该参数指定了一个批次可以使用的内存大小，按照字节数计算`，从上面的介绍，当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。当批次被填满，批次里的所有消息会被发送出去。**不过生产者不一定都会等到批次被填满才发送，任意条数的消息都可能被发送**  
- client.id  
此参数可以是任意的字符串，服务器会用它来识别消息的来源，一般配置在日志里  
- **max.in.flight.requests.per.connection**  
此参数指定了生产者在收到服务器响应之前可以发送多少消息，它的值越高，就会占用越多的内存，不过也会提高吞吐量。把它设为1可以保证消息是按照发送的顺序写入服务器  
- **timeout.ms、request.timeout.ms 和 metadata.fetch.timeout.ms**
    - `request.timeout.ms` 指定了生产者在发送数据时等待服务器返回的响应时间  
    - `metadata.fetch.timeout.ms` 指定了生产者在获取元数据（比如目标分区的首领是谁）时等待服务器返回响应的时间。如果等待时间超时，生产者要么重试发送数据，要么返回一个错误  
    - `timeout.ms` 指定了 broker 等待同步副本返回消息确认的时间，往往配合 `asks` 的配置使用。如果在指定时间内没有收到同步副本的确认，那么 broker 就会返回一个错误  
- max.block.ms  
    - 此参数指定了在调用 send() 方法或使用 partitionFor() 方法时从broker获取`RecordMetaData` 对象时，生产者的阻塞时间  
    - 当生产者的发送缓冲区已满，或者没有可用的元数据时，这些方法就会阻塞。在阻塞时间达到 max.block.ms 时，生产者会抛出超时异常  
- max.request.size  
该参数用于控制生产者发送的请求大小。它可以指能发送的单个消息的最大值，也可以指单个请求里所有消息的总大小  
- receive.buffer.bytes 和 send.buffer.bytes  
Kafka 是基于 TCP 实现的，为了保证可靠的消息传输，这两个参数分别指定了 TCP Socket 接收和发送数据包的缓冲区的大小。如果它们被设置为 -1，就使用操作系统的默认值。如果生产者或消费者与 broker 处于不同的数据中心，那么可以适当增大这些值  

##### 同步：Producer#send().get()
首先调用 send() 方法，然后再调用 get() 方法等待 Kafka 响应。如果服务器返回错误，get() 方法会抛出异常，如果没有发生错误，会得到 RecordMetadata 对象，可以用它来查看消息记录。这里就涉及到生产者在进行消息生产的时候容易出现的两种错误：  
- 可重试异常（继承RetriableException）
    - `LeaderNotAvailableException`：分区的Leader副本不可用，这可能是换届选举导致的瞬时的异常，重试几次就可以恢复  
    - `NotControllerException`：Controller主要是用来选择分区副本和每一个分区leader的副本信息，主要负责统一管理分区信息等，也可能是选举所致  
    - `NetWorkerException`：瞬时网络故障异常所致，可以通过再次建立连接来解决    
    - KafkaProducer 被配置为自动重试时，如果多次重试后仍无法解决问题，则会抛出重试异常  
- 不可重试异常  
    - 这类异常KafkaProducer 不会进行重试，直接抛出异常  
    - `SerializationException`：序列化失败异常  
    - `RecordToolLargeException`：消息尺寸过大导致  
到了这里，我不得不有一个疑问：之前在项目里使用的时候都是同步在发送消息，同一时间只能有一个消息在发送，这会造成许多消息无法直接发送，造成消息滞后，无法发挥效益最大化。那有没有异步的方式呢？答案自然是有的
##### 异步：带callback的Producer#send()
这里即可做到异步发送消息，还能利用回调对异常情况进行处理：  
首先实现回调需要定义一个实现了`org.apache.kafka.clients.producer.Callback`的类，这个接口只有一个 onCompletion方法。**其中RecordMetadata 和 Exception 不可能同时为空，消息发送成功时，Exception为null，消息发送失败时，metadata为空**  
然后，结合上面分析的两种类别的异常，此时可以进行分别处理：  
```java
ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>("CustomerCountry", "Huston", "America");
producer.send(producerRecord,new DemoProducerCallBack());
//回调函数定义
class DemoProducerCallBack implements Callback {
  public void onCompletion(RecordMetadata metadata, Exception exception) {
    //其中RecordMetadata 和 Exception 不可能同时为空，消息发送成功时，Exception为null，消息发送失败时，metadata为空
    if(null==e){
        //正常处理逻辑
        System.out.println("The offset of the record we just sent is: " + metadata.offset()); 
    }else{
            
          if(e instanceof RetriableException) {
             //处理可重试异常
          } else {
             //处理不可重试异常
          }
    }
  }
}
```

##### 基于事务的Producer#send()
下面这个例子中，虽然是循环发送，但是其实都是隶属于同一个事务  
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("transactional.id", "my-transactional-id");
Producer<String, String> producer = new KafkaProducer<>(props, new StringSerializer(),new StringSerializer());
producer.initTransactions();

try {
    producer.beginTransaction();
    for (int i = 0; i < 100; i++)
        producer.send(new ProducerRecord<>("my-topic", Integer.toString(i), Integer.toString(i)));
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
    // We can't recover from these exceptions, so our only option is to close the producer and exit.
    producer.close();
} catch (KafkaException e) {
    // For all other exceptions, just abort the transaction and try again.
    producer.abortTransaction();
}
producer.close();
```
##### 绅士关闭：Producer#close()
有开启自然有关闭：  
- producer.close()：优先把消息处理完毕，优雅退出  
- producer.close(timeout): 超时时，强制关闭
### 生产者分区策略
Kafka 对于数据的读写是以分区为粒度的，分区可以分布在多个主机（Broker）中，这样每个节点能够实现独立的数据写入和读取，并且能够通过增加新的节点来增加 Kafka 集群的吞吐量，通过分区部署在多个 Broker 来实现负载均衡的效果  
**这里有一点需要注意：比如已有X个分区了，而且历史消息也已经储存完毕，此时新增分区，只有新消息才会到新的分区，历史消息还是保持不动了**，这里就涉及到一个问题，如果历史有大量的历史消息需要被处理的时候，其实调大分区数量是无效的，同时增加服务节点也不行，因为kafka允许同组的多个分区被一个consumer消费，但是不允许一个分区被不同组的多个consumer消费，即**一个partition在同一个时刻只有一个consumer instance在消费**
##### 顺序轮询
Kafka 的分区策略指的就是将生产者发送到哪个分区的算法。Kafka提供了默认的分区策略(**顺序轮巡**)，同时也支持自定义分区策略，主要是靠显示配置生产者端的参数 `Partitioner#partition()`，位于`org.apache.kafka.clients.producer`下  
```java
public interface Partitioner extends Configurable, Closeable {

  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);

  public void close();

  default public void onNewBatch(String topic, Cluster cluster, int prevPartition) {}
}
```

##### 随机轮询  
**如果追求数据的均匀分布，还是使用轮询策略比较好**  

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```
##### 按消息键保序
这个策略也叫做 key-ordering 策略，Kafka 中每条消息都会有自己的key，一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略  

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

### Kafka 压缩
为什么启用压缩？说白了就是消息太大，需要变小一点 来使消息发的更快一些。其实这一点有点类似于Feign优化中的Gzip压缩，是一种经典的用 CPU 时间去换磁盘空间或者 I/O 传输量的思想，希望以 CPU 开销带来更少的磁盘占用或更少的网络 I/O 传输。
##### 什么是Kafka 压缩？
Kafka 的消息分为两层：消息集合 和 消息。一个消息集合中包含若干条日志项，而日志项才是真正封装消息的地方。Kafka 底层的消息日志由一系列消息集合日志项组成。Kafka 通常不会直接操作具体的一条条消息，它总是在消息集合这个层面上进行写入操作  
在 Kafka 中，压缩会发生在两个地方：Kafka Producer 和 Kafka Consumer

##### 怎么玩？
- Producer先压缩：`properties.put("compression.type", "gzip");`（使用GZIP的算法进行压缩）  
- Consumer再解压缩：因为采用的何种压缩算法是随着 key、value 一起发送过去的，所以消费者知道采用何种压缩算法   
### 消费者的那些事儿
##### 水平扩展提升消费能力
因为Kafka 消费者从属于消费者群组。一个群组中的消费者订阅的都是相同的主题，每个消费者接收主题一部分分区的消息，**一个partition在同一个时刻只有一个consumer instance在消费**，而且一般来说，消费者的个数都会建议和生产者的分区个数相同，这样既不会有消费不过来的消费者，也不会有空闲的消费者。  
如果应用需要读取全量消息，那么请为该应用设置一个消费组；如果该应用消费能力不足，那么可以考虑在这个消费组里增加消费者  
##### 消费者组
消费组不用多说，有个小细节需要注意：  
`group.id` 这个属性不是必须的，它指定了 KafkaConsumer 是属于哪个消费者群组。创建不属于任何一个群组的消费者也是可以的   
##### 消费方式  
- **点对点**  
一个消费者群组消费一个主题中的消息，这种消费模式又称为点对点的消费方式，点对点的消费方式又被称为消息队列  
- **发布-订阅**  
一个主题中的消息被多个消费者群组共同消费，这种消费模式又称为发布-订阅模式  
##### 分区重平衡  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka06.jpg)  
<center>rebalance的示意图</center>  
最初是一个消费者订阅一个主题并消费其全部分区的消息，后来有一个消费者加入群组，随后又有更多的消费者加入群组，而新加入的消费者实例分摊了最初消费者的部分消息，这种把分区的所有权通过一个消费者转到其他消费者的行为称为重平衡，英文名也叫做 Rebalance  
重平衡非常重要，它为消费者群组带来了高可用性 和 伸缩性，我们可以放心的添加消费者或移除消费者，不过在正常情况下我们并不希望发生这样的行为。在重平衡期间，消费者无法读取消息，造成整个消费者组在重平衡的期间都不可用。另外，当分区被重新分配给另一个消费者时，消息当前的读取状态会丢失，它有可能还需要去刷新缓存，在它重新恢复状态之前会拖慢应用程序。  
##### 线程安全性
在同一个群组中，我们无法让一个线程运行多个消费者，也无法让多个线程安全的共享一个消费者。按照规则，一个消费者使用一个线程，如果一个消费者群组中多个消费者都想要运行的话，那么必须让每个消费者在自己的线程中运行，可以使用线程池启动多个消费者进行进行处理  
##### Offset、偏移量
- 生产者Offset  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka08.jpg)  
<center>生产者的offset示意图</center>  
不管是多少个生产者，不管规定了这些生产者会写入哪一个分区。但只要生产者写入消息的时候，一定是每一个分区都有一个offset，这个offset就是生产者的offset，同时也是这个分区的最新最大的offset  
- 消费者Offset  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka09.jpg)  
<center>消费者的offset示意图</center>  
这是某一个分区的offset情况，我们已经知道生产者写入的offset是最新最大的值也就是12，而当Consumer A进行消费时，他从0开始消费，一直消费到了9，它的offset就记录在了9，再比如Consumer B就纪录在了11。等下一次他们再来消费时，他们可以选择接着上一次的位置消费，当然也可以选择从头消费，或者跳到最近的记录并从“现在”开始消费。  
- **总的说明**

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka10.jpg)  

1. 每个分区是由多个Segment组成，当Kafka要写数据到一个partition时，它会写入到状态为active的segment中。如果该segment被写满，则一个新的segment将会被新建，然后变成新的“active” segment  
2. 偏移量：分区中的每一条消息都会被分配的一个连续的id值，该值用于唯一标识分区中的每一条消息  
3. 每个segment中则保存了真实的消息数据。每个Segment对应于一个索引文件与一个日志文件。segment文件的生命周期是由Kafka Server的配置参数所决定的。比如说，server.properties文件中的参数项log.retention.hours=168就表示7天后删除老的消息文件  
4. 每个segment有以下3种数据文件：
    - **00000000000000000000.index**：基于偏移量的索引文件，存放着消息的offset和其对应的物理位置，是稀松索引,稀松索引可以加快速度，因为 index 不是为每条消息都存一条索引信息，而是每隔几条数据才存一条 index 信息，这样 index 文件其实很小。kafka在写入日志文件的时候，同时会写索引文件（.index和.timeindex）。默认情况下，有个参数`log.index.interval.bytes`限定了在日志文件写入多少数据，就要在索引文件写一条索引，默认是4KB，写4kb的数据然后在索引里写一条索引  
    - **00000000000000000000.log**：它是segment文件的数据文件，用于存储实际的消息。该文件是二进制格式的。log文件是存储在 `ConcurrentSkipListMap` 里的，是一个map结构，**key是文件名（offset）**，value是内容，这样在查找指定偏移量的消息时，用二分查找法就能快速定位到消息所在的数据文件和索引文件  
    - **00000000000000000000.timeindex**：基于时间戳的索引文件  
    - 命名规则：partition全局的第一个segment从0开始，**后续每个segment文件名为上一个segment文件最后一条消息的offset值**。没有数字则用0填充  
5. 新数据加在文件的末尾(调用内部方法)，不论文件多大，该操作的时间复杂度都是O(1)，但是在查找某个 offset 的时候，是顺序查找，如果文件很大的话，查找的效率就会很低  
6. **通过二分查找文件列表，快速定位到具体的segment文件，再以对应的.index作为索引在.log中查找具体的消息**  
消费者在每次调用poll() 方法进行定时轮询的时候，会返回由生产者写入 Kafka 但是还没有被消费者消费的记录，因此可以追踪到哪些记录是被群组里的哪个消费者读取的。消费者可以使用 Kafka 来追踪消息在分区中的位置（偏移量）  
消费者会向一个叫做 `_consumer_offset` 的特殊主题中发送消息，这个主题会保存每次所发送消息中的分区偏移量，**这个主题的主要作用就是消费者触发重平衡后记录偏移使用的**，消费者每次向这个主题发送消息，正常情况下不触发重平衡，这个主题是不起作用的，当触发重平衡后，消费者停止工作，每个消费者可能会分到对应的分区，这个主题就是让消费者能够继续处理消息所设置的  
- **消息丢失和重复消费**  
    - `重复消费：提交偏移量<处理的最后偏移`    
        ![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/kafka07.jpg)   
        <center>重复消费示意图</center>  
    - `消息丢失：提交偏移量>处理的最后偏移`    
##### 提交方式  
- 自动提交  
最简单的方式就是让消费者自动提交偏移量。如果 `enable.auto.commit` 被设置为true，那么每过 5s，消费者会自动把从 poll() 方法轮询到的最大偏移量提交上去。提交时间间隔由 `auto.commit.interval.ms` 控制，默认是 5s。与消费者里的其他东西一样，自动提交也是在轮询中进行的。消费者在每次轮询中会检查是否提交该偏移量了，如果是，那么就会提交从上一次轮询中返回的偏移量。
- 手动同步提交  
可以让应用程序决定何时提交偏移量。使用 `commitSync()` 提交偏移量。这个 API 会提交由 poll() 方法返回的最新偏移量，提交成功后马上返回，如果提交失败就抛出异常。commitSync() 将会提交由 poll() 返回的最新偏移量，如果处理完所有记录后要确保调用了 commitSync()，否则还是会有丢失消息的风险，如果发生了重平衡，从最近一批消息到发生在重平衡之间的所有消息都将被重复处理。  
- 手动异步提交  
异步提交 `commitAsync()` 与同步提交 `commitSync()` 最大的区别在于异步提交不会进行重试，同步提交会一直进行重试。  
**commitAsync() 也支持回调**，在 broker 作出响应时会执行回调。回调经常被用于记录提交错误或生成度量指标。如果要用它来进行重试，则一定要注意提交的顺序（可使用一个单调递增的序列号维护异步提交顺序）  
- 同步和异步组合提交  
一般情况下，针对偶尔出现的提交失败，不进行重试不会有太大的问题，因为如果提交失败是因为临时问题导致的，那么后续的提交总会有成功的。但是如果在关闭消费者或再均衡前的最后一次提交，就要确保提交成功。**因此，在消费者关闭之前一般会组合使用commitAsync和commitSync提交偏移量。**  
- 提交特定的偏移量  
一般提交偏移量的频率和处理消息批次的频率是一样的。如果 poll() 方法返回一大批数据，为了避免再均衡引发的重复处理整批消息，消费者 API 允许调用 commitSync() 和 commitAsync() 方法时传入希望提交的分区和偏移量的 map。不过因为消费者可能不只读取一个分区，你需要跟踪所有分区的偏移量，所以特定偏移量的提交会使得代码更加复杂  

说白了，问题的关键还是在于怎么处理好再平衡时的偏移量提交问题：  
因此在为消费者分配新分区或移除分区时，可以通过消费者 API 执行一些代码：在调用 ` KafkaConsumer#subscribe()` 方法（这个方法强制要求你为消费者设置一个消费者组（也就是`group.id`参数），随后就不需要处理分区的分配问题了。）时传入一个 `ConsumerRebalanceListener` 实例。`ConsumerRebalanceListener` 有两个需要实现的方法：  
1. `public void onPartitionsRevoked ( Collection<TopicPartition> partitions )` 方法会在消费者停止读取消息之后和再均衡开始之前被调用。如果在这里提交偏移量，下一个接管分区的消费者就会知道从哪里开始读取消息。  
2. `public void onPartionsAssigned ( Collection<TopicPartition> partitions )` 方法会在重新分配分区之后和消费者开始读取消息之前被调用。  

### 消息怎么保证不丢失  
##### 为啥会丢失呢？  
`kafka的服务层面原因`  
- leader副本和follower副本   
这个问题涉及到kafka的复制机制，kafka每个topic有多个分区，分区存储在磁盘上，kafka可以保证分区的数据是有序的，每个分区可以有多个副本。同时，副本按照是否是首领，可以分为首领副本和跟随者副本（这里对应的就是kafka集群中的leader和follower）  
- 同步副本和非同步副本  
除开leader副本外，其他副本按照和leader副本的同步的状态可以分为同步副本和非同步副本    
- 选举过程  
    - 所有的消息都是发送给leader的，消息消费也是从leader获取的。leader副本总是第一时间收到消息，或者消费消息。所以leader副本同时也一定是同步副本。其他follower都是和leader保持通信，同步leader的消息。当leader不可用时，会选举一个follower会变成leader  
    - 对于一个主一个从的两个kafka，做的集群来说。一个是leader副本，一个是follower副本。当follower副本一直能与leader副本保持同步的时候 follower副本是同步副本，当follower与leader无法保持同步的时候 follower副本则变成非同步副本。如果leader宕机，这时候系统需要选举一个follower来作为首领，kafka优先选择同步副本作为首领，当系统没有同步副本的时候。kafka如果选择非同步副本作为首领，则会丢失一部分数据，（这一部分数据就是非同步副本无法及时从首领副本更新的消息）。kafka如果不选择非同步副本作为首领，则此时kafka集群不可用。  

`消费者层面原因`  
常见的就是多线程消费时选择了自动提交偏移量  
##### kafka服务保证
- `避免不完全首领选举step1`  
kafka 选择非同步副本作为首领副本的行为叫做**不完全首领选举**，顾名思义，非同步副本此时的数据肯定会有丢失的，那么如何控制kafka在leader宕机时，同步副本不可用时不要选择非同步作为首领？通过kafka的另外一个参数`unclean.leader.election`来控制的，如果是true 则会发生不完全首领选举。  
- `避免不完全首领选举step2`  
除了谁定开关之外，还需要保证kafka系统中至少有两个同步副本。一个肯定是首领副本，另外一个是从的副本。否则，kafka也会巧妇难为无米之炊，那么怎么来保证呢？  
通过设置kafka的另外一个参数：**最小同步副本数**`min,insync.replicas`。这个参数的意思是 kafka收到生产者消息之后，至少几个同步副本完成同步之后，才给客户端消息确认。只有保证kafka收到生产者的消息且至少有 “最小同步副本数“ 的副本收到消息，才能保证在主宕机时消息不丢失。但是，数量多能保证高可用，效率也就低了。

**小细节：kafka 如何判断一个follower副本是不是同步副本？**  
满足两个个条件：1.在过去10秒内从首领获取过消息，并且是最新消息。 2.过去6秒内 和 zk直接发送过心跳。   

##### 生产者保证
- 选择合适的ack  
这个简单，通过配置ack的属性来控制。0的话不用说，消息很容易丢失。如果是1，那么代表发送过去，等待首领副本确认消息，认为成功。这里就有点意思了，首领肯定收到了消息，写入了分区文件，但是不一定落盘。如果是all,那么代表发送过去之后，消息被写入所有同步副本之后才会认为成功。  
注意这里是**所有同步副本**，不是所有副本。具体是多少同步副本，还要取决于上面设置的最小同步副本数和集群当前的同步副本数。选择这种配置会可靠，但是牺牲效率，不过，这个效率可以通过使用异步模式来提高。  
- 使用消息发送回调函数保证消息  
- 配置消息重发，减少因为网络波动的影响  

`所以，如果对于消息丢失要求十分严格，可以采用ack=all+合适的最小同步副本数、集群当前的同步副本数+异步send+回调函数+消息重发`  

##### 消费者保证
- 根据具体的业务来选择消息队列的消费方式，一种是发布订阅模式，一种是队列模式，尤其是group-id要配对  
- 设定好没有偏移量可以提交的时候，系统从哪里开始消费。`auto.offset.reset`的选择  
- 关闭自动提交，确认消费完成后再提交偏移量  
如果开启了自动提交，那么系统会自动进行提交offset。可能会引起，并未消费掉，就提交了offset.引起数据的丢失。与自动提交相关的是自动提交的间隔时间`auto.commit.interval.ms` 默认是5秒钟提交一次。  
但是这种方法可能引起消息的重复消费  
- 通过`ConsumerRebalanceListener`监听接口的实现类+`seek()`特定偏移量处开始处理消息，来保证重平衡后的offset提交准确性  