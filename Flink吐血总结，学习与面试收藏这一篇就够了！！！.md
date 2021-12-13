# Flink吐血总结，学习与面试收藏这一篇就够了！！！
### Flink 核心特点

#### 批流一体

所有的数据都天然带有时间的概念，必然发生在某一个时间点。把事件按照时间顺序排列起来，就形成了一个事件流，也叫作数据流。**「无界数据」**是持续产生的数据，所以必须持续地处理无界数据流。**「有界数据」**，就是在一个确定的时间范围内的数据流，有开始有结束，一旦确定了就不会再改变。

#### 可靠的容错能力

-   集群级容错


-   集群管理器集成（Hadoop YARN、Mesos 或 Kubernetes）
-   高可用性设置（HA 模式基于 ApacheZooKeeper）


-   应用级容错（ Checkpoint）


-   一致性（其本身支持 Exactly-Once 语义）
-   轻量级（检查点的执行异步和增量检查点）


-   高吞吐、低延迟

#### 运行时架构

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LQEn41Z2GS02O6O7hHCibq5BXomrtdW1cS2VZA43BSfIaAAZoazia96Vw/640?wx_fmt=png)

运行时架构图

-   Flink 客户端


-   提交 Flink 作业到 Flink 集群
-   Stream Graph 和 Job Graph 构建


-   JobManager


-   资源申请
-   任务调度
-   应用容错


-   TaskManager


-   接收 JobManager 分发的子任务，管理子任务
-   任务处理（消费数据、处理数据）

## Flink 应用

#### 数据流

##### DataStream 体系

1.  DataStream(每个 DataStream 都有一个 Transformation 对象)
2.  DataStreamSource（DataStream 的起点）
3.  DataStreamSink（DataStream 的输出）
4.  KeyedStream（表示根据指定的 Key 记性分组的数据流）
5.  WindowdeStream & AllWindowedStream（根据 key 分组且基于 WindowAssigner 切分窗口的数据流）
6.  JoinedStreams & CoGroupedStreams


1.  JoinedStreams 底层使用 CoGroupedStreams 来实现
2.  CoGrouped 侧重的是 Group，对数据进行分组，是对同一个 key 上的两组集合进行操作
3.  Join 侧重的是数据对，对同一个 key 的每一对元素进行操作


8.  ConnectedStreams（表示两个数据流的组合）
9.  BroadcastStream & BroadcastConnectedStream（DataStream 的广播行为）
10. IterativeStream（包含 IterativeStream 的 Dataflow 是一个有向有环图）
11. AsyncDataStream（在 DataStream 上使用异步函数的能力）

##### 处理数据 API

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LIMnTCqaiaSmTbrAwabCRU6xKRhRy6CVxoSzBFjzmpHgkZ3krNLPyLYQ/640?wx_fmt=png)

处理数据 API

## 核心抽象

### 环境对象

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26L038j26g5tAyeTOeY18NdmAz7hibOHl8ibQPDViaDMAFLJvly9B86668Ag/640?wx_fmt=png)

### 数据流元素

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LBFQ1PJShI7v90l7PCtvTgviabSdebOykwfzyiaTGEq6KARHp3f7CpYyA/640?wx_fmt=png)

1.  StreamRecord（数据流中的一条记录｜事件）


1.  数据的值本身
2.  时间戳（可选）


3.  LatencyMarker（用来近似评估延迟）


1.  周期性的在数据源算子中创造出来的时间戳
2.  算子编号
3.  数据源所在的 Task 编号


5.  Watemark（是一个时间戳，用来告诉算子所有时间早于等于 Watermark 的事件或记录都已经到达，不会再有比 Watermark 更早的记录，算子可以根据 Watermark 触发窗口的计算、清理资源等）
6.  StreamStatus（用来通知 Task 是否会继续接收到上游的记录或者 Watermark）


1.  空闲状态（IDLE）。
2.  活动状态（ACTIVE）。

### Flink 异步 IO

#### 原理

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LlwicEq338A0YXNcOvx7ibnU7J04FOUibice7OG2x7vDXt73WSKvDhSMHcQ/640?wx_fmt=png)

顺序输出模式（先收到的数据元素先输出，后续数据元素的异步函数调用无论是否先完成，都需要等待）

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LKYSMNI0t0ibsvYKrsFJZc5mvkdMEgPUb5tn2zjJvng4tnhHPYJamnKw/640?wx_fmt=png)

无序输出模式（先处理完的数据元素先输出，不保证消息顺序）

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LPB81fHJTfLz3YkvJyYpmR2bTvg7BsDZJm01HBQqMs4aUica6hIXkeXA/640?wx_fmt=png)

### 数据分区

-   ForwardPartitioner（用于在同一个 OperatorChain 中上下游算子之间的数据转发，实际上数据是直接传递给下游的）
-   ShufflePartitioner（随机将元素进行分区，可以确保下游的 Task 能够均匀地获得数据）
-   ReblancePartitioner（以 Round-robin 的方式为每个元素分配分区，确保下游的 Task 可以均匀地获得数据，避免数据倾斜）
-   RescalingPartitioner（用 Round-robin 选择下游的一个 Task 进行数据分区，如上游有 2 个 Source，下游有 6 个 Map，那么每个 Source 会分配 3 个固定的下游 Map，不会向未分配给自己的分区写入数据）
-   BroadcastPartitioner（将该记录广播给所有分区）
-   KeyGroupStreamPartitioner（KeyedStream 根据 KeyGroup 索引编号进行分区，该分区器不是提供给用户来用的）

## 窗口

### 实现原理

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LWSPibcS2UTAoI5KdkFShmxBGiaZMHpITwEoqeSOjctmP2K0ep3XO7hkA/640?wx_fmt=png)

-   WindowAssigner（用来决定某个元素被分配到哪个 / 哪些窗口中去）
-   WindowTrigger（决定一个窗口何时能够呗计算或清除，每一个窗口都拥有一个属于自己的 Trigger）
-   WindowEvictor（窗口数据的过滤器，可在 Window Function 执行前或后，从 Window 中过滤元素）


-   CountEvictor：计数过滤器。在 Window 中保留指定数量的元素，并从窗口头部开始丢弃其余元素
-   DeltaEvictor：阈值过滤器。丢弃超过阈值的数据记录
-   TimeEvictor：时间过滤器。保留最新一段时间内的元素

### Watermark （水印）

#### 作用

用于处理乱序事件，而正确地处理乱序事件，通常用 Watermark 机制结合窗口来实现

#### DataStream Watermark 生成

1.  Source Function 中生成 Watermark
2.  DataStream API 中生成 Watermark


1.  AssingerWithPeriodicWatermarks （周期性的生成 Watermark 策略，不会针对每个事件都生成）
2.  AssingerWithPunctuatedWatermarks （对每个事件都尝试进行 Watermark 的生成，如果生成的结果是 null 或 Watermark 小于之前的，则不会发往下游）

## 内存管理

### 自主内存管理

#### 原因

1.  JVM 内存管理的不足


1.  有效数据密度低
2.  垃圾回收（大数据场景下需要消耗大量的内存，更容易触发 Full GC ）
3.  OOM 问题影响稳定性
4.  缓存未命中问题（Java 对象在堆上存储时并不是连续的）


3.  自主内存管理


1.  堆上内存的使用、监控、调试简单，堆外内存出现问题后的诊断则较为复杂
2.  Flink 有时需要分配短生命周期的 MemorySegment，在堆外内存上分配比在堆上内存开销更高。
3.  在 Flink 的测试中，部分操作在堆外内存上会比堆上内存慢
4.  大内存（上百 GB）JVM 的启动需要很长时间，Full GC 可以达到分钟级。使用堆外内存，可以将大量的数据保存在堆外，极大地减小堆内存，避免 GC 和内存溢出的问题。
5.  高效的 IO 操作。堆外内存在写磁盘或网络传输时是 zero-copy，而堆上内存则至少需要 1 次内存复制。
6.  堆外内存是进程间共享的。也就是说，即使 JVM 进程崩溃也不会丢失数据。这可以用来做故障恢复（Flink 暂时没有利用这项功能，不过未来很可能会去做）
7.  堆外内存的优势
8.  堆外内存的不足

### 内存模型

#### 内存模型图

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26L5Dv8ow41oNApdkCuW1hQInYGhCc5McZT5iafQxbWStMfH5YVVJ0lXiaQ/640?wx_fmt=png)

#### MemorySegment（内存段）

一个 MemorySegment 对应着一个 32KB 大小的内存块。这块内存既可以是堆上内存（Java 的 byte 数组），也可以是堆外内存（基于 Netty 的 DirectByteBuffer）

##### 图解

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26Lvhm7oviaADm2fJVNxfuGHoMnPt6wzWvct0gX1CwY2pgdcB7Dh9ACnBw/640?wx_fmt=png)

##### 结构

-   BYTE_ARRAY_BASE_OFFSET（二进制字节数组的起始索引）
-   LITTLE_ENDIAN（判断是否为 Little Endian 模式的字节存储顺序，若不是，就是 Big Endian 模式）


-   Big Endian：低地址存放最高有效字节（MSB）
-   Little Endian：低地址存放最低有效字节（LSB）X86 机器


-   HeapMemory（如果 MemeorySegment 使用堆上内存，则表示一个堆上的字节数组（byte［］），如果 MemorySegment 使用堆外内存，则为 null）
-   address（字节数组对应的相对地址）
-   addressLimit（标识地址结束位置）
-   size（内存段的字节数）

##### 实现

-   HybirdMemorySegment：用来分配堆上和堆外内存和堆上内存，Flink 在实际使用中只使用了改方式。原因是当有多个实现时，JIT 无法直接在编译时自动识别优化
-   HeapMemorySegment：用来分配堆上内存，实际没有实现

#### MemroyManager（内存管理器）

实际申请的是堆外内存，通过 RocksDB 的 Block Cache 和 WriterBufferManager 参数来限制，RocksDB 使用的内存量

## State（状态）

状态管理需要考虑的因素：

1.  状态数据的存储和访问
2.  状态数据的备份和恢复
3.  状态数据的划分和动态扩容
4.  状态数据的清理

### 分类

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LB4ib6ZWEQCfeicWNqEAg0EEuBnFsMTXYIWDMHHKEAjJAViaanKpyUbLHg/640?wx_fmt=png)

### 状态存储

-   MemoryStateBackend：纯内存，适用于验证、测试，不推荐生产环境
-   FsStateBackend：内存 + 文件，适用于长周期大规模的数据
-   RocksDBStateBackend：RocksDB，适用于长周期大规模的数据

### 重分布

-   ListState：并行度在改变的时候，会将并发上的每个 List 都取出，然后把这些 List 合并到一个新的 List, 根据元素的个数均匀分配给新的 Task
-   UnionListState: 把划分的方式交给用户去做，当改变并发的时候，会将原来的 List 拼接起来，然后不做划分，直接交给用户
-   BroadcastState: 变并发的时候，把这些数据分发到新的 Task 即可
-   KeyState：Key-Group 数量取决于最大并行度（MaxParallism）

## 作业提交

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LMacAsdC92FR7PgPEwInorbEiaa1qSZQMsiaPjM782zXhDC54h4yqwgzg/640?wx_fmt=png)

## 资源管理

### 关系图

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LxPKtq81wrJkMuDp2cngPo251B5r4WdRB2lIibFKAOXyQjPHxibloaDAA/640?wx_fmt=png)

### Slot 选择策略

-   LocationPreferenceSlotSelectionStrategy（位置优先的选择策略）


-   DefaultLocationPreferenceSlotSelectionStrategy（默认策略），该策略不考虑资源的均衡分配，会从满足条件的可用 Slot 集合选择第 1 个
-   EvenlySpreadOutLocationPreferenceSlotSelectionStrategy（均衡策略），该策略考虑资源的均衡分配，会从满足条件的可用 Slot 集合中选择剩余资源最多的 Slot，尽量让各个 TaskManager 均衡地承担计算压力


-   PreviousAllocationSlotSelectionStrategy（已分配 Slot 优先的选择策略），如果当前没有空闲的已分配 Slot，则仍然会使用位置优先的策略来分配和申请 Slot

## 调度

-   SchedulerNG （调度器）


-   作用
-   实现

1.  DefaultScheduler（使用 ScchedulerStrategy 来实现）
2.  LegacyScheduler（实际使用了原来的 ExecutionGraph 的调度逻辑）


1.  作业的生命周期管理（开始调度、挂起、取消）
2.  作业执行资源的申请、分配、释放
3.  作业状态的管理（发布过程中的状态变化、作业异常时的 FailOver
4.  作业的信息提供，对外提供作业的详细信息

-   SchedulingStrategy（调度策略）


-   实现

1.  EagerSchelingStrategy（该调度策略用来执行流计算作业的调度）
2.  LazyFromSourceSchedulingStrategy（该调度策略用来执行批处理作业的调度）


1.  startScheduling：调度入口，触发调度器的调度行为
2.  restartTasks：重启执行失败的 Task，一般是 Task 执行异常导致的
3.  onExecutionStateChange：当 Execution 的状态发生改变时
4.  onPartitionConsumable：当 IntermediateResultParitititon 中的数据可以消费时

-   ScheduleMode（调度模式）

1.  Eager 调度（该模式适用于流计算。一次性申请需要所有的资源，如果资源不足，则作业启动失败。）
2.  Lazy_From_Sources 分阶段调度（适用于批处理。从 Source Task 开始分阶段调度，申请资源的时候，一次性申请本阶段所需要的所有资源。上游 Task 执行完毕后开始调度执行下游的 Task，读取上游的数据，执行本阶段的计算任务，执行完毕之后，调度后一个阶段的 Task，依次进行调度，直到作业执行完成）
3.  Lazy_From_Sources_With_Batch_Slot_Request 分阶段 Slot 重用调度（适用于批处理。与分阶段调度基本一样，区别在于该模式下使用批处理资源申请模式，可以在资源不足的情况下执行作业，但是需要确保在本阶段的作业执行中没有 Shuffle 行为）

### 关键组件

#### JobMaster

1.  调度执行和管理（将 JobGraph 转化为 ExecutionGraph，调度 Task 的执行，并处理 Task 的异常）

-   InputSplit 分配
-   结果分区跟踪
-   作业执行异常

3.  作业 Slot 资源管理
4.  检查点与保存点
5.  监控运维相关
6.  心跳管理

#### Task

结构

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LdcrpItwLZd9K0wmrfnewMN6eIJkr3LNVgry2xkf9XtDQqh0OBibTUzw/640?wx_fmt=png)

### 作业调度失败

#### 失败异常分类

-   NonRecoverableError：不可恢复的错误。此类错误意味着即便是重启也无法恢复作业到正常状态，一旦发生此类错误，则作业执行失败，直接退出作业执行
-   PartitionDataMissingError：分区数据不可访问错误。下游 Task 无法读取上游 Task 产生的数据，需要重启上游的 Task
-   EnvironmentError：环境的错误。这种错误需要在调度策略上进行改进，如使用黑名单机制，排除有问题的机器、服务，避免将失败的 Task 重新调度到这些机器上。
-   RecoverableError：可恢复的错误

## 容错

### 容错保证语义

-   At-Most-Once（最多一次）
-   At-Leat-Once（最少一次）
-   Exactly-Once（引擎内严格一次）
-   End-to-End Exaacly-Once （端到端严格一次）

### 保存点恢复

1.  算子顺序的改变，如果对应的 UID 没变，则可以恢复，如果对应的 UID 变了则恢复失败。
2.  作业中添加了新的算子，如果是无状态算子，没有影响，可以正常恢复，如果是有状态的算子，跟无状态的算子一样处理。
3.  从作业中删除了一个有状态的算子，默认需要恢复保存点中所记录的所有算子的状态，如果删除了一个有状态的算子，从保存点恢复的时候被删除的 OperatorID 找不到，所以会报错，可以通过在命令中添加 - allowNonRestoredState （short: -n）跳过无法恢复的算子。
4.  添加和删除无状态的算子，如果手动设置了 UID，则可以恢复，保存点中不记录无状态的算子，如果是自动分配的 UID，那么有状态算子的 UID 可能会变（Flink 使用一个单调递增的计数器生成 UID，DAG 改版，计数器极有可能会变），很有可能恢复失败。
5.  恢复的时候调整并行度，Flink1.2.0 及以上版本, 如果没有使用作废的 API，则没问题；1.2.0 以下版本需要首先升级到 1.2.0 才可以。

### 端到端严格一次

#### 前提条件

-   数据源支持断点读取
-   外部存储支持回滚机制或者满足幂等性

### 图解

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LRp7O1DO4k7bkNnza4W6icV8FjleMoia9EVvqw99ibrAuQCYGd12TFnGIg/640?wx_fmt=png)

#### 实现

TwoPhaseCommitSinkFunction

1.  beginTransaction，开启一个事务，在临时目录中创建一个临时文件，之后写入数据到该文件中。此过程为不同的事务创建隔离，避免数据混淆。
2.  preCommit。预提交阶段。将缓存数据块写出到创建的临时文件，然后关闭该文件，确保不再写入新数据到该文件，同时开启一个新事务，执行属于下一个检查点的写入操作。
3.  commit。在提交阶段，以原子操作的方式将上一阶段的文件写入真正的文件目录下。如果提交失败，Flink 应用会重启，并调用 TwoPhaseCommitSinkFunction#recoverAndCommit 方法尝试恢复并重新提交事务。
4.  abort。一旦终止事务，删除临时文件。

## Flink SQL

### 关系图

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTIxibEDof281h0icUabpR26LYZvzDVB6tRWDtUavgrjNgLPPW4YSib3uWC4PYicvI3PRz1DMh2hQO3kw/640?wx_fmt=png)

## FLINK API

### DataStrem JOIN

#### Window JOIN

`stream.join(otherStream)  
    .where(<KeySelector>)  
    .equalTo(<KeySelector>)  
    .window(<WindowAssigner>)  
    .apply(<JoinFunction>)  
`

### Tumbling Window Join

\`DataStream<Integer> orangeStream = ...  
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)  
    .where(<KeySelector>)  
    .equalTo(<KeySelector>)  
    .window(TumblingEventTimeWindows.of(Time.milliseconds(2)))  
    .apply (new JoinFunction&lt;Integer, Integer, String> (){  
        @Override  
        public String join(Integer first, Integer second) {  
            return first + "," + second;  
        }  
    });

\`

### Sliding Window Join

\`DataStream<Integer> orangeStream = ...  
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)  
    .where(<KeySelector>)  
    .equalTo(<KeySelector>)  
    .window(SlidingEventTimeWindows.of(Time.milliseconds(2) /_ size _/, Time.milliseconds(1) /_ slide _/))  
    .apply (new JoinFunction&lt;Integer, Integer, String> (){  
        @Override  
        public String join(Integer first, Integer second) {  
            return first + "," + second;  
        }  
    });

\`

### Session Window Join

\`DataStream<Integer> orangeStream = ...  
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)  
    .where(<KeySelector>)  
    .equalTo(<KeySelector>)  
    .window(EventTimeSessionWindows.withGap(Time.milliseconds(1)))  
    .apply (new JoinFunction&lt;Integer, Integer, String> (){  
        @Override  
        public String join(Integer first, Integer second) {  
            return first + "," + second;  
        }  
    });

\`

## FLINK 往期

[Flink 最锋利的武器 Flink SQL（入门篇）](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484537&idx=1&sn=cae4fba6fdc1478e4ab6badb229e1f01&chksm=eb96f273dce17b650e80c4a916e321502192b76f9edbbf968567b22010134c0df931081c7532&scene=21#wechat_redirect)  

[FlinkSQL 窗口，让你眼前一亮，是否可以大吃一惊呢](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484549&idx=1&sn=0fd300b771d3e7f1f1398401022896df&chksm=eb96f28fdce17b99bf0f2629ca963405a8b9291df1e35eb3a5e36d93c377ba4a521bd7b5f3a3&scene=21#wechat_redirect)  

[FlinkSQL 自带 Sink 无法满足需求，自定义实现 Sink](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484566&idx=1&sn=b412696f3eeecae5863fcd47203abc65&chksm=eb96f29cdce17b8a57cfc19ce907034fbbebd45c2b4139f8003ecc529e8463efa18df9e2174d&scene=21#wechat_redirect)  

[Flink 的 Checkpoint 机制详解](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484454&idx=1&sn=99a39e2bccf4ddd3bd39187088fde917&chksm=eb96f22cdce17b3a122146a77096a97dcb0a0b5b61ce99d3c707876adf8f68c37436706316a3&scene=21#wechat_redirect)  

[Flink 的一致性保证](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484404&idx=1&sn=e384864bb6cf2bc7e029580d95706163&chksm=eb96f5fedce17ce854d6768525fdab8bcaa2bc0409ebc40d8907dd8ef171da5e948dfd68c2c0&scene=21#wechat_redirect)  

[Flink CEP - Flink 的复杂事件处理](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484384&idx=1&sn=8ec253c52feed1b33e89b56820157bf2&chksm=eb96f5eadce17cfc9f16983bf7736c3b756b0d2fe24487513ad8c9879e95d23b48240a5bfe5d&scene=21#wechat_redirect)  

[Flink 的状态与容错](http://mp.weixin.qq.com/s?__biz=MzI4MjU4MzkwOQ==&mid=2247484362&idx=1&sn=d3e4463606984dfa30fa5354def828cc&chksm=eb96f5c0dce17cd63e907e18904d9c0375e31a3e4bf86e1783a3c7364be6b22ccaf936afb32e&scene=21#wechat_redirect)  

同学共进，点赞，转发，在看，关注，是我学习之动力。

和我联系吧，交流大数据知识, 一起成长\~\~~ 
 [https://mp.weixin.qq.com/s/44G_siAfCLINOR0bBrun3g](https://mp.weixin.qq.com/s/44G_siAfCLINOR0bBrun3g)
