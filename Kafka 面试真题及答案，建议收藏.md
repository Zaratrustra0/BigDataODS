# Kafka 面试真题及答案，建议收藏
Kafka 可以说是必知必会的了，首先面试大数据岗位的时候必问 kafka，甚至现在 java 开发岗位也会问到 kafka 一些消息队列相关的知识点。先来看看有哪些最新的 Kafka 相关面试点：

一、基础摸底

1.1、你们 Kafka 集群的硬盘一共多大？有多少台机器？日志保存多久？用什么监控的？

1.2、Kafka 分区数、副本数和 topic 数量多少比较合适？

1.3、Kafka 中的 HW、LEO、ISR、AR 分别是什么意思？

1.4、Kafka 中的消息有序吗？怎么实现的？

1.5、topic 的分区数可以增加或减少吗？为什么？

1.6、你知道 kafka 是怎么维护 offset 的吗？

1.7、你们是怎么对 Kafka 进行压测的？

二、感觉还不错，接着深入考察

2.1、创建或者删除 topic 时，Kafka 底层执行了哪些逻辑？

2.2、你了解 Kafka 的日志目录结构吗？

2.3、Kafka 中需要用到选举吗？对应选举策略是什么？

2.4、追问，聊聊你对 ISR 的了解？

2.5、聊聊 Kafka 分区分配策略？

2.6、当 Kafka 消息数据出现了积压，应该怎么处理？

2.7、Kafka 是怎么实现 Exactly Once 的？

2.8、追问、谈谈你对 Kafka 幂等性的理解？

2.9、你对 Kafka 事务了解多少？

2.10、Kafka 怎么实现如此高的读写效率？

三、侃侃而谈

3.1、说说你常用的 broker 参数优化？

3.2、那怎么进行 producer 优化呢？

**总结最准确的答案如下：** 

一、基础摸底

**1.1、你们 Kafka 集群的硬盘一共多大？有多少台机器？日志保存多久？用什么监控的？**

这里考察应试者对 kafka 实际生产部署的能力，也是为了验证能力的真实程度，如果这个都答不好，那可能就不会再继续下去了。

一般企业判断这些指标有一个标准：

集群硬盘大小：每天的数据量 / 70%\* 日志保存天数；

机器数量：Kafka 机器数量 = 2_（峰值生产速度_副本数 / 100）+1；

日志保存时间：可以回答保存 7 天；

监控 Kafka：一般公司有自己开发的监控器，或者 cdh 配套的监控器，另外还有一些开源的监控器：kafkaeagle、KafkaMonitor、KafkaManager。

**1.2、Kafka 分区数、副本数和 topic 数量多少比较合适？**

首先要知道**分区数并不是越多越好**，一般分区数不要超过集群机器数量。分区数越多占用内存越大 （ISR 等），一个节点集中的分区也就越多，当它宕机的时候，对系统的影响也就越大。

分区数一般设置为：3-10 个。

副本数一般设置为：2-3 个。

topic 数量需要根据日志类型来定，一般有多少个日志类型就定多少个 topic，不过也有对日志类型进行合并的。

**1.3、Kafka 中的 HW、LEO、ISR、AR 分别是什么意思？**

LEO：每个副本的最后一条消息的 offset

HW：一个分区中所有副本最小的 offset

ISR：与 leader 保持同步的 follower 集合

AR：分区的所有副本

**1.4、Kafka 中的消息有序吗？怎么实现的？**

kafka 无法保证整个 topic 多个分区有序，但是由于每个分区（partition）内，每条消息都有一个 offset，故**可以保证分区内有序**。

**1.5、topic 的分区数可以增加或减少吗？为什么？**

topic 的分区数只能增加不能减少，因为减少掉的分区也就是被删除的分区的数据难以处理。

增加 topic 命令如下：

```sql
bin/kafka-topics.sh 

```

关于 topic 还有一个面试点要知道：消费者组中的消费者个数如果超过 topic 的分区，那么就会有消费者消费不到数据。

**1.6、你知道 kafka 是怎么维护 offset 的吗？**

1\.**维护 offset 的原因**：由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。

2. **维护 offset 的方式 \*\***：\*\* Kafka 0.9 版本之前，consumer 默认将 offset 保存在 Zookeeper 中，从 0.9 版本开始，consumer 默认将 offset 保存在 Kafka 一个内置的 topic 中，该 topic 为\*\*\_\_consumer_offsets\*\*。

3\.**需要掌握的关于 offset 的常识**：消费者提交消费位移时提交的是当前消费到的最新消息的 offset+1 而不是 offset。

**1.7、你们是怎么对 Kafka 进行压测的？**

Kafka 官方自带了压力测试脚本（kafka-consumer-perf-test.sh、kafka-producer-perf-test.sh）， Kafka 压测时，可以查看到哪个地方出现了瓶颈（CPU，内存，网络 IO），一般都是网络 IO 达到瓶颈。

二、感觉还不错，接着深入考察

**2.1、创建或者删除 topic 时，Kafka 底层执行了哪些逻辑？**

以创建 topic 为例，比如我们执行如下命令，创建了一个叫做 csdn 的分区数为 1，副本数为 3 的 topic。

```sql
bin/kafka-topics.sh 

```

这行命令，在 kafka 底层需要经过三个步骤来处理：

1. 在 zookeeper 中的 / brokers/topics 节点下创建一个新的 topic 节点，如：/brokers/topics/csdn；

2. 然后会触发 Controller 的监听程序；

3. 最后 kafka Controller 负责 topic 的创建工作，并更新 metadata cache，到这里 topic 创建完成。

**2.2、你了解 Kafka 的日志目录结构吗？**

1. 每个 Topic 都可以分为一个或多个 Partition，Topic 其实是比较抽象的概念，但是 Partition 是比较具体的东西；

2. 其实 Partition 在服务器上的表现形式就是一个一个的文件夹，由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 segment；

3. 每组 Segment 文件又包含 .index 文件、.log 文件、.timeindex 文件（早期版本中没有）三个文件。.log 和. index 文件位于一个文件夹下，该文件夹的命名规则为：topic 名称 + 分区序号。例如，csdn 这个 topic 有 2 个分区，则其对应的文件夹为 csdn-0,csdn-1；

4. log 文件就是实际存储 Message 的地方，而 index 和 timeindex 文件为索引文件，用于检索消息

**2.3、Kafka 中需要用到选举吗？对应选举策略是什么？**

一共有两处需要用到选举，首先是 partition 的 leader，用到的选举策略是 ISR；然后是 kafka Controller，用先到先得的选举策略。

**2.4、追问，聊聊你对 ISR 的了解？**

ISR 就是 kafka 的副本同步队列，全称是 In-Sync Replicas。ISR 中包括 Leader 和 Follower。如果 Leader 进程挂掉，会在 ISR 队列中选择一个服务作为新的 Leader。有 replica.lag.max.messages（延 迟条数）和 replica.lag.time.max.ms（延迟时间）两个参数决定一台服务是否可以加入 ISR 副 本队列，在 0.10 版本移除了 replica.lag.max.messages 参数，防止服务频繁的进去队列。

任意一个维度超过阈值都会把 Follower 剔除出 ISR，存入 OSR（Outof-Sync Replicas） 列表，新加入的 Follower 也会先存放在 OSR 中。

**2.5、聊聊 Kafka 分区分配策略？**

在 Kafka 内部存在三种默认的分区分配策略：Range ， RoundRobin 以及 0.11.x 版本引入的 Sticky。Range 是默认策略。Range 是对每个 Topic 而言的（即一个 Topic 一个 Topic 分），首先 对同一个 Topic 里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。然后用 Partitions 分区的个数除以消费者线程的总数来决定每个消费者线程消费几个分区。如果除不尽，那么前面几个消费者线程将会多消费一个分区。

三种分区分配策略详见文章：深入分析 Kafka 架构（三）：消费者消费方式、分区分配策略（Range 分配策略、RoundRobin 分配策略、Sticky 分配策略）、offset 维护

文中对三种分区分配策略举例并进行了非常详细的对比，值得一看。

**2.6、当 Kafka 消息数据出现了积压，应该怎么处理？**

数据积压主要可以从两个角度去分析：

1. 如果是 Kafka 消费能力不足，则可以考虑增加 Topic 的分区数，并且同时提升消费 组的消费者数量，消费者数 = 分区数。（两者缺一不可）

2. 如果是下游的数据处理不及时：提高每批次拉取的数量。如果是因为批次拉取数据过少（拉取 数据 / 处理时间 &lt; 生产速度），也会使处理的数据小于生产的数据，造成数据积压。

**2.7、Kafka 是怎么实现 Exactly Once 的？**

在实际情况下，我们对于某些比较重要的消息，需要保证 exactly once 语义，也就是保证每条消息被发送且仅被发送一次，不能重复。在 0.11 版本之后，Kafka 引入了幂等性机制（idempotent），配合 acks = -1 时的 at least once 语义，实现了 producer 到 broker 的 exactly once 语义。

**idempotent + at least once = exactly once**

使用时，只需将 enable.idempotence 属性设置为 true，kafka 自动将 acks 属性设为 - 1。

**2.8、追问、谈谈你对 Kafka 幂等性的理解？**

Producer 的幂等性指的是当发送同一条消息时，数据在 Server 端只会被持久化一次，数据不丟不重，但是这里的幂等性是有条件的：

1. 只能保证 Producer 在单个会话内不丟不重，如果 Producer 出现意外挂掉再重启是 无法保证的。**因为幂等性情况下，是无法获取之前的状态信息，\*\***因此是无法做到跨会话级别的不丢不重 \*\*。

2. 幂等性不能跨多个 Topic-Partition，只能保证单个 Partition 内的幂等性，当涉及多个 Topic-Partition 时，这中间的状态并没有同步。

**2.9、你对 Kafka 事务了解多少？**

Kafka 是在 0.11 版本开始引入了事务支持。事务可以保证 Kafka 在 Exactly Once 语义的基 础上，生产和消费可以跨分区和会话，要么全部成功，要么全部失败。

1. Producer 事务：

为了实现跨分区跨会话的事务，需要引入一个全局唯一的 Transaction ID，并将 Producer 获得的 PID 和 Transaction ID 绑定。这样当 Producer 重启后就可以通过正在进行的 Transaction ID 获得原来的 PID。

为了管理 Transaction，Kafka 引入了一个新的组件 Transaction Coordinator。Producer 就 是通过和 Transaction Coordinator 交互获得 Transaction ID 对应的任务状态。Transaction Coordinator 还负责将事务所有写入 Kafka 的一个内部 Topic，这样即使整个服务重启，由于 事务状态得到保存，进行中的事务状态可以得到恢复，从而继续进行。

2. Consumer 事务：

上述事务机制主要是从 Producer 方面考虑，对于 Consumer 而言，事务的保证就会相对较弱，尤其时无法保证 Commit 的信息被精确消费。这是由于 Consumer 可以通过 offset 访问任意信息，而且不同的 Segment File 生命周期不同，同一事务的消息可能会出现重启后被删除的情况。

**2.10、Kafka 怎么实现如此高的读写效率？**

1. 首先 kafka 本身是分布式集群，同时采用了分区技术，具有较高的并发度；

2. 顺序写入磁盘，Kafka 的 producer 生产数据，要写入到 log 文件中，写的过程是一直追加到文件末端，为顺序写。

官网有数据表明，同样的磁盘，顺序写能到 600M/s，而随机写只有 100K/s。这 与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间。

3. 零拷贝技术

三、侃侃而谈

**3.1、说说你常用的 broker 参数优化？**

1. 网络和 IO 操作线程配置优化

```apache

num.network.threads=cpu 核数+1

num.io.threads=cpu 核数*2
```

2. log 数据文件策略

```apache

log.flush.interval.ms=1000
```

3. 日志保存策略

```cpp
# 保留三天，也可以更短 （log.cleaner.delete.retention.ms）
log.retention.hours=72
```

4. replica 相关配置

```css
offsets.topic.replication.factor:3
# 这个参数指新创建一个 topic 时，默认的 Replica 数量,Replica 过少会影响数据的可用性，太多则会白白浪费存储资源，一般建议在 2~3 为宜
```

**3.2、那怎么进行 producer 优化呢？**  

```css
buffer.memory:33554432 (32m)
# 在 Producer 端用来存放尚未发送出去的 Message 的缓冲区大小。缓冲区满了之后可以选择阻塞发送或抛出异常，由 block.on.buffer.full 的配置来决定。

compression.type:none
#默认发送不进行压缩，这里其实可以配置一种适合的压缩算法，可以大幅度的减缓网络压力和Broker 的存储压力。
```

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXeFzuCNYd7EAPsScZvZ6KhcWvvLhicX9SVRibv8pY7MtyrEYicyrfXfae9E6OQbUtIUr4bicpKBJSFNmA/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/nkwHybhD8YfjbkTFF8cnqQ](https://mp.weixin.qq.com/s/nkwHybhD8YfjbkTFF8cnqQ)
