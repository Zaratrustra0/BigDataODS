# 32 道常见的 Kafka 面试题你都会吗？附答案
最近很多粉丝后台留言问了一些大数据的面试题，其中包括了大量的 Kafka、Spark 等相关的问题，所以我特意抽出时间整理了一些大数据相关面试题，本文是 Kafka 面试相关问题，其他系列面试题后面会陆续整理，欢迎关注**过往记忆大数据**公众号。

1、Kafka 都有哪些特点？

-   高吞吐量、低延迟：kafka 每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个 topic 可以分多个 partition, consumer group 对 partition 进行 consume 操作。
-   可扩展性：kafka 集群支持热扩展
-   持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
-   容错性：允许集群中节点失败（若副本数量为 n, 则允许 n-1 个节点失败）
-   高并发：支持数千个客户端同时读写

2、请简述下你在哪些场景下会选择 Kafka？  

-   日志收集：一个公司可以用 Kafka 可以收集各种服务的 log，通过 kafka 以统一接口服务的方式开放给各种 consumer，例如 hadoop、HBase、Solr 等。
-   消息系统：解耦和生产者和消费者、缓存消息等。
-   用户活动跟踪：Kafka 经常被用来记录 web 用户或者 app 用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到 kafka 的 topic 中，然后订阅者通过订阅这些 topic 来做实时的监控分析，或者装载到 hadoop、数据仓库中做离线分析和挖掘。
-   运营指标：Kafka 也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
-   流式处理：比如 spark streaming 和 Flink

3、 Kafka 的设计架构你知道吗？

简单架构如下

![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0nv1fnjRA095Q28vuBibmUu2ia75fEf9U6Qz9XMKibBOBokLC5JQyYTkUqxUicOWyzmInwssk1O8KGzYgQ/640?wx_fmt=png)

详细如下

![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0nv1fnjRA095Q28vuBibmUu2iamzHvw8rL7oicEic4b0fej5eJSytgVXbG2JlytKMtTZcUcbw7rPiclIw3w/640?wx_fmt=png)

Kafka 架构分为以下几个部分

-   Producer ：消息生产者，就是向 kafka broker 发消息的客户端。
-   Consumer ：消息消费者，向 kafka broker 取消息的客户端。  
-   Topic ：可以理解为一个队列，一个 Topic 又分为一个或多个分区，  
-   Consumer Group：这是 kafka 用来实现一个 topic 消息的广播（发给所有的 consumer）和单播（发给任意一个 consumer）的手段。一个 topic 可以有多个 Consumer Group。  
-   Broker ：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic。  
-   Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker 上，每个 partition 是一个有序的队列。partition 中的每条消息都会被分配一个有序的 id（offset）。将消息发给 consumer，kafka 只保证按一个 partition 中的消息的顺序，不保证一个 topic 的整体（多个 partition 间）的顺序。  
-   Offset：kafka 的存储文件都是按照 offset.kafka 来命名，用 offset 做名字的好处是方便查找。例如你想找位于 2049 的位置，只要找到 2048.kafka 的文件即可。当然 the first offset 就是 00000000000.kafka。

4、Kafka 分区的目的？  

分区对于 Kafka 集群的好处是：实现负载均衡。分区对于消费者来说，可以提高并发度，提高效率。

5、你知道 Kafka 是如何做到消息的有序性？

kafka 中的每个 partition 中的消息在写入时都是有序的，而且单独一个 partition 只能由一个消费者去消费，可以在里面保证消息的顺序性。但是分区之间的消息是不保证有序的。

6、Kafka 的高可靠性是怎么实现的？

可以参见我这篇文章：[Kafka 是如何保证数据可靠性和一致性](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716970&idx=1&sn=3875dd83ca35c683bfa42135c55a03ab&chksm=887da65cbf0a2f4aeae51f4d41fa8dec9c66af17fbc423eb5a1b0d35d20348880c8b2539ddbf&scene=21#wechat_redirect)  

7、请谈一谈 Kafka 数据一致性原理

一致性就是说不论是老的 Leader 还是新选举的 Leader，Consumer 都能读到一样的数据。

![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0nvrlvQw2rzMhxIzPfTGxF6LiaI8D8udv6lVEETsia8bqAorQeV9UAMToQy2UdLUvVJN1LCvjp34z5yg/640?wx_fmt=png)

假设分区的副本为 3，其中副本 0 是 Leader，副本 1 和副本 2 是 follower，并且在 ISR 列表里面。虽然副本 0 已经写入了 Message4，但是 Consumer 只能读取到 Message2。因为所有的 ISR 都同步了 Message2，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，对应于上图的副本 2，这个很类似于木桶原理。

这样做的原因是还没有被足够多副本复制的消息被认为是 “不安全” 的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。如果我们允许消费者读取这些消息，可能就会破坏一致性。试想，一个消费者从当前 Leader（副本 0） 读取并处理了 Message4，这个时候 Leader 挂掉了，选举了副本 1 为新的 Leader，这时候另一个消费者再去从新的 Leader 读取消息，发现这个消息其实并不存在，这就导致了数据不一致性问题。

当然，引入了 High Water Mark 机制，会导致 Broker 间的消息复制因为某些原因变慢，那么消息到达消费者的时间也会随之变长（因为我们会先等待消息复制完毕）。延迟时间可以通过参数 replica.lag.time.max.ms 参数配置，它指定了副本在复制消息时可被允许的最大延迟时间。

8、ISR、OSR、AR 是什么？  

ISR：In-Sync Replicas 副本同步队列

OSR：Out-of-Sync Replicas 

AR：Assigned Replicas 所有副本

ISR 是由 leader 维护，follower 从 leader 同步数据有一些延迟（具体可以参见 [图文了解 Kafka 的副本复制机制](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716907&idx=1&sn=3aaf4be490baf2697633b7470cf76457&chksm=887da79dbf0a2e8b31983bf37018adb0e9026c87dff5d1d43a3b4209580d71ecbe1f085be96f&scene=21#wechat_redirect)），超过相应的阈值会把 follower 剔除出 ISR, 存入 OSR（Out-of-Sync Replicas）列表，新加入的 follower 也会先存放在 OSR 中。AR=ISR+OSR。

9、LEO、HW、LSO、LW 等分别代表什么

-   LEO：是 LogEndOffset 的简称，代表当前日志文件中下一条
-   HW：水位或水印（watermark）一词，也可称为高水位 (high watermark)，通常被用在流式处理领域（比如 Apache Flink、Apache Spark 等），以表征元素或事件在基于时间层面上的进度。在 Kafka 中，水位的概念反而与时间无关，而是与位置信息相关。严格来说，它表示的就是位置信息，即位移（offset）。取 partition 对应的 ISR 中 最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置上一条信息。
-   LSO：是 LastStableOffset 的简称，对未完成的事务而言，LSO 的值等于事务中第一条消息的位置 (firstUnstableOffset)，对已完成的事务而言，它的值同 HW 相同 
-   LW：Low Watermark 低水位, 代表 AR 集合中最小的 logStartOffset 值。

10、Kafka 在什么情况下会出现消息丢失？

可以参见我这篇文章：[Kafka 是如何保证数据可靠性和一致性](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716970&idx=1&sn=3875dd83ca35c683bfa42135c55a03ab&chksm=887da65cbf0a2f4aeae51f4d41fa8dec9c66af17fbc423eb5a1b0d35d20348880c8b2539ddbf&scene=21#wechat_redirect)

11、怎么尽可能保证 Kafka 的可靠性  

可以参见我这篇文章：[Kafka 是如何保证数据可靠性和一致性](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716970&idx=1&sn=3875dd83ca35c683bfa42135c55a03ab&chksm=887da65cbf0a2f4aeae51f4d41fa8dec9c66af17fbc423eb5a1b0d35d20348880c8b2539ddbf&scene=21#wechat_redirect)

12、消费者和消费者组有什么关系？  

每个消费者从属于消费组。具体关系如下：

![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0ntKhPVwWAsmic9hgxmoLZJw0ulkJUgHGYtmqeGF1HLXP4nRIC2yicV9qsNjibibwZaAEbR3Feic3Eqf88A/640?wx_fmt=png)

13、Kafka 的每个分区只能被一个消费者线程，如何做到多个线程同时消费一个分区？

[参见我这篇文章：Airbnb 是如何通过 balanced Kafka reader 来扩展 Spark streaming 实时流处理能力的](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716863&idx=1&sn=20d42a18ad084eb1adc8db126cae71cd&chksm=887da7c9bf0a2edf8b30a3dd783a4f3f1097078ff3a840d25f57b8eaa06815c63fadd0bcb68b&scene=21#wechat_redirect)  

14、数据传输的事务有几种？  

数据传输的事务定义通常有以下三种级别：

（1）最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输

（2）最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.

（3）精确的一次（Exactly once）: 不会漏传输也不会重复传输, 每个消息都传输被

15、Kafka 消费者是否可以消费指定分区消息？

Kafa consumer 消费消息时，向 broker 发出 fetch 请求去消费特定分区的消息，consumer 指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer 拥有了 offset 的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的

16、Kafka 消息是采用 Pull 模式，还是 Push 模式？

Kafka 最初考虑的问题是，customer 应该从 brokes 拉取消息还是 brokers 将消息推送到 consumer，也就是 pull 还 push。在这方面，Kafka 遵循了一种大部分消息系统共同的传统的设计：producer 将消息推送到 broker，consumer 从 broker 拉取消息。

一些消息系统比如 Scribe 和 Apache Flume 采用了 push 模式，将消息推送到下游的 consumer。这样做有好处也有坏处：由 broker 决定消息推送的速率，对于不同消费速率的 consumer 就不太好处理了。消息系统都致力于让 consumer 以最大的速率最快速的消费消息，但不幸的是，push 模式下，当 broker 推送的速率远大于 consumer 消费的速率时，consumer 恐怕就要崩溃了。最终 Kafka 还是选取了传统的 pull 模式。

Pull 模式的另外一个好处是 consumer 可以自主决定是否批量的从 broker 拉取数据。Push 模式必须在不知道下游 consumer 消费能力和消费策略的情况下决定是立即推送每条消息还是缓存之后批量推送。如果为了避免 consumer 崩溃而采用较低的推送速率，将可能导致一次只推送较少的消息而造成浪费。Pull 模式下，consumer 就可以根据自己的消费能力去决定这些策略。

Pull 有个缺点是，如果 broker 没有可供消费的消息，将导致 consumer 不断在循环中轮询，直到新消息到 t 达。为了避免这点，Kafka 有个参数可以让 consumer 阻塞知道新消息到达 (当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发

17、Kafka 消息格式的演变清楚吗？

Kafka 的消息格式经过了四次大变化，具体可以参见我这篇文章：[Apache Kafka 消息格式的演变 (0.7.x~0.10.x)](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650714506&idx=1&sn=6499c694e0ab80a8cf0186544507dfd0&chksm=887dacfcbf0a25ea2106ccddbaa8c39bae2f4dd0754a4528bc198d0b8d164115b2ea103db6ff&scene=21#wechat_redirect)。[](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650714506&idx=1&sn=6499c694e0ab80a8cf0186544507dfd0&chksm=887dacfcbf0a25ea2106ccddbaa8c39bae2f4dd0754a4528bc198d0b8d164115b2ea103db6ff&scene=21#wechat_redirect)

18、Kafka 偏移量的演变清楚吗？  

参见我这篇文章：[图解 Apache Kafka 消息偏移量的演变 (0.7.x~0.10.x)](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650714529&idx=1&sn=e85eff6266ccac2d6532bb636c5b92c2&chksm=887dacd7bf0a25c1032e52c28b06d46c76de6dc290100f176b2ac8ebfb063f0c81efe83b4bf5&scene=21#wechat_redirect)

19、Kafka 高效文件存储设计特点  

-   Kafka 把 topic 中一个 parition 大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
-   通过索引信息可以快速定位 message 和确定 response 的最大大小。
-   通过 index 元数据全部映射到 memory，可以避免 segment file 的 IO 磁盘操作。
-   通过索引文件稀疏存储，可以大幅降低 index 文件元数据占用空间大小

20、Kafka 创建 Topic 时如何将分区放置到不同的 Broker 中  

-   副本因子不能大于 Broker 的个数；
-   第一个分区（编号为 0）的第一个副本放置位置是随机从 brokerList 选择的；
-   其他分区的第一个副本放置位置相对于第 0 个分区依次往后移。也就是如果我们有 5 个 Broker，5 个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个 Broker 上，依次类推；
-   剩余的副本相对于第一个副本放置位置其实是由 nextReplicaShift 决定的，而这个数也是随机产生的

具体可以参见 [Kafka 创建 Topic 时如何将分区放置到不同的 Broker 中](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650716375&idx=1&sn=4e1ac64aa6f7eaa5a210c3fb51977b7d&chksm=887da5a1bf0a2cb7855e99aecc0e003c3728c34dedc741c70ec9ca18345e541906b9c85c8b00&scene=21#wechat_redirect)。  

21、Kafka 新建的分区会在哪个目录下创建  

在启动 Kafka 集群之前，我们需要配置好 log.dirs 参数，其值是 Kafka 数据的存放目录，这个参数可以配置多个目录，目录之间使用逗号分隔，通常这些目录是分布在不同的磁盘上用于提高读写性能。

当然我们也可以配置 log.dir 参数，含义一样。只需要设置其中一个即可。

如果 log.dirs 参数只配置了一个目录，那么分配到各个 Broker 上的分区肯定只能在这个目录下创建文件夹用于存放数据。

但是如果 log.dirs 参数配置了多个目录，那么 Kafka 会在哪个文件夹中创建分区目录呢？答案是：Kafka 会在含有分区目录最少的文件夹中创建新的分区目录，分区目录名为 Topic 名 + 分区 ID。注意，是分区文件夹总数最少的目录，而不是磁盘使用量最少的目录！也就是说，如果你给 log.dirs 参数新增了一个新的磁盘，新的分区目录肯定是先在这个新的磁盘上创建直到这个新的磁盘目录拥有的分区目录不是最少为止。

具体可以参见我博客：[https://www.iteblog.com/archives/2231.html](https://www.iteblog.com/archives/2231.html)

22、谈一谈 Kafka 的再均衡  

在 Kafka 中，当有新消费者加入或者订阅的 topic 数发生变化时，会触发 Rebalance(再均衡：在同一个消费者组当中，分区的所有权从一个消费者转移到另外一个消费者) 机制，Rebalance 顾名思义就是重新均衡消费者消费。Rebalance 的过程如下：

第一步：所有成员都向 coordinator 发送请求，请求入组。一旦所有成员都发送了请求，coordinator 会从中选择一个 consumer 担任 leader 的角色，并把组成员信息以及订阅信息发给 leader。

第二步：leader 开始分配消费方案，指明具体哪个 consumer 负责消费哪些 topic 的哪些 partition。一旦完成分配，leader 会将这个方案发给 coordinator。coordinator 接收到分配方案之后会把方案发给各个 consumer，这样组内的所有成员就都知道自己应该消费哪些分区了。

所以对于 Rebalance 来说，Coordinator 起着至关重要的作用

23、谈谈 Kafka 分区分配策略

参见我这篇文章 [Kafka 分区分配策略 (Partition Assignment Strategy)](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650715861&idx=1&sn=03ff472f21fd429ac09559aca8f2b2bc&chksm=887daba3bf0a22b58e46e1c214f6b44592ada2d1b3e21137ead9d63ff86e41f094660252fd16&scene=21#wechat_redirect)

24、Kafka Producer 是如何动态感知主题分区数变化的？

[参见我这篇文章：Kafka Producer 是如何动态感知 Topic 分区数变化](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=403008823&idx=1&sn=442e1909d509f312d5057e4cfc478793&chksm=0d8670413af1f957ebdcea08cef0244447a5bd23bafd3a50efd326a949e4fa2142493476b497&scene=21#wechat_redirect)  

25、 Kafka 是如何实现高吞吐率的？  

Kafka 是分布式消息系统，需要处理海量的消息，Kafka 的设计是把所有的消息都写入速度低容量大的硬盘，以此来换取更强的存储能力，但实际上，使用硬盘并没有带来过多的性能损失。kafka 主要使用了以下几个方式实现了超高的吞吐率：

-   顺序读写；  
-   零拷贝
-   文件分段  
-   批量发送
-   数据压缩。

[具体参见：Kafka 是如何实现高吞吐率的](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=402790390&idx=1&sn=f0200f45e6697d703bf694fed7573fe1&chksm=0d810a803af68396857cf45264bead78e7f3b56cd8e49853bc798c50bc3ed4fea4440f52c901&scene=21#wechat_redirect)

26、Kafka 监控都有哪些？

[参见我另外几篇文章：Apache Kafka 监控之 KafkaOffsetMonitor](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=200581484&idx=1&sn=3e14a01cf1eb514dfdde73a6f032643d&chksm=1e77bc1a2900350cf546522ff9ce4a6387c0f797ba056ff18f19180c06ad6c8817e4ba337974&scene=21#wechat_redirect)

[雅虎开源的 Kafka 集群管理器 (Kafka Manager)](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=203369107&idx=1&sn=badcbedd5dd6bc1975307175d03e1a4f&chksm=199c37e52eebbef35106892a80b641394d6467478537ca0a0ff5a1ac605e336e3f18d45d2282&scene=21#wechat_redirect)

[Apache Kafka 监控之 Kafka Web Console](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=200584408&idx=1&sn=30a4ac3f68972bcfd8375708ba78de74&chksm=1e77b1ae290038b82bd8fbe9032ade23b7952ec8f8e07acd3d12dc29537ea0612b115956f471&scene=21#wechat_redirect)

还有 JMX  

27、如何为 Kafka 集群选择合适的 Topics/Partitions 数量

[](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650714873&idx=1&sn=56f9f9027f14e9dbaa4a2cfff6cf16e3&chksm=887daf8fbf0a26998802f4fd1b4c318c13ff2e2cb1150a6f966cf9432b04fb7b2b9b7dcca9c3&scene=21#wechat_redirect)[](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=200581484&idx=1&sn=3e14a01cf1eb514dfdde73a6f032643d&chksm=1e77bc1a2900350cf546522ff9ce4a6387c0f797ba056ff18f19180c06ad6c8817e4ba337974&scene=21#wechat_redirect)[参见我另外几篇文章](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=200581484&idx=1&sn=3e14a01cf1eb514dfdde73a6f032643d&chksm=1e77bc1a2900350cf546522ff9ce4a6387c0f797ba056ff18f19180c06ad6c8817e4ba337974&scene=21#wechat_redirect)：如何为 Kafka 集群选择合适的 Topics/Partitions 数量

28、谈谈你对 Kafka 事务的了解？  

参见这篇文章：[http://www.jasongj.com/kafka/transaction/](http://www.jasongj.com/kafka/transaction/)

29、谈谈你对 Kafka 幂等的了解？  

参见这篇文章：[https://www.jianshu.com/p/b1599f46229b](https://www.jianshu.com/p/b1599f46229b)

30、Kafka 缺点？  

-   由于是批量发送，数据并非真正的实时；
-   对于 mqtt 协议不支持；
-   不支持物联网传感数据直接接入；
-   仅支持统一分区内消息有序，无法实现全局消息有序；
-   监控不完善，需要安装插件；
-   依赖 zookeeper 进行元数据管理；

31、Kafka 新旧消费者的区别

旧的 Kafka 消费者 API 主要包括：SimpleConsumer（简单消费者） 和 ZookeeperConsumerConnectir（高级消费者）。SimpleConsumer 名字看起来是简单消费者，但是其实用起来很不简单，可以使用它从特定的分区和偏移量开始读取消息。高级消费者和现在新的消费者有点像，有消费者群组，有分区再均衡，不过它使用 ZK 来管理消费者群组，并不具备偏移量和再均衡的可操控性。

现在的消费者同时支持以上两种行为，所以为啥还用旧消费者 API 呢？

 ![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0ntKhPVwWAsmic9hgxmoLZJw0wCDuduXPmy15YzibhHAGmZbbwofEbO4Pkic98eN9RYTlvy2ts4FdpFtg/640?wx_fmt=png)

32、Kafka 分区数可以增加或减少吗？为什么？

我们可以使用 bin/kafka-topics.sh 命令对 Kafka 增加 Kafka 的分区数据，但是 Kafka 不支持减少分区数。

Kafka 分区数据不支持减少是由很多原因的，比如减少的分区其数据放到哪里去？是删除，还是保留？删除的话，那么这些没消费的消息不就丢了。如果保留这些消息如何放到其他分区里面？追加到其他分区后面的话那么就破坏了 Kafka 单个分区的有序性。如果要保证删除分区数据插入到其他分区保证有序性，那么实现起来逻辑就会非常复杂。  

本文参考

[https://blog.csdn.net/linke1183982890/article/details/83303003](https://blog.csdn.net/linke1183982890/article/details/83303003)

[https://www.cnblogs.com/FG123/p/10095125.html](https://www.cnblogs.com/FG123/p/10095125.html)

[https://www.cnblogs.com/chenmingjun/p/10480793.html](https://www.cnblogs.com/chenmingjun/p/10480793.html)

**推荐阅读**

**▬**

1.  [Apache Kafka 2.5 稳定版发布，新特性抢先看](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650719790&idx=1&sn=14b04c8180dbcb67dd600cbaaa42ec9b&chksm=887ddb58bf0a524ee82d7aa6c28e68912dda1baddf9ccd3f20fbc43e02b270132f83cfeaef1b&scene=21#wechat_redirect)  
2.  [实战 | 利用 Delta Lake 使 Spark SQL 支持跨表 CRUD 操作](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650719777&idx=1&sn=3f8cc019e5aa613c4bbbc417266a18ba&chksm=887ddb57bf0a5241571d858c5ec64e81ad8c7024c38275c826f5203e3c9605eb75e2ac3e3116&scene=21#wechat_redirect)  
3.  [滴滴 ElasticSearch 平台跨版本升级以及平台重构之路](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650719757&idx=1&sn=070ff98c8efd5bc5f48e6161d8600dcc&chksm=887ddb7bbf0a526da271d46faad8f6c45c4c184296fe5da75ee6f25874e31220a10d9c7f2017&scene=21#wechat_redirect)  
4.  [Apache Doris 在美团外卖数仓中的应用实践](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650719737&idx=2&sn=f2d099d8ba6dd5d89d94d455c98ca53c&chksm=887dd88fbf0a51990ac28ac894ce02d99eff03d704a66c778afdd47fda7e4c371830e9832446&scene=21#wechat_redirect)  

![](https://mmbiz.qpic.cn/mmbiz_png/0yBD9iarX0nvCib9EktAUIx7RAZiaHf5LnOQ98ygYib1fx8V5t5W3HMJ1IMvfiaTlAmpVOM2QAwoXvdiblFhlZ4MCa8A/640?wx_fmt=png)

过往记忆大数据微信群，请添加微信：fangzhen0219, 备注【进群】 
 [https://mp.weixin.qq.com/s/Fq3OYVi5mF2iJt9E3b-fkA](https://mp.weixin.qq.com/s/Fq3OYVi5mF2iJt9E3b-fkA)
