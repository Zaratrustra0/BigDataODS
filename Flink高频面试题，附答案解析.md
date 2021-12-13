# Flink高频面试题，附答案解析
## 1. Flink 的容错机制（checkpoint）

Checkpoint 容错机制是 Flink 可靠性的基石，可以保证 Flink 集群在某个算子因为某些原因 (如 异常退出) 出现故障时，能够将整个应用流图的状态恢复到故障之前的某一状态，保证应用流图状态的一致性。Flink 的 Checkpoint 机制原理来自 “Chandy-Lamport algorithm” 算法。

每个需要 Checkpoint 的应用在启动时，Flink 的 JobManager 为其创建一个 CheckpointCoordinator(检查点协调器)，CheckpointCoordinator 全权负责本应用的快照制作。

**CheckpointCoordinator(检查点协调器)**，CheckpointCoordinator 全权负责本应用的快照制作。![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFC5n0MavwFE6hrqerBJUGzOwCGZkKXJx5ibuiaCYI2JicpBIrKtWIIIibyfSZicBfMjx9cacM7QEQjAWw/640?wx_fmt=png)

1.  CheckpointCoordinator(检查点协调器) 周期性的向该流应用的所有 source 算子发送 barrier(屏障)。
2.  当某个 source 算子收到一个 barrier 时，便暂停数据处理过程，然后将自己的当前状态制作成快照，并保存到指定的持久化存储中，最后向 CheckpointCoordinator 报告自己快照制作情况，同时向自身所有下游算子广播该 barrier，恢复数据处理
3.  下游算子收到 barrier 之后，会暂停自己的数据处理过程，然后将自身的相关状态制作成快照，并保存到指定的持久化存储中，最后向 CheckpointCoordinator 报告自身快照情况，同时向自身所有下游算子广播该 barrier，恢复数据处理。
4.  每个算子按照步骤 3 不断制作快照并向下游广播，直到最后 barrier 传递到 sink 算子，快照制作完成。
5.  当 CheckpointCoordinator 收到所有算子的报告之后，认为该周期的快照制作成功; 否则，如果在规定的时间内没有收到所有算子的报告，则认为本周期快照制作失败。

**文章推荐**：

[Flink 可靠性的基石 - checkpoint 机制详细解析](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247483947&idx=1&sn=adae434f4e32b31be51627888e7d9f76&scene=21#wechat_redirect)

## 2. Flink Checkpoint 与 Spark 的相比，Flink 有什么区别或优势吗

Spark Streaming 的 Checkpoint 仅仅是针对 Driver 的故障恢复做了数据和元数据的 Checkpoint。而 Flink 的 Checkpoint 机制要复杂了很多，它采用的是轻量级的分布式快照，实现了每个算子的快照，及流动中的数据的快照。

## 3. Flink 中的 Time 有哪几种

Flink 中的时间有三种类型，如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFC5n0MavwFE6hrqerBJUGzOoZd4pcpPdibjJrFdA5Y2vKzGgnvz1JOicuCwAtVXsONeLUdxib9qxhicA/640?wx_fmt=png)

-   **Event Time**：是事件创建的时间。它通常由事件中的时间戳描述，例如采集的日志数据中，每一条日志都会记录自己的生成时间，Flink 通过时间戳分配器访问事件时间戳。
-   **Ingestion Time**：是数据进入 Flink 的时间。
-   **Processing Time**：是每一个执行基于时间操作的算子的本地系统时间，与机器相关，默认的时间属性就是 Processing Time。

例如，一条日志进入 Flink 的时间为`2021-01-22 10:00:00.123`，到达 Window 的系统时间为`2021-01-22 10:00:01.234`，日志的内容如下：  
`2021-01-06 18:37:15.624 INFO Fail over to rm2`

对于业务来说，要统计 1min 内的故障日志个数，哪个时间是最有意义的？—— eventTime，因为我们要根据日志的生成时间进行统计。

## 4. 对于迟到数据是怎么处理的

Flink 中 WaterMark 和 Window 机制解决了流式数据的乱序问题，对于因为延迟而顺序有误的数据，可以根据 eventTime 进行业务处理，对于延迟的数据 Flink 也有自己的解决办法，主要的办法是给定一个允许延迟的时间，在该时间范围内仍可以接受处理延迟数据：

-   设置允许延迟的时间是通过 allowedLateness(lateness: Time) 设置
-   保存延迟数据则是通过 sideOutputLateData(outputTag: OutputTag\[T]) 保存
-   获取延迟数据是通过 DataStream.getSideOutput(tag: OutputTag\[X]) 获取

**文章推荐**：

[Flink 中极其重要的 Time 与 Window 详细解析](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247483905&idx=1&sn=11434f3788a8a78418d21bacddcedbf7&scene=21#wechat_redirect)

## 5. Flink 的运行必须依赖 Hadoop 组件吗

Flink 可以完全独立于 Hadoop，在不依赖 Hadoop 组件下运行。但是做为大数据的基础设施，Hadoop 体系是任何大数据框架都绕不过去的。Flink 可以集成众多 Hadooop 组件，例如 Yarn、Hbase、HDFS 等等。例如，Flink 可以和 Yarn 集成做资源调度，也可以读写 HDFS，或者利用 HDFS 做检查点。

## 6. Flink 集群有哪些角色？各自有什么作用

有以下三个角色：

**JobManager 处理器：** 

也称之为 Master，用于协调分布式执行，它们用来调度 task，协调检查点，协调失败时恢复等。Flink 运行时至少存在一个 master 处理器，如果配置高可用模式则会存在多个 master 处理器，它们其中有一个是 leader，而其他的都是 standby。

**TaskManager 处理器：** 

也称之为 Worker，用于执行一个 dataflow 的 task(或者特殊的 subtask)、数据缓冲和 data stream 的交换，Flink 运行时至少会存在一个 worker 处理器。

**Clint 客户端：** 

Client 是 Flink 程序提交的客户端，当用户提交一个 Flink 程序时，会首先创建一个 Client，该 Client 首先会对用户提交的 Flink 程序进行预处理，并提交到 Flink 集群中处理，所以 Client 需要从用户提交的 Flink 程序配置中获取 JobManager 的地址，并建立到 JobManager 的连接，将 Flink Job 提交给 JobManager

## 7. Flink 资源管理中 Task Slot 的概念

在 Flink 中每个 TaskManager 是一个 JVM 的进程, 可以在不同的线程中执行一个或多个子任务。为了控制一个 worker 能接收多少个 task。worker 通过 task slot（任务槽）来进行控制（一个 worker 至少有一个 task slot）。

## 8. Flink 的重启策略了解吗

Flink 支持不同的重启策略，这些重启策略控制着 job 失败后如何重启：

1.  **固定延迟重启策略**

固定延迟重启策略会尝试一个给定的次数来重启 Job，如果超过了最大的重启次数，Job 最终将失败。在连续的两次重启尝试之间，重启策略会等待一个固定的时间。

2.  **失败率重启策略**

失败率重启策略在 Job 失败后会重启，但是超过失败率后，Job 会最终被认定失败。在两个连续的重启尝试之间，重启策略会等待一个固定的时间。

3.  **无重启策略**

Job 直接失败，不会尝试进行重启。

## 9. Flink 是如何保证 Exactly-once 语义的

Flink 通过实现**两阶段提交**和状态保存来实现端到端的一致性语义。分为以下几个步骤：

开始事务（beginTransaction）创建一个临时文件夹，来写把数据写入到这个文件夹里面

预提交（preCommit）将内存中缓存的数据写入文件并关闭

正式提交（commit）将之前写完的临时文件放入目标目录下。这代表着最终的数据会有一些延迟

丢弃（abort）丢弃临时文件

若失败发生在预提交成功后，正式提交前。可以根据状态来提交预提交的数据，也可删除预提交的数据。

**文章推荐**：

[八张图搞懂 Flink 端到端精准一次处理语义 Exactly-once](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247484113&idx=2&sn=dceb0b19c31dedaa33ce0d59ea830bf0&scene=21#wechat_redirect)

## 10. 如果下级存储不支持事务，Flink 怎么保证 exactly-once

端到端的 exactly-once 对 sink 要求比较高，具体实现主要有**幂等写入**和**事务性写入**两种方式。

幂等写入的场景依赖于业务逻辑，更常见的是用事务性写入。而事务性写入又有预写日志（WAL）和两阶段提交（2PC）两种方式。

如果外部系统不支持事务，那么可以用预写日志的方式，把结果数据先当成状态保存，然后在收到 checkpoint 完成的通知时，一次性写入 sink 系统。

## 11. Flink 是如何处理反压的

Flink 内部是基于 producer-consumer 模型来进行消息传递的，Flink 的反压设计也是基于这个模型。Flink 使用了高效有界的分布式阻塞队列，就像 Java 通用的阻塞队列（BlockingQueue）一样。下游消费者消费变慢，上游就会受到阻塞。

## 12. Flink 中的状态存储

Flink 在做计算的过程中经常需要存储中间状态，来避免数据丢失和状态恢复。选择的状态存储策略不同，会影响状态持久化如何和 checkpoint 交互。Flink 提供了三种状态存储方式：**MemoryStateBackend、FsStateBackend、RocksDBStateBackend**。

## 13. Flink 是如何支持流批一体的

这道题问的比较开阔，如果知道 Flink 底层原理，可以详细说说，如果不是很了解，就直接简单一句话：**Flink 的开发者认为批处理是流处理的一种特殊情况。批处理是有限的流处理。Flink 使用一个引擎支持了 DataSet API 和 DataStream API**。

## 14. Flink 的内存管理是如何做的

Flink 并不是将大量对象存在堆上，而是将对象都序列化到一个预分配的内存块上。此外，Flink 大量的使用了堆外内存。如果需要处理的数据超出了内存限制，则会将部分数据存储到硬盘上。Flink 为了直接操作二进制数据实现了自己的序列化框架。

## 15. Flink CEP 编程中当状态没有到达的时候会将数据保存在哪里

在流式处理中，CEP 当然是要支持 EventTime 的，那么相对应的也要支持数据的迟到现象，也就是 watermark 的处理逻辑。CEP 对未匹配成功的事件序列的处理，和迟到数据是类似的。在 Flink CEP 的处理逻辑中，状态没有满足的和迟到的数据，都会存储在一个 Map 数据结构中，也就是说，如果我们限定判断事件序列的时长为 5 分钟，那么内存中就会存储 5 分钟的数据，这在我看来，也是对内存的极大损伤之一。

**文章推荐**：

[详解 Flink CEP](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247485117&idx=1&sn=6dd10883e11a7bd128e0d9c3a178bfa9&scene=21#wechat_redirect)

**--END--**

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

非常欢迎大家加我**个人微信**，有关大数据的问题我们在**群内**一起讨论

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zE6jg8pGqTM8bodQoPTqicfQUcAzbIQl9RmicTcXT7ecVRuEV0ZeicTU9nbpb0ggJhUc15E5Ly7OE5OA/640?wx_fmt=jpeg)

长按上方扫码二维码，加我微信，拉你进群

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zG2nW9msUBGp3lmbrXyyQPD4keYgsgVhnvmT3zBc5J9qQIxJNMxrzg6Laso7PoPGuQaMStSglsnibA/640?wx_fmt=png)

点个**在看**，支持一下 
 [https://mp.weixin.qq.com/s/9BbHr5kwcxu6ml0izFbwTQ](https://mp.weixin.qq.com/s/9BbHr5kwcxu6ml0izFbwTQ)
