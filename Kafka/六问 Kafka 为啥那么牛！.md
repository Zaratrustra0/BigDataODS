# 六问 Kafka 为啥那么牛！
![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2wiaicw1TYjVUYVE7tOY6IxrZJCqq3QGrAyaFGLJMgWKicXNftQWcXeEjg/640?wx_fmt=png)

### 1.1 Kafka 概述

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib23YVVyVwPT0PaiaceyTSJJblrhsAqic33VSOrYqrHPJexY47KNRx7xhUg/640?wx_fmt=png)

Kafka 架构 (下方该是 Consumer)

Kafka 是一个**分布式**的基于**发布 / 订阅**模式的消息队列，依靠其强悍的吞吐量，Kafka 主要应用于大数据实时处理领域。在数据采集、传输、存储的过程中发挥着举足轻重的作用。

1.  Apache Kafka 由 **Scala** 写成，是由 Apache 软件基金会开发的一个开源消息系统项目。该项目的目标是为处理实时数据提供一个统一、高通量、低等待的平台。
2.  Kafka 是一个分布式消息队列，Kafka 对消息保存时根据 Topic 进行归类，Kafka 集群有多个 Kafka 实例组成，每个实例 Server 称为 broker。
3.  无论是 Kafka 集群还是 Consumer 都依赖于 ZooKeeper 集群保存一些 meta 信息，来保证系统可用性

### 1.2 Kafka 优点

1.  支持多个生产者和消费者。
2.  支持 broker 的横向拓展。
3.  副本集机制，实现数据冗余，保证数据不丢失。
4.  通过 topic 将数据进行分类。
5.  通过**分批**发送压缩数据的方式，减少数据传输开销，提高吞高量。
6.  支持多种模式的消息，消息是基于磁盘实现数据的持久化。
7.  高性能的处理信息，在大数据的情况下，可以保证亚秒级的消息延迟。
8.  一个消费者可以支持多种 topic 的消息。
9.  对 CPU、内存、网络的消耗比较小。
10. 支持跨数据中心的数据复制跟镜像集群。

### 1.3 Kafka 缺点

1.  由于是批量发送，所以数据达不到真正的实时。
2.  只能支持统一分区内消息有序，无法实现全局消息有序。
3.  监控不完善，需要安装插件。
4.  会丢失数据，并且不支持事务。
5.  可能会**重复消费**数据，消息会乱序，可用保证一个固定的 partition 内部的消息是有序的，但是一个 topic 有多个 partition 的话，就不能保证有序了，需要[zookeeper](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect)的支持，topic 一般需要人工创建，部署和维护一般都比 mq 高。

### 1.4 Kafka 架构

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2HnjfBvicic2zM0icgNTFqAcVR2RhNVYMeicxUSSiaZGKqtiab8RaBwxaKEpA/640?wx_fmt=png)

1.  **Broker**：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic。
2.  **Producer** ：消息生产者，就是向 Kafka broker 发消息的客户端。
3.  **Consumer** ：消息消费者，向 Kafka broker 拉取消息来消费。可以根据 Consumer 的消费能力以适当的速率消费消息。
4.  **Topic** ：可以理解为一个队列，生产者和消费者面向的都是一个 topic。
5.  **Partition**：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker 上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列，有点平衡分摊生产者机制。
6.  **Replication**：为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和若干个 follower。
7.  **leader**：一个分区有一个 Leader，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。
8.  **follower**：一个分区有一个 Follower，实时从 leader 中同步数据，保持和 leader 数据的同步。leader 发生故障时，某个 follower 会成为新的 follower。注意 Kafka 中 副本数不能超过 Broker 数！
9.  **Consumer Group** ：消费者组由多个 consumer 组成。组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
10. **offset**：消费者在具体消费某个 topic 中的消息时，可以指定起始偏移量。

### 1.5 ZooKeeper 作用

[ZooKeeper](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect) 在 Kafka 中有举足轻重的地位，一般提供如下功能：

##### 1.5.1 Broker 注册

Broker 是分布式部署并且相互之间相互独立，但是需要有一个注册系统能够将整个集群中的 Broker 管理起来，比如用 ZooKeeper。

##### 1.5.2 Topic 注册

在 Kafka 中同一个 Topic 的消息会被分成多个 Partition 并将其分布在多个 Broker 上，这些 Partition 信息及与 Broker 的对应关系也都是由 Zookeeper 在维护，由专门的节点来记录。

##### 1.5.3 生产者负载均衡

同一个 Topic 消息会被分区并将其分布在多个 Broker 上，因此，生产者需要将消息合理地发送到这些分布式的 Broker 上。

1.  老式的四层负载均衡，根据生产者的 IP 地址和端口来为其确定一个相关联的 Broker。一般一个生产者只会对应单个 Broker，但实际系统中的每个生产者产生的消息量及每个 Broker 的消息存储量都是不一样的。
2.  使用 [Zookeeper](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect) 进行负载均衡，由于每个 Broker 启动时，都会完成 Broker 注册过程，生产者会通过该节点的变化来动态地感知到 Broker 服务器列表的变更，这样就可以实现动态的负载均衡机制。

##### 1.5.4 消费者负载均衡

Kafka 中的消费者同样需要进行负载均衡来实现多个消费者合理地从对应的 Broker 服务器上接收消息，每个消费者分组包含若干消费者，`每条消息都只会发送给分组中的一个消费者`，不同的消费者分组消费自己特定的 Topic 下面的消息，互不干扰。

##### 1.5.5 分区 与 消费者 的关系

Kafka 会为每个 Consumer Group 分配个全局唯一 Group ID，Group 内的 Consumer 共享该 ID，Kafka 规定 每个 partition 信息只能被同组的一个 Consumer 消费，在 Zk 中记录 partition 跟 Consumer 关系，每个消费者一旦确定了对一个消息分区的消费权力，需要将其 Consumer ID 写入到 Zookeeper 对应消息分区的临时节点上。

##### 1.5.6 消息消费进度 Offset 记录

Consumer 对指定消息分区进行消费的过程中，需要定时地将分区消息的消费进度 Offset 记录到 Zookeeper 上，以便在该 Consumer 进行重启或者其他 Consumer 重新接管该消息分区的消息消费后，能够从之前的进度开始继续进行消息消费。

##### 1.5 7 消费者注册

为让同一个 Topic 下不同分区的消息尽量均衡地被多个 Consumer 消费而进行 Consumer 与消息分区分配的过程。

1.  Consumer 启动后在 ZK 下创建个节点，并且每个 Consumer 会对 Consumer Group 中的 Consumer 的变化注册监听，目的是为了保证 Consumer 负载均衡。
2.  Consumer 会对 Broker 列表监听，发生变化会进行 Consumer 负载均衡。

### 2.1 写入方式

producer 采用 push 模式将消息发布到 broker，每条消息都被 append 到 patition 中，属于 **顺序写磁盘** ，顺序写比随机写要起码提速 3 个数量级！

### 2.2 分区 Partition

##### 2.2.1  Partition 简介

消息发送时都被发送到一个 topic，其本质就是一个目录，而 topic 是由一些 分区日志 Partition Logs 组成，其组织结构如下图所示：  

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2ZEfFheVjT0Imiae5EFuGKgLOJPicpYo4Jo4Hic4BplxDdoyI9wzOqZgJQ/640?wx_fmt=png)

Partition 发生

可以看到每个 Partition 中的消息都是有序的，生产的消息被不断追加到 Partition log 上，其中的每一个消息都被赋予了一个唯一的 offset 值。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2Qp299ZwBia9ySXwriamNIG1dqrXdo4DyhxlAPzXiaT7VZ1q58PichZQEJQ/640?wx_fmt=png)

消费者

通过分区可以 方便在集群中扩展，可以提高并发。

形象理解：

> Kafka 的设计源自生活，好比为公路运输，不同的起始点和目的地需要修不同高速公路（主题），高速公路上可以提供多条车道（分区），流量大的公路 (主题) 多修几条车道 (分区) 保证畅通，流量小的公路少修几条车道避免浪费。收费站好比消费者，车多的时候多开几个一起收费避免堵在路上，车少的时候开几个让汽车并道就好了。

##### 2.2.2 分区原则

我们需要将 producer 发送的数据封装成一个 ProducerRecord 对象。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2DzGmZWvZDgXrcLtYCc2ib1gQ0TTVBn3ls3nlLmykkzO2iaicDBNvgN93w/640?wx_fmt=png)

数据封装

1.  指明 partition 的情况下，直接将指明的值直接作为 partiton 值。
2.  没有指明 partition 值但有 key 的情况下，将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值。
3.  既没有 partition 值又没有 key 值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与 topic 可用的 partition 总数取余得到 partition 值，也就是常说的 **round-robin** 算法。

### 2.3 Kafka 文件存储机制

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2jQIPicEW14wibdjZ9Je3iboeS5uogUDYkfyL5YmOwuIzmKibo5VSMYufqg/640?wx_fmt=png)

Kafka 存储结构

1.  Kafka 中消息是以 topic 进行分类的，生产者跟消费者都是面向 topic 的，topic 只是逻辑上的概念，而 Partition 是物理上的概念，每个 Partition 对应个 log 文件，每个分区用 ``_.index` 存放数据索引，`_.log``存储数据。index 文件中的元数据指向对应 log 文件中 Message 的物理偏移地址 (参考 **kaldi**、**Neo4j**)。
2.  为防止 log 文件过大导致数据定位效率低下，Kafka 采取了`分片`和`索引`机制，将每个 partition 分为多个**segment**。每个 segment 对应``_.index` 跟 `_.log``。这些文件位于一个文件夹下，该文件夹的命名规则为：topic 名称 + 分区序号。例如 first 这个 topic 有三个分区，则其对应的文件夹为 first-0、first-1、first-2。


    100000000000000000000.index200000000000000000000.log300000000000000170410.index400000000000000170410.log500000000000000239430.index600000000000000239430.log

注意：index 和 log 文件以当前 segment 的第一条消息的 offset 命名。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib22yyR8Xtvb0eeuy2aNLLqUc3J6CwoWLLqm1a6IvPEp4wQBcvUrwMaNg/640?wx_fmt=png)

数据查找过程

### 2.4 如何保证消息顺序执行

##### 2.4.1 顺序错乱

1.  Kafka 一个 topic，一个 partition，一个 Consumer，但是 Consumer 内部进行多线程消费，这样数据也会出现顺序错乱问题。

    ![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2zf1jn5BIn8Sz3m5IM6QrYwxKXyoiaiaLfVP3Nkk6VyOWibVnmbplN6Xibw/640?wx_fmt=png)

    多线程消费
2.  数据有顺序的数据写入到了不同的 partition 里面，不同的消费者去消费，但是每个 Consumer 的执行时间是不固定的，无法保证先读到消息的 Consumer 一定先完成操作，这样就会出现消息并没有按照顺序执行，造成数据顺序错误。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2kAImP1VD0JBhlUXECM2DrBNcqsFIZCRSDFLWQjS6IJILnnGfcM5xcA/640?wx_fmt=png)

多个消费者

##### 2.4.2 解决办法

1.  确保同一个消息发送到同一个 partition，一个 topic，一个 partition，一个 consumer，内部单线程消费。

> 1.  写入同一个 Partition 的信息一定有序。
> 2.  给信息指定一个 key，key 相同则一定写入同一个 partition。
> 3.  从同一个 Partition 读取信息一定有序。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2TH92dKGErMRaic4ib3bRYFqgEibNibZs0uQubAYGuK7BupOORItQbaCINQ/640?wx_fmt=png)

单线程消费

2.  在 1 的基础上，在一个 Consumer 上根据信息 ID 映射到不同队列，以此加速消费。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib28svrB2EAWIia6vHibacUwYib35iaicgjG2JZ2KAD0GZFy5Ppa4hzzVZviayA/640?wx_fmt=png)

内存队列

### 4.1 消息传递语义

消息传递语义 message delivery semantic ，Kafka 为确保消息在 producer 和 consumer 之间传输。有以下三种传输保障（delivery guarantee）：

1.  **at most once**：最多一次，消息可能会丢，但绝不会重复传输。
2.  **at least once**：至少一次，消息绝不会丢，但可能会重复传输。
3.  **exactly once**：精确传递一次。消息被处理且只会被处理一次。不丢失不重复就一次。

理想情况下肯定希望系统的消息传递是严格 exactly once，但很难做到。接下来会按照 消息的传播流程大致说下。

### 4.2 信息从生产者到 Broker

##### 4.2.1 生产者信息发送至 Broker

大致步骤如下：

1.  producer 从 ZK 找到目标 Partition 的 Leader 元数据。
2.  producer 发送消息给 Leader。
3.  Leader 接受消息持久化，然后根据 acks 配置选择如何同步 Follower。
4.  Followder 按照前面说的同步数据后给 Leader 回复 ack。
5.  Leader 跟 Follower 同步完毕后 Leader 给 producer 回复 ack。

对于 Leader 回复 ack，Kafka 为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡。

1.  request.required.acks = 0

> 1.  producer`不等待` broker 的 ack，提供了一个最低的延迟，broker 接收到还没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据，对应 At Most Once 模式。
> 2.  但凡没落盘成功信息就丢失了，一般生产不用。

2.  request.required.acks  = 1

> 1.  此乃默认值，producer 等待 broker 的 ack，partition 的`leader 落盘成功后返回 ack`，如果在 follower 同步成功之前 leader 故障，那么将会丢失数据；认为 leader 返回 信息就成功了。

3.  request.required.acks = -1 / all

> 1.  producer 等待 broker 的 ack，partition 的 leader 和 follower (ISR 中的)`全部落盘`成功后才返回 ack。
> 2.  但如果在 leader 收到信息返回 ok，follower 收到信息但是发送 ack 时 leader 故障，此时生产者会`重新`给 follower 发送个信息。
> 3.  对应 At Least Once 模式。

##### 4.2.2 如何保证幂等性

如果业务需要数据  Exactly Once，在早期的 Kafka 版本中 只能在下游去重，现在引入了个幂等性，意思就是无论生产者发送多少个重复消息，Server 端只会持久化一条数据，

> At Least Once + 幂等性 = Exactly Once

启动幂等性，在生产者参数中 enable.idompotence= true，开启幂等性的生产者在初始化时候会被分配一个 PID，发送同一个 Partition 的消息会附带 Sequence Number，Broker 会对做缓存，以此来判断唯一性。但是如果 PID 重启就会发生变化，同时不同 partition 也具有不同的主键，幂等性无法保证跨分区会话的 Exactly Once。

### 4.3 Kafka Broker 信息落磁盘

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2yxe7t14XIOticE3mtcSSAgYxdzIDEye6WXibrruib8uRTomwTYHK0tkrA/640?wx_fmt=png)

数据落盘过程

Kafka Broker 收到信息后，如何落盘是通过 producer.type 来设定的，一般两个值。

1.  sync，默认模式，数据必须最终落盘才算 OK。
2.  async，异步模式，数据刷新到 OS 的 Page Cache 就返回，此时如果机器突然出问题，信息就丢失了。

### 4.4 消费者从 Kafka Broker 消费数据

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2wTh3I69HiaX3pJB8DVWGZR7wWRoJJFwPy2syhoQc40l6dohibcjicUmNQ/640?wx_fmt=png)

消费数据

Consumer 是以 Consumer Group 消费者组的方式工作，由一个或者多个消费者组成一个组，共同消费一个 topic。每个分区在同一时间只能由 group 中的一个消费者读取，但是多个 group 可以同时消费这个 partition。如果一个消费者失败了，那么其他的 group 成员会自动负载均衡读取之前失败的消费者读取的分区。Consumer Group 从 Broker 拉取消息来消费主要分为两个阶段：

1.  获得数据，提交 Offset。
2.  开始处理数据。

如果你先提交 offset 再处理数据可能在处理数据时出现异常导致数据丢失。而如果你先处理数据再提交 offset， 如果提交 offset 失败可能导致信息重复消费。

PS：

> pull 模式不足之处是，如果 kafka 没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，Kafka 的消费者在消费数据时会传入一个时长参数 timeout，如果当前没有数据可供消费，consumer 会等待一段时间之后再返回，这段时长即为 timeout。

同一个 group.id 中的消费者，对于一个 topic 中的多个 partition 中的消息消费，存在着一定的分区分配策略。

在 Kafka 中存在着两种分区分配策略，通过 partition.assignment.strategy 来设置。

1.  **RangeAssignor** 范围分区策略，也是默认模式。
2.  **RoundRobinAssignor** 分配策略，轮询分区模式。

### 5.1  RangeAssignor 范围分区策略

Range 范围分区策略是对每个 topic 而言的。首先对同一个 topic 里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。假如现在有 10 个分区，3 个消费者，排序后的分区将会是 p0~p9。消费者排序完之后将会是 C1-0、C2-0、C3-0。通过 Partitions 数 / Consumer 数 来决定每个消费者应该消费几个分区。如果除不尽，那么前面几个消费者将会多消费 1 个分区。

| 消费者  | 消费的分区          |
| ---- | -------------- |
| C1-0 | 消费 p0、1、2、3 分区 |
| C2-0 | 消费 4、5、6 分区    |
| C3-0 | 消费 7、8、9 分区    |

**Range 范围分区的弊端**：

如上只是针对 1 个 topic 而言，C1-0 消费者多消费 1 个分区影响不是很大。如果有 N 多个 topic，那么针对每个 topic，消费者 C1-0 都将多消费 1 个分区，topic 越多，C1-0 消费的分区会比其他消费者明显多消费 N 个分区。这就是 Range 范围分区的一个很明显的弊端了.

### 5.2  RoundRobinAssignor 轮询分区策略

RoundRobin 轮询分区策略是把所有的 partition 和所有的 consumer 都列出来，然后按照 hascode 进行排序，最后通过轮询算法来分配 partition 给到各个消费者。轮询分区分为如下两种情况：

1.  同一个 Consumer Group 内 Consumer  订阅信息相同
2.  同一个 Consumer Group 内 Consumer  订阅信息不相同

##### 5.2.1 Consumer Group 内 Consumer  订阅信息相同

如果同一消费组内，所有的消费者订阅的消息都是相同的，那么 RoundRobin 策略的分区分配会是均匀的。

例如同一消费者组中，有 3 个消费者 C0、C1 和 C2，都订阅了 2 个主题 t0 和 t1，并且每个主题都有 3 个分区 (p0、p1、p2)，那么所订阅的所以分区可以标识为 t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终分区分配结果如下：

| 消费者 | 消费的分区           |
| --- | --------------- |
| C0  | 消费 t0p0、t1p0 分区 |
| C1  | 消费 t0p1、t1p1 分区 |
| C2  | 消费 t0p2、t1p2 分区 |

##### 5.2.1 Consumer Group 内 Consumer  订阅信息不相同

同一消费者组内，所订阅的消息是不相同的，那么分区分配就不是完全的轮询分配，有可能会导致分区分配的不均匀。如果某个消费者没有订阅消费组内的某个 topic，那么在分配分区的时候，此消费者将不会分配到这个 topic 的任何分区。

例如同一消费者组中有 3 个消费者 C0、C1、C2，他们共订阅了 3 个主题 t0、t1、t2，这 3 个主题分别有 1、2、3 个分区 (即 t0 有 1 个分区 (p0)，t1 有 2 个分区 (p0、p1)，t2 有 3 个分区 (p0、p1、p2))，即整个消费者所订阅的所有分区可以标识为 t0p0、t1p0、t1p1、t2p0、t2p1、t2p2。然后消费者 C0 订阅的是主题 t0，消费者 C1 订阅的是主题 t0 和 t1，消费者 C2 订阅的是主题 t0、t1 和 t2，最终分区分配结果如下：

| 消费者 | 消费的分区                        |
| --- | ---------------------------- |
| C0  | 消费 t0p0 分区                   |
| C1  | 消费 t1p0 分区                   |
| C2  | 消费 t1p1、 t2p0、 t2p1、 t2p2 分区 |

Kafka 可支持百万 TPS 跟如下几个特性有关。

### 6.1 顺序读写数据

信息存储在硬盘中，硬盘由很多盘片组成，显微镜观察盘片会看见盘片表面凹凸不平，凸起的地方被磁化代表数字 1，凹的地方是没有被磁化代表数字 0，因此硬盘可以以二进制来存储表示文字、图片等信息。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2cibvFeJy9J1BxhXN5w0ribsaz3m1NM6lia3J7CjA8P02Dt4MorD0q5N8Q/640?wx_fmt=png)

磁盘平面图

上图是[硬盘](https://mp.weixin.qq.com/s?__biz=MzkxNTE3NjQ3MA==&mid=2247487894&idx=1&sn=750822f3d5dc8a53cea3546142db848e&scene=21#wechat_redirect)的实际图，可能无法理解内部构造，我们来看个形象的图：

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2WxMbaCuffWOUWib0R9Mmxw9j9aB0vSUlRVDdJ41mekh52VTDy0tMqnQ/640?wx_fmt=png)

磁盘内部图

1.  系统通过`磁头`从`盘面`读取数据，磁头在盘面上的飞行高度只是人类头发直径的千分之一。
2.  硬盘里的`盘片`跟 CD 光盘的长相类似，一个`盘片`有上下两个`盘面`，每个`盘面`都可以存储数据。
3.  每个`盘面`会被划分出超级多的同心圆`磁道`，同心圆的半径是不同的。
4.  所有磁盘上的同一磁道构成一个`柱面`，相同`磁道`的同一个`扇区`被称为`簇`。数据的读写按照`柱面`从上到下进行，一个柱面写满后，才移到下一个扇区开始写数据。
5.  一个磁道是被划分成一段段的`圆弧`(`扇区`)，每个`扇区`用来存储 512 个字节跟其他信息。由于同心圆的扇区弧度相同而半径不同所以外圈线速度比内圈线速度大。
6.  系统每次读取一个扇区效率太低，所以操作系统是按照`block`来进行读取数据的，一个`block`(块) 一般有多个扇区组成。在每块的大小是 4~64KB。
7.  页`page`，默认 4KB，操作系统经常与内存和硬盘这两种存储设备进行通信，类似于块的概念，都需要一种虚拟的基本单位。所以与内存操作，是虚拟一个页的概念来作为最小单位。与硬盘打交道，就是以块为最小单位。

> 1.  **扇区**：硬盘的最小读写单元
> 2.  **块 / 簇**：是操作系统针对硬盘读写的最小单元
> 3.  **page**：是内存与操作系统之间操作的最小单元。

一次访盘的读 / 写请求完成过程由三个动作组成：

1.  **寻道**：磁头从开始移动到数据所在磁道所需要的时间，平均 10ms 左右。
2.  **旋转延迟**：盘片旋转将请求数据所在扇区移至读写磁头下方所需要的时间，旋转延迟取决于磁盘转速。如果是 5400 转每分钟的磁盘，平均大概为 5 ms。
3.  **数据传输**：磁头位从目标扇区第一个位置，到访问完所有数据的耗时。假如 5400 转的磁道有 400 个扇区，我只访问一个则耗时 0.0278ms。

可以发现读取主要耗时是在前两个，如果我顺序读取则`寻道`跟`旋转延迟`只用一次即可。而如果随机读取呢则可能经历多次`寻道`跟`旋转延迟`，两者相差几乎 **3**个数量级。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2wnj3dva6kiaj0kov5RxL6YpmAUX3WMZDdCnInHiacWNcUEwiapCThz2TA/640?wx_fmt=png)

随机跟顺序读写在磁盘跟内存中

### 6.2  Memory Mapped Files 内存映射文件

1.  虚拟内存系统 通过将虚拟内存分割为称作虚拟页 (Virtual Page，VP) 大小固定的块，一般情况下，每个虚拟页的大小默认是 4KB。同样的，物理内存也被分割为物理页(Physical Page，PP)，也为 4KB。
2.  服务器可直接用 操作系统的 Page 来实现物理内存到文件的映射，用户操作读写数据会直接到 Page 中，操作系统会根据映射自动的将对物理内存的操作同步到硬盘上。实现类似**顺序读写内存**的功能。
3.  缺点在 Broker 信息落盘时候也说了，落的不是真正磁盘可能导致**数据丢失**。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib21gwG11EK5HPGp6vPfszlgB7iaYMWZPvpNwSx1XwuZ159mbDibLmWcx3g/640?wx_fmt=png)

内存映射

### 6.3 Zero Copy

##### 6.3.1  直接内存存取 DMA

CPU 发出指令操作 IO 来进行读写操作，大部分情况下其实只是把数据读取到内存，然后从内存传到 IO 即可，所以数据其实可以不经过 CPU 的。

`Direct Memory Access` 的出现就是为批量数据的输入 / 输出而提速的。DMA 是指外部设备不通过 CPU 而直接与系统内存交换数据的接口技术。这样数据的传送速度就取决于存储器和外设的工作速度。

如果数据传输的时候只用到了 DMA 传输而没经过 CPU 复制数据，则我们称之为零拷贝 Zero Copy。用了 Zero Copy 技术耗时性能起码减半。

##### 6.3.2 Kafka 读写对比

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUHrF3xFGlAkMVmLdhklhib2QvcSzZGjNTS6KGKFBDQxYHJyRudY8Ov1ZynpicHsrvPrmIpjtGkfuyQ/640?wx_fmt=png)

零拷贝

如上黑色流程是没用 `Zero Copy` 技术流程：

1.  DMA 传输，磁盘读取数据到操作系统内存 Page Cache 区。
2.  CPU 搬运，数据从 Page Cache 区数据复制到用户内存区。
3.  CPU 搬运，数据从用户内存区到  Socket Cache 区。
4.  DMA 传输，数据从 Socket Cache 区传输到 NIC 网卡缓存区。

红色流程是用 `Zero Copy` 技术流程：

1.  DMA 传输，磁盘读取数据到操作系统内存 Page Cache 区。
2.  DMA 传输，数据从 系统内存 Page Cache 区传输到 NIC 网卡缓存区。

### 6.4 Batch Deal

消费者拉取数据的时候，Kafka 不是一个一个的来送数据的，而是批量发送来处理的，这样可以节省网络传输，增大系统的 TPS，不过也有个缺点就是，我们的数据不是真正的实时处理的，而真正的实时还是要看 `Flink`。

> 1.  Kafka 为什么要分区 ：[https://www.zhihu.com/question/28925721](https://www.zhihu.com/question/28925721)
> 2.  关于磁盘读取：[https://blog.csdn.net/holybin/article/details/21175781](https://blog.csdn.net/holybin/article/details/21175781)
> 3.  Kafka 百万 TPS：[https://mp.weixin.qq.com/s/Fb1cW0oN7xYeb1oI2ixtgQ](https://mp.weixin.qq.com/s/Fb1cW0oN7xYeb1oI2ixtgQ)

* * *

\[

RabbitMQ 高频考点

2021-01-27

![](https://mmbiz.qpic.cn/mmbiz_jpg/wJvXicD0z2dWaq3xqm2lcKtvTxfBPgAdENmDzvyjHJ14WjbQPeZgzJqgVpz9BpNBTW6fVoOaL1mugtI3XJuhqcw/640?wx_fmt=jpeg)

]([http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247490325&idx=1&sn=ab7cfedc7b8f2361cc1fab9418b314b8&chksm=ebdefa2ddca9733b76cffbcba2d5f8c0c61bd36d5ebba20aaecb07148aa0fd7a5781e16ab03e&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247490325&idx=1&sn=ab7cfedc7b8f2361cc1fab9418b314b8&chksm=ebdefa2ddca9733b76cffbcba2d5f8c0c61bd36d5ebba20aaecb07148aa0fd7a5781e16ab03e&scene=21#wechat_redirect))

\[

Hadoop 超燃之路

2021-01-25

![](https://mmbiz.qpic.cn/mmbiz_jpg/wJvXicD0z2dUVOoVjXO8NxclHAJnVXItbQbiampfndueDvr2pAFzRQE55LN7qkcWpbKxmJ5REm8yGjlMMx07Eq4Q/640?wx_fmt=jpeg)

]([http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247490316&idx=1&sn=1fc31b36b525f92c368dad6fbbf56434&chksm=ebdefa34dca9732268c2a1600161e6ea07667f9b3bbccfb02648c89cae3df1c0b00e7732cc24&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247490316&idx=1&sn=1fc31b36b525f92c368dad6fbbf56434&chksm=ebdefa34dca9732268c2a1600161e6ea07667f9b3bbccfb02648c89cae3df1c0b00e7732cc24&scene=21#wechat_redirect))

\[

20 张图带你到 HBase 的世界遨游

2021-01-20

![](https://mmbiz.qpic.cn/mmbiz_jpg/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKm5Y7OYBC1fuibFF0cfiaz0rpEgCgsia4PyzeVRTlxBzw92AkdiaE7PicWIw/640?wx_fmt=jpeg)

]([http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247490034&idx=1&sn=b8624b8d13de9aacd2b69891b5a62379&chksm=ebdef8cadca971dc95f07681a3be03281e0b9350595e38713944ec32159979995eaf9dd5cae2&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247490034&idx=1&sn=b8624b8d13de9aacd2b69891b5a62379&chksm=ebdef8cadca971dc95f07681a3be03281e0b9350595e38713944ec32159979995eaf9dd5cae2&scene=21#wechat_redirect))

\[

讲解 Zookeeper 的五个核心知识点

2021-01-15

![](https://mmbiz.qpic.cn/mmbiz_jpg/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianiaOnGyTD1B41LSJEzdSeickeyBMpRriaicy5WmfnDLkjzSIadTNeyVY8lg/640?wx_fmt=jpeg)

]([http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect))

\[

浅谈大数据中的 2PC、3PC、Paxos、Raft、ZAB

2020-11-03

![](https://mmbiz.qpic.cn/mmbiz_jpg/wJvXicD0z2dUyAByrX07E8icI6ichu8DK2cZN6rmwE4TibO9G7Ja2lic7E3iafKibBicopgSFOGOTLY6fdW7s49FoWmTdA/640?wx_fmt=jpeg)

]([http://mp.weixin.qq.com/s?\_\_biz=MzI4NjI1OTI4Nw==&mid=2247485515&idx=1&sn=60763ddda77928943bfd3d57e0c9256e&chksm=ebdee973dca9606535b5bc8f3ee4ab4b8d3ae4167ba3aa523514cd9f092710e183d0f259ccf5&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485515&idx=1&sn=60763ddda77928943bfd3d57e0c9256e&chksm=ebdee973dca9606535b5bc8f3ee4ab4b8d3ae4167ba3aa523514cd9f092710e183d0f259ccf5&scene=21#wechat_redirect))

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMeonKPbsia8sChuc9pNHetEGGtFFDL6VD6fBHno15D2NNI4iaeywAtymQ/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/st-7k7WH5pvLZwA_o9jPpw](https://mp.weixin.qq.com/s/st-7k7WH5pvLZwA_o9jPpw)
