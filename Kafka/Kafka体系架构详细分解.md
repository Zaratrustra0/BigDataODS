# Kafka体系架构详细分解
点击上方**蓝色字体**，选择 “**设为星标**”

回复” 资源 “获取更多资源

![](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg)

作者：luozhiyun

地址：[http://suo.im/5uYoJ0](http://suo.im/5uYoJ0)

[![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2OflopvKZ9wvmiaF2Mm2eU12FNITkgCB4EJmYLnYicozYViaHmuU4U0kaO6nYRIAwy5hpBqNnbqiaxHyQ/640?wx_fmt=jpeg)
](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

[**大数据技术与架构**](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

点击右侧关注，大数据开发领域最强公众号！

[![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2MSGY5RTojm5sPV2pSicy3oK14Micdx2DKIZ4BIg0U5ic0nA0IhQ07XHQB52oOvibaI1OibicKUOYFgL77g/640?wx_fmt=jpeg)
](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2OflopvKZ9wvmiaF2Mm2eU12qDVEwFpcVj3wMTHSWPbg0HsibEvVCxnbc9sQFmaAX6NeQax13IquBNg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)

[**暴走大数据**](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)

点击右侧关注，暴走大数据！

[![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PicAKtwvjRxPMic7Bia3foLlzB01ia6f1SITvsCNveIlJXLGzvb5icuQ1nBpcmqH68AAUO3owU01Z9dlA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)

## 基本概念

### Kafka 体系架构

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibT0tcib1gxJknTnBQNj61TNibsX5MaB7dZnE6hm3t5ia8FX5ALcXNX2ibSIg/640?wx_fmt=png)

Kafka 体系架构包括若干 Producer、若干 Broker、若干 Consumer，以及一个 ZooKeeper 集群。

在 Kafka 中还有两个特别重要的概念—主题（Topic）与分区（Partition）。

Kafka 中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题（发送到 Kafka 集群中的每一条消息都要指定一个主题），而消费者负责订阅主题并进行消费。

主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题，很多时候也会把分区称为主题分区（Topic-Partition）。

Kafka 为分区引入了多副本（Replica）机制，通过增加副本数量可以提升容灾能力。同一分区的不同副本中保存的是相同的消息（在同一时刻，副本之间并非完全一样），副本之间是 “一主多从” 的关系，其中 leader 副本负责处理读写请求，follower 副本只负责与 leader 副本的消息同步。当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供服务。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTCZaE4Z2MRnsMBgibOs33EpPfdmU0iadZKGAia6NnJAN4qFXUZvFNLMszw/640?wx_fmt=png)

如上图所示，Kafka 集群中有 4 个 broker，某个主题中有 3 个分区，且副本因子（即副本个数）也为 3，如此每个分区便有 1 个 leader 副本和 2 个 follower 副本。

### 数据同步

分区中的所有副本统称为 AR（Assigned Replicas）。所有与 leader 副本保持一定程度同步的副本（包括 leader 副本在内）组成 ISR（In-Sync Replicas），ISR 集合是 AR 集合中的一个子集。

与 leader 副本同步滞后过多的副本（不包括 leader 副本）组成 OSR（Out-of-Sync Replicas），由此可见，AR=ISR+OSR。在正常情况下，所有的 follower 副本都应该与 leader 副本保持一定程度的同步，即 AR=ISR，OSR 集合为空。

Leader 副本负责维护和跟踪 ISR 集合中所有 follower 副本的滞后状态，当 follower 副本落后太多或失效时，leader 副本会把它从 ISR 集合中剔除。默认情况下，当 leader 副本发生故障时，只有在 ISR 集合中的副本才有资格被选举为新的 leader。

HW 是 High Watermark 的缩写，俗称高水位，它标识了一个特定的消息偏移量（offset），消费者只能拉取到这个 offset 之前的消息。  
LEO 是 Log End Offset 的缩写，它标识当前日志文件中下一条待写入消息的 offset。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTibEjA31PbPpgIORn2YFVnu4zk8tRl4nXEROkmIac8xukIhQK8icjozRg/640?wx_fmt=png)

如上图所示，第一条消息的 offset（LogStartOffset）为 0，最后一条消息的 offset 为 8，offset 为 9 的消息用虚线框表示，代表下一条待写入的消息。日志文件的 HW 为 6，表示消费者只能拉取到 offset 在 0 至 5 之间的消息，而 offset 为 6 的消息对消费者而言是不可见的。

## Kafka 生产者客户端的整体结构

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTbqMkfGpJ8kAEqEibf0XPVxJ9KlkGpvzmqePre4VIBcF9aiaiakwGic2zqw/640?wx_fmt=png)

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和 Sender 线程（发送线程）。

在主线程中由 KafkaProducer 创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（RecordAccumulator，也称为消息收集器）中。Sender 线程负责从 RecordAccumulator 中获取消息并将其发送到 Kafka 中。

**RecordAccumulator**  
RecordAccumulator 主要用来缓存消息以便 Sender 线程可以批量发送，进而减少网络传输的资源消耗以提升性能。

主线程中发送过来的消息都会被追加到 RecordAccumulator 的某个双端队列（Deque）中，在 RecordAccumulator 的内部为每个分区都维护了一个双端队列。

消息写入缓存时，追加到双端队列的尾部；Sender 读取消息时，从双端队列的头部读取。

Sender 从 RecordAccumulator 中获取缓存的消息之后，会进一步将原本 &lt;分区, Deque&lt; ProducerBatch>> 的保存形式转变成 &lt;Node, List&lt; ProducerBatch> 的形式，其中 Node 表示 Kafka 集群的 broker 节点。

KafkaProducer 要将此消息追加到指定主题的某个分区所对应的 leader 副本之前，首先需要知道主题的分区数量，然后经过计算得出（或者直接指定）目标分区，之后 KafkaProducer 需要知道目标分区的 leader 副本所在的 broker 节点的地址、端口等信息才能建立连接，最终才能将消息发送到 Kafka。

所以这里需要一个转换，对于网络连接来说，生产者客户端是与具体的 broker 节点建立的连接，也就是向具体的 broker 节点发送消息，而并不关心消息属于哪一个分区。

**InFlightRequests**  
请求在从 Sender 线程发往 Kafka 之前还会保存到 InFlightRequests 中，InFlightRequests 保存对象的具体形式为 Map&lt;NodeId, Deque>，它的主要作用是缓存了已经发出去但还没有收到响应的请求（NodeId 是一个 String 类型，表示节点的 id 编号）。

### 拦截器

生产者拦截器既可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。

生产者拦截器的使用也很方便，主要是自定义实现 org.apache.kafka.clients.producer. ProducerInterceptor 接口。ProducerInterceptor 接口中包含 3 个方法：

Copy

`public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);  
public void onAcknowledgement(RecordMetadata metadata, Exception exception);  
public void close();  
`

KafkaProducer 在将消息序列化和计算分区之前会调用生产者拦截器的 onSend() 方法来对消息进行相应的定制化操作。一般来说最好不要修改消息 ProducerRecord 的 topic、key 和 partition 等信息。

KafkaProducer 会在消息被应答（Acknowledgement）之前或消息发送失败时调用生产者拦截器的 onAcknowledgement() 方法，优先于用户设定的 Callback 之前执行。这个方法运行在 Producer 的 I/O 线程中，所以这个方法中实现的代码逻辑越简单越好，否则会影响消息的发送速度。

close() 方法主要用于在关闭拦截器时执行一些资源的清理工作。

### 序列化器

生产者需要用序列化器（Serializer）把对象转换成字节数组才能通过网络发送给 Kafka。而在对侧，消费者需要用反序列化器（Deserializer）把从 Kafka 中收到的字节数组转换成相应的对象。

生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的，如果生产者使用了某种序列化器，比如 StringSerializer，而消费者使用了另一种序列化器，比如 IntegerSerializer，那么是无法解析出想要的数据的。

序列化器都需要实现 org.apache.kafka.common.serialization.Serializer 接口，此接口有 3 个方法：

Copy

`public void configure(Map<String, ?> configs, boolean isKey)  
public byte[] serialize(String topic, T data)  
public void close()  
`

configure() 方法用来配置当前类，serialize() 方法用来执行序列化操作。而 close() 方法用来关闭当前的序列化器。

如下：

Copy

\`public class StringSerializer implements Serializer<String> {  
    private String encoding = "UTF8";

    @Override  
    public void configure(Map<String, ?> configs, boolean isKey) {  
        String propertyName = isKey ? "key.serializer.encoding" :  
                "value.serializer.encoding";  
        Object encodingValue = configs.get(propertyName);  
        if (encodingValue == null)  
            encodingValue = configs.get("serializer.encoding");  
        if (encodingValue != null && encodingValue instanceof String)  
            encoding = (String) encodingValue;  
    }

    @Override  
    public byte[] serialize(String topic, String data) {  
        try {  
            if (data == null)  
                return null;  
            else  
                return data.getBytes(encoding);  
        } catch (UnsupportedEncodingException e) {  
            throw new SerializationException("Error when serializing " +  
                    "string to byte[] due to unsupported encoding " + encoding);  
        }  
    }

    @Override  
    public void close() {  
        // nothing to do  
    }  

}

\`

configure() 方法，这个方法是在创建 KafkaProducer 实例的时候调用的，主要用来确定编码类型。

serialize 用来编解码，如果 Kafka 客户端提供的几种序列化器都无法满足应用需求，则可以选择使用如 Avro、JSON、Thrift、ProtoBuf 和 Protostuff 等通用的序列化工具来实现，或者使用自定义类型的序列化器来实现。

### 分区器

消息经过序列化之后就需要确定它发往的分区，如果消息 ProducerRecord 中指定了 partition 字段，那么就不需要分区器的作用，因为 partition 代表的就是所要发往的分区号。

如果消息 ProducerRecord 中没有指定 partition 字段，那么就需要依赖分区器，根据 key 这个字段来计算 partition 的值。分区器的作用就是为消息分配分区。

Kafka 中提供的默认分区器是 org.apache.kafka.clients.producer.internals.DefaultPartitioner，它实现了 org.apache.kafka.clients.producer.Partitioner 接口，这个接口中定义了 2 个方法，具体如下所示。

Copy

`public int partition(String topic, Object key, byte[] keyBytes,  
                     Object value, byte[] valueBytes, Cluster cluster);  
public void close();  
`

其中 partition() 方法用来计算分区号，返回值为 int 类型。partition() 方法中的参数分别表示主题、键、序列化后的键、值、序列化后的值，以及集群的元数据信息，通过这些信息可以实现功能丰富的分区器。close() 方法在关闭分区器的时候用来回收一些资源。

在默认分区器 DefaultPartitioner 的实现中，close() 是空方法，而在 partition() 方法中定义了主要的分区分配逻辑。如果 key 不为 null，那么默认的分区器会对 key 进行哈希，最终根据得到的哈希值来计算分区号，拥有相同 key 的消息会被写入同一个分区。如果 key 为 null，那么消息将会以轮询的方式发往主题内的各个可用分区。

自定义的分区器，只需同 DefaultPartitioner 一样实现 Partitioner 接口即可。由于每个分区下的消息处理都是有顺序的，我们可以利用自定义分区器实现在某一系列的 key 都发送到一个分区中，从而实现有序消费。

## Broker

### Broker 处理请求流程

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTE4VNibrUMicO2yFibEHwVOwxyjQm6ycicia9sz3rvSXy3LIxpjo64iau6YXw/640?wx_fmt=png)

在 Kafka 的架构中，会有很多客户端向 Broker 端发送请求，Kafka 的 Broker 端有个 SocketServer 组件，用来和客户端建立连接，然后通过 Acceptor 线程来进行请求的分发，由于 Acceptor 不涉及具体的逻辑处理，非常得轻量级，因此有很高的吞吐量。

接着 Acceptor 线程采用轮询的方式将入站请求公平地发到所有网络线程中，网络线程池默认大小是 3 个，表示每台 Broker 启动时会创建 3 个网络线程，专门处理客户端发送的请求，可以通过 Broker 端参数 num.network.threads 来进行修改。

那么接下来处理网络线程处理流程如下：  
![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTgWE88oHr0CticKKK4x6MIctFoUG9mtoW2fUg5yDZW2rtcY3eiahWneOg/640?wx_fmt=png)

当网络线程拿到请求后，会将请求放入到一个共享请求队列中。Broker 端还有个 IO 线程池，负责从该队列中取出请求，执行真正的处理。如果是 PRODUCE 生产请求，则将消息写入到底层的磁盘日志中；如果是 FETCH 请求，则从磁盘或页缓存中读取消息。

IO 线程池处中的线程是执行请求逻辑的线程，默认是 8，表示每台 Broker 启动后自动创建 8 个 IO 线程处理请求，可以通过 Broker 端参数 num.io.threads 调整。

Purgatory 组件是用来缓存延时请求（Delayed Request）的。比如设置了 acks=all 的 PRODUCE 请求，一旦设置了 acks=all，那么该请求就必须等待 ISR 中所有副本都接收了消息后才能返回，此时处理该请求的 IO 线程就必须等待其他 Broker 的写入结果。

### 控制器

在 Kafka 集群中会有一个或多个 broker，其中有一个 broker 会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。

#### 控制器是如何被选出来的？

Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：第一个成功创建 /controller 节点的 Broker 会被指定为控制器。

在 ZooKeeper 中的 /controller_epoch 节点中存放的是一个整型的 controller_epoch 值。controller_epoch 用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为 “控制器的纪元”。

controller_epoch 的初始值为 1，即集群中第一个控制器的纪元为 1，当控制器发生变更时，每选出一个新的控制器就将该字段值加 1。Kafka 通过 controller_epoch 来保证控制器的唯一性，进而保证相关操作的一致性。

每个和控制器交互的请求都会携带 controller_epoch 这个字段，如果请求的 controller_epoch 值小于内存中的 controller_epoch 值，则认为这个请求是向已经过期的控制器所发送的请求，那么这个请求会被认定为无效的请求。

如果请求的 controller_epoch 值大于内存中的 controller_epoch 值，那么说明已经有新的控制器当选了。

#### 控制器是做什么的？

-   主题管理（创建、删除、增加分区）
-   分区重分配
-   Preferred 领导者选举  
    Preferred 领导者选举主要是 Kafka 为了避免部分 Broker 负载过重而提供的一种换 Leader 的方案。
-   集群成员管理（新增 Broker、Broker 主动关闭、Broker 宕机）  
    控制器组件会利用 Watch 机制检查 ZooKeeper 的 /brokers/ids 节点下的子节点数量变更。目前，当有新 Broker 启动后，它会在 /brokers 下创建专属的 znode 节点。一旦创建完毕，ZooKeeper 会通过 Watch 机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker 作业。
-   数据服务  
    控制器上保存了最全的集群元数据信息。  
    ![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTODoNOAYSkjGnP6Cf1X2hGPQVZQGiaRM11oa2PmibFiaAumGeJYYXJxkzg/640?wx_fmt=jpeg)

#### 控制器宕机了怎么办？

当运行中的控制器突然宕机或意外终止时，Kafka 能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器。这个过程就被称为 Failover，该过程是自动完成的，无需你手动干预。

![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTcpQGocnraVRvjNiad5NZqaicSyoiaiaya48RcziaHfuSAm7KOCWOHc3Ciang/640?wx_fmt=jpeg)

## 消费者

### 消费组

在 Kafka 中，每个消费者都有一个对应的消费组。当消息发布到主题后，只会被投递给订阅它的每个消费组中的一个消费者。每个消费者只能消费所分配到的分区中的消息。而每一个分区只能被一个消费组中的一个消费者所消费。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTH7WzwF9WGJ4WgaIbHFGOqvPnkk8aicK1Uq8ZwxibYNYkvrR0EiaGWD6tg/640?wx_fmt=png)

如上图所示，我们可以设置两个消费者组来实现广播消息的作用，消费组 A 和组 B 都可以接受到生产者发送过来的消息。

消费者与消费组这种模型可以让整体的消费能力具备横向伸缩性，我们可以增加（或减少）消费者的个数来提高（或降低）整体的消费能力。对于分区数固定的情况，一味地增加消费者并不会让消费能力一直得到提升，如果消费者过多，出现了**消费者的个数大于分区个数**的情况，就会有消费者分配不到任何分区。

如下：一共有 8 个消费者，7 个分区，那么最后的消费者 C7 由于分配不到任何分区而无法消费任何消息。  
![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTrYdtryqFpNjib3ibz89crHNtI4keSKQiarfHsUgB5TZweDFqEpnwImDtQ/640?wx_fmt=png)

### 消费端分区分配策略

Kafka 提供了消费者客户端参数 partition.assignment.strategy 来设置消费者与订阅主题之间的分区分配策略。

**RangeAssignor 分配策略**  
默认情况下，采用 RangeAssignor 分配策略。

RangeAssignor 分配策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配，以保证分区尽可能均匀地分配给所有的消费者。对于每一个主题，RangeAssignor 策略会将消费组内所有订阅这个主题的消费者按照名称的字典序排序，然后为每个消费者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会被多分配一个分区。

假设消费组内有 2 个消费者 C0 和 C1，都订阅了主题 t0 和 t1，并且每个主题都有 4 个分区，那么订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t0p3、t1p0、t1p1、t1p2、t1p3。最终的分配结果为：

Copy

`消费者 C0：t0p0、t0p1、t1p0、t1p1  
消费者 C1：t0p2、t0p3、t1p2、t1p3  
`

假设上面例子中 2 个主题都只有 3 个分区，那么订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：

Copy

`消费者 C0：t0p0、t0p1、t1p0、t1p1  
消费者 C1：t0p2、t1p2  
`

可以明显地看到这样的分配并不均匀。

**RoundRobinAssignor 分配策略**  
RoundRobinAssignor 分配策略的原理是将消费组内所有消费者及消费者订阅的所有主题的分区按照字典序排序，然后通过轮询方式逐个将分区依次分配给每个消费者。

如果同一个消费组内所有的消费者的订阅信息都是相同的，那么 RoundRobinAssignor 分配策略的分区分配会是均匀的。

如果同一个消费组内的消费者订阅的信息是不相同的，那么在执行分区分配的时候就不是完全的轮询分配，有可能导致分区分配得不均匀。

假设消费组内有 3 个消费者（C0、C1 和 C2），t0、t0、t1、t2 主题分别有 1、2、3 个分区，即整个消费组订阅了 t0p0、t1p0、t1p1、t2p0、t2p1、t2p2 这 6 个分区。

具体而言，消费者 C0 订阅的是主题 t0，消费者 C1 订阅的是主题 t0 和 t1，消费者 C2 订阅的是主题 t0、t1 和 t2，那么最终的分配结果为：

Copy

`消费者 C0：t0p0  
消费者 C1：t1p0  
消费者 C2：t1p1、t2p0、t2p1、t2p2  
`

可以看 到 RoundRobinAssignor 策略也不是十分完美，这样分配其实并不是最优解，因为完全可以将分区 t1p1 分配给消费者 C1。

**StickyAssignor 分配策略**  
这种分配策略，它主要有两个目的：

1.  分区的分配要尽可能均匀。
2.  分区的分配尽可能与上次分配的保持相同。

假设消费组内有 3 个消费者（C0、C1 和 C2），它们都订阅了 4 个主题（t0、t1、t2、t3），并且每个主题有 2 个分区。也就是说，整个消费组订阅了 t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1 这 8 个分区。最终的分配结果如下：

Copy

`消费者 C0：t0p0、t1p1、t3p0  
消费者 C1：t0p1、t2p0、t3p1  
消费者 C2：t1p0、t2p1  
`

再假设此时消费者 C1 脱离了消费组，那么分配结果为：

Copy

`消费者 C0：t0p0、t1p1、t3p0、t2p0  
消费者 C2：t1p0、t2p1、t0p1、t3p1  
`

StickyAssignor 分配策略如同其名称中的 “sticky” 一样，让分配策略具备一定的“黏性”，尽可能地让前后两次分配相同，进而减少系统资源的损耗及其他异常情况的发生。

### 再均衡（Rebalance）

再均衡是指分区的所属权从一个消费者转移到另一消费者的行为，它为消费组具备高可用性和伸缩性提供保障，使我们可以既方便又安全地删除消费组内的消费者或往消费组内添加消费者。

弊端：

1.  在再均衡发生期间，消费组内的消费者是无法读取消息的。
2.  Rebalance 很慢。如果一个消费者组里面有几百个 Consumer 实例，Rebalance 一次要几个小时。
3.  在进行再均衡的时候消费者当前的状态也会丢失。比如消费者消费完某个分区中的一部分消息时还没有来得及提交消费位移就发生了再均衡操作，之后这个分区又被分配给了消费组内的另一个消费者，原来被消费完的那部分消息又被重新消费一遍，也就是发生了重复消费。

Rebalance 发生的时机有三个：

1.  组成员数量发生变化
2.  订阅主题数量发生变化
3.  订阅主题的分区数发生变化

后两类通常是业务的变动调整所导致的，我们一般不可控制，我们主要说说因为组成员数量变化而引发的 Rebalance 该如何避免。

当 Consumer Group 完成 Rebalance 之后，每个 Consumer 实例都会定期地向 Coordinator 发送心跳请求，表明它还存活着。如果某个 Consumer 实例不能及时地发送这些心跳请求，Coordinator 就会认为该 Consumer 已经 “死” 了，从而将其从 Group 中移除，然后开启新一轮 Rebalance。

Consumer 端可以设置**session.timeout.ms**，默认是 10s，表示如果 Coordinator 在 10 秒之内没有收到 Group 下某 Consumer 实例的心跳，它就会认为这个 Consumer 实例已经挂了。

Consumer 端还可以设置**heartbeat.interval.ms**，表示发送心跳请求的频率。

以及**max.poll.interval.ms** 参数，它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。它的默认值是 5 分钟，表示你的 Consumer 程序如果在 5 分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会主动发起 “离开组” 的请求，Coordinator 也会开启新一轮 Rebalance。

所以知道了上面几个参数后，我们就可以避免以下两个问题：

1.  非必要 Rebalance 是因为未能及时发送心跳，导致 Consumer 被 “踢出”Group 而引发的。  
    所以我们在生产环境中可以这么设置：

-   设置 session.timeout.ms = 6s。
-   设置 heartbeat.interval.ms = 2s。

3.  必要 Rebalance 是 Consumer 消费时间过长导致的。如何消费任务时间达到 8 分钟，而 max.poll.interval.ms 设置为 5 分钟，那么也会发生 Rebalance，所以如果有比较重的任务的话，可以适当调整这个参数。
4.  Consumer 端的频繁的 Full GC 导致的长时间停顿，从而引发了 Rebalance。

### 消费者组再平衡全流程

重平衡过程是靠消费者端的心跳线程（Heartbeat Thread），通知到其他消费者实例的。

当协调者决定开启新一轮重平衡后，它会将 “REBALANCE_IN_PROGRESS” 封装进心跳请求的响应中，发还给消费者实例。当消费者实例发现心跳响应中包含了 “REBALANCE_IN_PROGRESS”，就能立马知道重平衡又开始了，这就是重平衡的通知机制。

所以，实际上 heartbeat.interval.ms 不止是设置了心跳的间隔时间，还可以控制重平衡通知的频率。

#### 消费者组状态机

重平衡一旦开启，Broker 端的协调者组件就要完成整个重平衡流程，Kafka 设计了一套消费者组状态机（State Machine）来实现。

Kafka 为消费者组定义了 5 种状态，它们分别是：Empty、Dead、PreparingRebalance、CompletingRebalance 和 Stable。

![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTk6zNNiaRuflLmdWQs8a6JD92KW5cX8IO5jQNGQ7etibrmflZicfqOPvsg/640?wx_fmt=jpeg)

状态机的各个状态流转：  
![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTk6zNNiaRuflLmdWQs8a6JD92KW5cX8IO5jQNGQ7etibrmflZicfqOPvsg/640?wx_fmt=jpeg)

当有新成员加入或已有成员退出时，消费者组的状态从 Stable 直接跳到 PreparingRebalance 状态，此时，所有现存成员就必须重新申请加入组。当所有成员都退出组后，消费者组状态变更为 Empty。Kafka 定期自动删除过期位移的条件就是，组要处于 Empty 状态。因此，如果你的消费者组停掉了很长时间（超过 7 天），那么 Kafka 很可能就把该组的位移数据删除了。

#### 组协调器（GroupCoordinator）

GroupCoordinator 是 Kafka 服务端中用于管理消费组的组件。协调器最重要的职责就是负责执行消费者再均衡的操作。

#### 消费者端重平衡流程

在消费者端，重平衡分为两个步骤：分别是加入组和等待领导者消费者（Leader Consumer）分配方案。即 JoinGroup 请求和 SyncGroup 请求。

1.  加入组  
    当组内成员加入组时，它会向协调器发送 JoinGroup 请求。在该请求中，每个成员都要将自己订阅的主题上报，这样协调器就能收集到所有成员的订阅信息。
2.  选择消费组领导者  
    一旦收集了全部成员的 JoinGroup 请求后，协调者会从这些成员中选择一个担任这个消费者组的领导者。  
    这里的领导者是具体的消费者实例，它既不是副本，也不是协调器。领导者消费者的任务是收集所有成员的订阅信息，然后根据这些信息，制定具体的分区消费分配方案。
3.  选举分区分配策略  
    这个分区分配的选举是根据消费组内的各个消费者投票来决定的。  
    协调器会收集各个消费者支持的所有分配策略，组成候选集 candidates。每个消费者从候选集 candidates 中找出第一个自身支持的策略，为这个策略投上一票。计算候选集中各个策略的选票数，选票数最多的策略即为当前消费组的分配策略。  
    如果有消费者并不支持选出的分配策略，那么就会报出异常 IllegalArgumentException：Member does not support protocol。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTHbm4DfjicOssJJI65qkMH2Dy5KI3MScKuEuCFm44UNESYKIesOoibicOw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTHl77FDVLxg3HWrtOWMJ80t75ywlDYckiaZO8vHmjiaCtwZt0tfy1M5KA/640?wx_fmt=png)

4.  发送 SyncGroup 请求  
    协调器会把消费者组订阅信息封装进 JoinGroup 请求的响应体中，然后发给领导者，由领导者统一做出分配方案，然后领导者发送 SyncGroup 请求给协调器。  
    ![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M3QBcLamp7mDxTLBcYiaWibTtBXEyn3PAAwicA2c2srrQogic0xWLpcD8X45fVGoXGdKIIFvEIHW5LjQ/640?wx_fmt=png)
5.  响应 SyncGroup  
    组内所有的消费者都会发送一个 SyncGroup 请求，只不过不是领导者的请求内容为空，然后就会接收到一个 SyncGroup 响应，接受订阅信息。

**欢迎点赞 + 收藏 + 转发朋友圈素质三连**

\***\* 文章不错？\*\***点个【在看】吧！**\*\***👇**\*\*** 
 [https://mp.weixin.qq.com/s/C6dfvzFkNDYgiNeZ4eWPBQ](https://mp.weixin.qq.com/s/C6dfvzFkNDYgiNeZ4eWPBQ)
