# 【大数据面试题】Flink企业级面试题60连击
感谢读者胖子大佬提供的企业面试题。本文因为时间关系只有部分答案，**后续的答案小编会持续补全，请持续关注本系列**。部分答案可以参考[**《Flink 面试通关手册》**](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486653&idx=1&sn=30d93538517974ff1690d3da0906324e&chksm=fd3d4a28ca4ac33e4ac40eb72bcc17db099cab47467dbc0a44ecfdea3640cb9961d1663ee79a&scene=21#wechat_redirect)。  

年后升职加薪就靠它了。胖子大佬就在交流群里，需要加群的公众号回复【加群】。

**【开了赞赏，大家可以随意打赏，小编会用打赏金额✖️10 倍奖励给我们的胖子大佬】** 

#### 1、Flink 如何保证精确一次性消费

Flink 保证精确一次性消费主要依赖于两种 Flink 机制

1、Checkpoint 机制

2、二阶段提交机制

**Checkpoint 机制**

主要是当 Flink 开启 Checkpoint 的时候，会往 Source 端插入一条 barrir，然后这个 barrir 随着数据流向一直流动，当流入到一个算子的时候，这个算子就开始制作 checkpoint，制作的是从 barrir 来到之前的时候当前算子的状态，将状态写入状态后端当中。然后将 barrir 往下流动，当流动到 keyby 或者 shuffle 算子的时候，例如当一个算子的数据，依赖于多个流的时候，这个时候会有 barrir 对齐，也就是当所有的 barrir 都来到这个算子的时候进行制作 checkpoint，依次进行流动，当流动到 sink 算子的时候，并且 sink 算子也制作完成 checkpoint 会向 jobmanager 报告 checkpoint n 制作完成。

**二阶段提交机制**

Flink 提供了 CheckpointedFunction 与 CheckpointListener 这样两个接口，CheckpointedFunction 中有 snapshotState 方法，每次 checkpoint 触发执行方法，通常会将缓存数据放入状态中，可以理解为一个 hook，这个方法里面可以实现预提交，CheckpointListyener 中有 notifyCheckpointComplete 方法，checkpoint 完成之后的通知方法，这里可以做一些额外的操作。例如 FLinkKafkaConumerBase 使用这个来完成 Kafka offset 的提交，在这个方法里面可以实现提交操作。在 2PC 中提到如果对应流程例如某个 checkpoint 失败的话，那么 checkpoint 就会回滚，不会影响数据一致性，那么如果在通知 checkpoint 成功的之后失败了，那么就会在 initalizeSate 方法中完成事务的提交，这样可以保证数据的一致性。最主要是根据 checkpoint 的状态文件来判断的。

#### 2、flink 和 spark 区别

flink 是一个类似 spark 的 “开源技术栈”，因为它也提供了批处理，流式计算，图计算，交互式查询，机器学习等。flink 也是内存计算，比较类似 spark，但是不一样的是，spark 的计算模型基于 RDD，将流式计算看成是特殊的批处理，他的 DStream 其实还是 RDD。而 flink 吧批处理当成是特殊的流式计算，但是批处理和流式计算的层的引擎是两个，抽象了 DataSet 和 DataStream。flink 在性能上也表现的很好，流式计算延迟比 spark 少，能做到真正的流式计算，而 spark 只能是准流式计算。而且在批处理上，当迭代次数变多，flink 的速度比 spark 还要快，所以如果 flink 早一点出来，或许比现在的 Spark 更火。

#### 3、Flink 的状态可以用来做什么？

Flink 状态主要有两种使用方式：

1.  checkpoint 的数据恢复
2.  逻辑计算

#### 4、Flink 的 waterMark 机制，Flink watermark 传递机制

Flink 中的 watermark 机制是用来处理乱序的，flink 的时间必须是 event time ，有一个简单的例子就是，假如窗口是 5 秒，watermark 是 2 秒，那么 总共就是 7 秒，这个时候什么时候会触发计算呢，假设数据初始时间是 1000，那么等到 6999 的时候会触发 5999 窗口的计算，那么下一个就是 13999 的时候触发 10999 的窗口

其实这个就是 watermark 的机制，在多并行度中，例如在 kafka 中会所有的分区都达到才会触发窗口

#### 5、Flink 的时间语义

Event Time 事件产生的时间

Ingestion time 事件进入 Flink 的时间

processing time 事件进入算子的时间

#### 6、Flink window join

1、window join，即按照指定的字段和滚动滑动窗口和会话窗口进行 inner join

2、是 coGoup 其实就是 left join 和 right join，

3、interval join 也就是 在窗口中进行 join 有一些问题，因为有些数据是真的会后到的，时间还很长，那么这个时候就有了 interval join 但是必须要是事件时间，并且还要指定 watermark 和水位以及获取事件时间戳。并且要设置 偏移区间，因为 join 也不能一直等的。

#### 7、flink 窗口函数有哪些

Tumbing window

Silding window

Session window

Count winodw

#### 8、keyedProcessFunction 是如何工作的。假如是 event time 的话

keyedProcessFunction 是有一个 ontime 操作的，假如是 event 时间的时候 那么 调用的时间就是查看，event 的 watermark 是否大于 trigger time 的时间，如果大于则进行计算，不大于就等着，如果是 kafka 的话，那么默认是分区键最小的时间来进行触发。

#### 9、flink 是怎么处理离线数据的例如和离线数据的关联？

1、async io

2、broadcast

3、async io + cache

4、open 方法中读取，然后定时线程刷新，缓存更新是先删除，之后再来一条之后再负责写入缓存

#### 10、flink 支持的数据类型

DataSet Api 和 DataStream Api、Table Api

#### 11、Flink 出现数据倾斜怎么办

**Flink 数据倾斜如何查看：** 

在 flink 的 web ui 中可以看到数据倾斜的情况，就是每个 subtask 处理的数据量差距很大，例如有的只有一 M 有的 100M 这就是严重的数据倾斜了。

**KafkaSource 端发生的数据倾斜**

例如上游 kafka 发送的时候指定的 key 出现了数据热点问题，那么就在接入之后，做一个负载均衡（前提下游不是 keyby）。

**聚合类算子数据倾斜**

预聚合加全局聚合

#### 12、flink 维表关联怎么做的

1、async io

2、broadcast

3、async io + cache

4、open 方法中读取，然后定时线程刷新，缓存更新是先删除，之后再来一条之后再负责写入缓存

#### 13、Flink checkpoint 的超时问题 如何解决。

1、是否网络问题

2、是否是 barrir 问题

3、查看 webui，是否有数据倾斜

4、有数据倾斜的话，那么解决数据倾斜后，会有改善，

#### 14、flinkTopN 与离线的 TopN 的区别

topn 无论是在离线还是在实时计算中都是比较常见的功能，不同于离线计算中的 topn，实时数据是持续不断的，这样就给 topn 的计算带来很大的困难，因为要持续在内存中维持一个 topn 的数据结构，当有新数据来的时候，更新这个数据结构

#### 15、sparkstreaming 和 flink 里 checkpoint 的区别

sparkstreaming 的 checkpoint 会导致数据重复消费

但是 flink 的 checkpoint 可以 保证精确一次性，同时可以进行增量，快速的 checkpoint 的，有三个状态后端，memery、rocksdb、hdfs

#### 16、简单介绍一下 cep 状态编程

Complex Event Processing（CEP）：

FLink Cep 是在 FLink 中实现的复杂时间处理库，CEP 允许在无休止的时间流中检测事件模式，让我们有机会掌握数据中重要的部分，一个或多个由简单事件构成的时间流通过一定的规则匹配，然后输出用户想得到的数据，也就是满足规则的复杂事件。

#### 17、 Flink cep 连续事件的可选项有什么

#### 18、如何通过 flink 的 CEP 来实现支付延迟提醒

#### 19、Flink cep 你用过哪些业务场景

#### 20、cep 底层如何工作

#### 21、cep 怎么老化

#### 22、cep 性能调优

#### 23、Flink 的背压，介绍一下 Flink 的反压，你们是如何监控和发现的呢。

Flink 没有使用任何复杂的机制来解决反压问题，Flink 在数据传输过程中使用了分布式阻塞队列。我们知道在一个阻塞队列中，当队列满了以后发送者会被天然阻塞住，这种阻塞功能相当于给这个阻塞队列提供了反压的能力。

当你的任务出现反压时，如果你的上游是类似 Kafka 的消息系统，很明显的表现就是消费速度变慢，Kafka 消息出现堆积。

如果你的业务对数据延迟要求并不高，那么反压其实并没有很大的影响。但是对于规模很大的集群中的大作业，反压会造成严重的 “并发症”。首先任务状态会变得很大，因为数据大规模堆积在系统中，这些暂时不被处理的数据同样会被放到“状态” 中。另外，Flink 会因为数据堆积和处理速度变慢导致 checkpoint 超时，而 checkpoint 是 Flink 保证数据一致性的关键所在，最终会导致数据的不一致发生。

**Flink Web UI**

Flink 的后台页面是我们发现反压问题的第一选择。Flink 的后台页面可以直观、清晰地看到当前作业的运行状态。

Web UI，需要注意的是，只有用户在访问点击某一个作业时，才会触发反压状态的计算。在默认的设置下，Flink 的 TaskManager 会每隔 50ms 触发一次反压状态监测，共监测 100 次，并将计算结果反馈给 JobManager，最后由 JobManager 进行反压比例的计算，然后进行展示。

在生产环境中 Flink 任务有反压有三种 OK、LOW、HIGH

OK 正常

LOW 一般

HIGH 高负载

#### 24、Flink 的 CBO，逻辑执行计划和物理执行计划

Flink 的优化执行其实是借鉴的数据库的优化器来生成的执行计划。

CBO，成本优化器，代价最小的执行计划就是最好的执行计划。传统的数据库，成本优化器做出最优化的执行计划是依据统计信息来计算的。Flink 的成本优化器也一样。Flink 在提供最终执行前，优化每个查询的执行逻辑和物理执行计划。这些优化工作是交给底层来完成的。根据查询成本执行进一步的优化，从而产生潜在的不同决策：如何排序连接，执行哪种类型的连接，并行度等等。

// TODO

#### 25、Flink 中数据聚合，不使用窗口怎么实现聚合

-   valueState 用于保存单个值
-   ListState 用于保存 list 元素
-   MapState 用于保存一组键值对
-   ReducingState 提供了和 ListState 相同的方法，返回一个 ReducingFunction 聚合后的值。
-   AggregatingState 和 ReducingState 类似，返回一个 AggregatingState 内部聚合后的值

#### 26、Flink 中 state 有哪几种存储方式

Memery、RocksDB、HDFS

#### 27、Flink 异常数据怎么处理

异常数据在我们的场景中，一般分为缺失字段和异常值数据。

**异常值：**  例如宝宝的年龄的数据，例如对于母婴行业来讲，一个宝宝的年龄是一个至关重要的数据，可以说是最重要的，因为宝宝大于 3 岁几乎就不会在母婴上面购买物品。像我们的有当日、未知、以及很久的时间。这样都属于异常字段，这些数据我们会展示出来给店长和区域经理看，让他们知道多少个年龄是不准的。如果要处理的话，可以根据他购买的时间来进行实时矫正，例如孕妇服装、奶粉的段位、纸尿裤的大小，以及奶嘴啊一些能够区分年龄段的来进行处理。我们并没有实时处理这些数据，我们会有一个底层的策略任务夜维去跑，一个星期跑一次。

**缺失字段：**  例如有的字段真的缺失的很厉害，能修补就修补。不能修补就放弃，就像上家公司中的新闻推荐过滤器。

#### 28、Flink 监控你们怎么做的

1、我们监控了 Flink 的任务是否停止

2、我们监控了 Flink 的 Kafka 的 LAG

3、我们会进行实时数据对账，例如销售额。

#### 29、Flink 有数据丢失的可能吗

Flink 有三种数据消费语义：

1.  At Most Once 最多消费一次 发生故障有可能丢失
2.  At Least Once 最少一次 发生故障有可能重复
3.  Exactly-Once 精确一次 如果产生故障，也能保证数据不丢失不重复。

**flink 新版本已经不提供 At-Most-Once 语义。** 

#### 30、Flink interval join 你能简单的写一写吗

DataStream<T> keyed1 = ds1.keyBy(o -> o.getString("key"))  
DataStream<T> keyed2 = ds2.keyBy(o -> o.getString("key"))  
// 右边时间戳 - 5s&lt;= 左边流时间戳 &lt;= 右边时间戳 - 1s  
keyed1.intervalJoin(keyed2).between(Time.milliseconds(-5), Time.milliseconds(5))  

#### 31、Flink 提交的时候 并行度如何制定，以及资源如何配置

并行度根据 kafka topic 的并行度，一个并行度 3 个 G

#### 32、Flink 的 boardcast join 的原理是什么

利用 broadcast State 将维度数据流广播到下游所有 task 中。这个 broadcast 的流可以与我们的事件流进行 connect，然后在后续的 process 算子中进行关联操作即可。

#### 33、flink 的 source 端断了，比如 kafka 出故障，没有数据发过来，怎么处理？

会有报警，监控的 kafka 偏移量也就是 LAG。

#### 34、flink 有什么常用的流的 API?

window join 啊 cogroup 啊 map flatmap，async io 等

#### 35、flink 的水位线，你了解吗，能简单介绍一下吗

Flink 的 watermark 是一种延迟触发的机制。

一般 watermark 是和 window 结合来进行处理乱序数据的，Watermark 最根本就是一个时间机制，例如我设置最大乱序时间为 2s，窗口时间为 5 秒，那么就是当事件时间大于 7s 的时候会触发窗口。当然假如有数据分区的情况下，例如 kafka 中接入 watermake 的话，那么 watermake 是会流动的，取的是所有分区中最小的 watermake 进行流动，因为只有最小的能够保证，之前的数据都已经来到了，可以触发计算了。

#### 36、Flink 怎么维护 Checkpoint？在 HDFS 上存储的话会有小文件吗

默认情况下，如果设置了 Checkpoint 选项，Flink 只保留最近成功生成的 1 个 Checkpoint。当 Flink 程序失败时，可以从最近的这个 Checkpoint 来进行恢复。但是，如果我们希望保留多个 Checkpoint，并能够根据实际需要选择其中一个进行恢复，这样会更加灵活。Flink 支持保留多个 Checkpoint，需要在 Flink 的配置文件 conf/flink-conf.yaml 中，添加如下配置指定最多需要保存 Checkpoint 的个数。

关于小文件问题可以参考[代达罗斯之殇 - 大数据领域小文件问题解决攻略](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247495807&idx=1&sn=d23efef601586506e00435889a8e1657&chksm=fd3eb6eaca493ffcc18ccfc6c1112124be13486d8778427099e37b4ed536be1ffe1348f55131&token=915430468&lang=zh_CN&scene=21#wechat_redirect)。

#### 37、Spark 和 Flink 的序列化，有什么区别吗？

Spark 默认使用的是 Java 序列化机制，同时还有优化的机制，也就是 kryo

Flink 是自己实现的序列化机制，也就是 TypeInformation

#### 38、Flink 是怎么处理迟到数据的？但是实际开发中不能有数据迟到，怎么做？

Flink 的 watermark 是一种延迟触发的机制。

一般 watermark 是和 window 结合来进行处理乱序数据的，Watermark 最根本就是一个时间机制，例如我设置最大乱序时间为 2s，窗口时间为 5 秒，那么就是当事件时间大于 7s 的时候会触发窗口。当然假如有数据分区的情况下，例如 kafka 中接入 watermake 的话，那么 watermake 是会流动的，取的是所有分区中最小的 watermake 进行流动，因为只有最小的能够保证，之前的数据都已经来到了，可以触发计算了。

#### 39、画出 flink 执行时的流程图。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P05dSo5cRYHSNkbZGicKJmuOWEicqqf7xObFCpUEXUfCOZaZ7AcdBFtND5Y2ANeKaLY87NdsicxmMXw/640?wx_fmt=png)

#### 40、Flink 分区分配策略

#### 41、Flink 关闭后状态端数据恢复得慢怎么办？

#### 42、了解 flink 的 savepoint 吗？讲一下 savepoint 和 checkpoint 的不同和各有什么优势

#### 43、flink 的状态后端机制

Flink 的状态后端是 Flink 在做 checkpoint 的时候将状态快照持久化，有三种状态后端 Memery、HDFS、RocksDB

#### 44、flink 中滑动窗口和滚动窗口的区别，实际应用的窗口是哪种？用的是窗口长度和滑动步长是多少？

#### 45、用 flink 能替代 spark 的批处理功能吗

Flink 未来的目标是批处理和流处理一体化，因为批处理的数据集你可以理解为是一个有限的数据流。Flink 在批出理方面，尤其是在今年 Flink 1.9 Release 之后，合入大量在 Hive 方面的功能，你可以使用 Flink SQL 来读取 Hive 中的元数据和数据集，并且使用 Flink SQL 对其进行逻辑加工，不过目前 Flink 在批处理方面的性能，还是干不过 Spark 的。

目前看来，Flink 在批处理方面还有很多内容要做，当然，如果是实时计算引擎的引入，Flink 当然是首选。

#### 46、flink 计算的 UV 你们是如何设置状态后端保存数据

可以使用布隆过滤器。

#### 47、sparkstreaming 和 flink 在执行任务上有啥区别，不是简单的流处理和微批，sparkstreaming 提交任务是分解成 stage，flink 是转换 graph，有啥区别？

#### 48、flink 把 streamgraph 转化成 jobGraph 是在哪个阶段？

#### 49、Flink 中的 watermark 除了处理乱序数据还有其他作用吗？

还有 kafka 数据顺序消费的处理。

#### 50、flink 你一般设置水位线设置多少

我们之前设置的水位线是 6s

#### 52、Flink 任务提交流程

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P05dSo5cRYHSNkbZGicKJmu1YBlJXo2DyfCjkMyjia1K2icQSJIXS4oaQpvibKL8iatz3c4UnXSib9Omvg/640?wx_fmt=png)

Flink 任务提交后，Client 向 HDFS 上传 Flink 的 jar 包和配置，之后向 Yarn ResourceManager 提交任务，ResourceManager 分配 Container 资源并通知对应的 NodeManager 启动 ApplicationMaster，ApplicationMaster 启动后加载 Flink 的 jar 包和配置构建环境，然后启动 JobManager；之后 Application Master 向 ResourceManager 申请资源启动 TaskManager ，ResourceManager 分配 Container 资源后，由 ApplicationMaster 通知资源所在的节点的 NodeManager 启动 TaskManager，NodeManager 加载 Flink 的 Jar 包和配置构建环境并启动 TaskManager，TaskManager 启动向 JobManager 发送心跳，并等待 JobManager 向其分配任务。

#### 53、Flink 技术架构图

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P05dSo5cRYHSNkbZGicKJmu9ImnhdTKaHFFXhstH0yz3QrVia6V1AEhmzicoD6voozXk923RpYrpjhQ/640?wx_fmt=png)

#### 54、flink 如何实现在指定时间进行计算。

#### 55、手写 Flink topN

#### 57、Flink 的 Join 算子有哪些

一般 join 是发生在 window 上面的:

1、window join，即按照指定的字段和滚动滑动窗口和会话窗口进行 inner join

2、是 coGoup 其实就是 left join 和 right join，

3、interval join 也就是 在窗口中进行 join 有一些问题，因为有些数据是真的会后到的，时间还很长，那么这个时候就有了 interval join 但是必须要是事件时间，并且还要指定 watermark 和水位以及获取事件时间戳。并且要设置 偏移区间，因为 join 也不能一直等的。

#### 58、Flink1.10 有什么新特性吗？

**内存管理及配置优化**

Flink 目前的 TaskExecutor 内存模型存在着一些缺陷，导致优化资源利用率比较困难，例如：

-   流和批处理内存占用的配置模型不同
-   流处理中的 RocksDB state backend 需要依赖用户进行复杂的配置

为了让内存配置变的对于用户更加清晰、直观，Flink 1.10 对 TaskExecutor 的内存模型和配置逻辑进行了较大的改动 （FLIP-49 \[7]）。这些改动使得 Flink 能够更好地适配所有部署环境（例如 Kubernetes, Yarn, Mesos），让用户能够更加严格的控制其内存开销。

**Managed 内存扩展**

Managed 内存的范围有所扩展，还涵盖了 RocksDB state backend 使用的内存。尽管批处理作业既可以使用堆内内存也可以使用堆外内存，使用 RocksDB state backend 的流处理作业却只能利用堆外内存。因此为了让用户执行流和批处理作业时无需更改集群的配置，我们规定从现在起 managed 内存只能在堆外。

**简化 RocksDB 配置**

此前，配置像 RocksDB 这样的堆外 state backend 需要进行大量的手动调试，例如减小 JVM 堆空间、设置 Flink 使用堆外内存等。现在，Flink 的开箱配置即可支持这一切，且只需要简单地改变 managed 内存的大小即可调整 RocksDB state backend 的内存预算。

另一个重要的优化是，Flink 现在可以限制 RocksDB 的 native 内存占用，以避免超过总的内存预算—这对于 Kubernetes 等容器化部署环境尤为重要。

**统一的作业提交逻辑** 

在此之前，提交作业是由执行环境负责的，且与不同的部署目标（例如 Yarn, Kubernetes, Mesos）紧密相关。这导致用户需要针对不同环境保留多套配置，增加了管理的成本。

在 Flink 1.10 中，作业提交逻辑被抽象到了通用的 Executor 接口。新增加的 ExecutorCLI （引入了为任意执行目标指定配置参数的统一方法。此外，随着引入 JobClient 负责获取 JobExecutionResult，获取作业执行结果的逻辑也得以与作业提交解耦。

**原生 Kubernetes 集成（Beta）**

对于想要在容器化环境中尝试 Flink 的用户来说，想要在 Kubernetes 上部署和管理一个 Flink standalone 集群，首先需要对容器、算子及像 kubectl 这样的环境工具有所了解。

在 Flink 1.10 中，我们推出了初步的支持 session 模式的主动 Kubernetes 集成（FLINK-9953）。其中，“主动” 指 Flink ResourceManager (K8sResMngr) 原生地与 Kubernetes 通信，像 Flink 在 Yarn 和 Mesos 上一样按需申请 pod。用户可以利用 namespace，在多租户环境中以较少的资源开销启动 Flink。这需要用户提前配置好 RBAC 角色和有足够权限的服务账号。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P05dSo5cRYHSNkbZGicKJmurMP0YFn2CQRXSnnDWnJLXg6AFGtxeCsuWDZUvnVDKibfyZWnia1pxl6g/640?wx_fmt=png)

**Table API/SQL: 生产可用的 Hive 集成**

Flink 1.9 推出了预览版的 Hive 集成。该版本允许用户使用 SQL DDL 将 Flink 特有的元数据持久化到 Hive Metastore、调用 Hive 中定义的 UDF 以及读、写 Hive 中的表。Flink 1.10 进一步开发和完善了这一特性，带来了全面兼容 Hive 主要版本的生产可用的 Hive 集成。

**Batch SQL 原生分区支持**

此前，Flink 只支持写入未分区的 Hive 表。在 Flink 1.10 中，Flink SQL 扩展支持了 INSERT OVERWRITE 和 PARTITION 的语法（FLIP-63 ），允许用户写入 Hive 中的静态和动态分区。

-   写入静态分区

        INSERT { INTO | OVERWRITE } TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement1 FROM from_statement;
-   写入动态分区

        INSERT { INTO | OVERWRITE } TABLE tablename1 select_statement1 FROM from_statement;

    对分区表的全面支持，使得用户在读取数据时能够受益于分区剪枝，减少了需要扫描的数据量，从而大幅提升了这些操作的性能。

另外，除了分区剪枝，Flink 1.10 的 Hive 集成还引入了许多数据读取方面的优化，例如：

-   投影下推：Flink 采用了投影下推技术，通过在扫描表时忽略不必要的域，最小化 Flink 和 Hive 表之间的数据传输量。这一优化在表的列数较多时尤为有效。
-   LIMIT 下推：对于包含 LIMIT 语句的查询，Flink 在所有可能的地方限制返回的数据条数，以降低通过网络传输的数据量。
-   读取数据时的 ORC 向量化：为了提高读取 ORC 文件的性能，对于 Hive 2.0.0 及以上版本以及非复合数据类型的列，Flink 现在默认使用原生的 ORC 向量化读取器。

#### 59、Flink 的重启策略

**固定延迟重启策略**

固定延迟重启策略是尝试给定次数重新启动作业。如果超过最大尝试次数，则作业失败。在两次连续重启尝试之间，会有一个固定的延迟等待时间。

**故障率重启策略**

故障率重启策略在故障后重新作业，当设置的故障率（failure rate）超过每个时间间隔的故障时，作业最终失败。在两次连续重启尝试之间，重启策略延迟等待一段时间。

**无重启策略**

作业直接失败，不尝试重启。

**后备重启策略**

使用群集定义的重新启动策略。这对于启用检查点的流式传输程序很有帮助。默认情况下，如果没有定义其他重启策略，则选择固定延迟重启策略。

#### 60、Flink 什么时候用 aggregate() 或者 process()

**aggregate：**  增量聚合

**process：**  全量聚合

当计算累加操作时候可以使用 aggregate 操作。

当计算窗口内全量数据的时候使用 process，例如排序等操作。

#### 61、Flink 优化 你了解多少

#### 62、Flink 内存溢出怎么办

#### 63、说说 Flink 中的 keyState 包含哪些数据结构

#### 64、Flink shardGroup 的概念

 [https://mp.weixin.qq.com/s/FWit1b_6Me4Ay7UF6NtL3Q](https://mp.weixin.qq.com/s/FWit1b_6Me4Ay7UF6NtL3Q)
