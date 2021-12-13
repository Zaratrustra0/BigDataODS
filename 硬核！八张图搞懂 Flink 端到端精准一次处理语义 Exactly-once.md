# 硬核！八张图搞懂 Flink 端到端精准一次处理语义 Exactly-once
进入主页，点击右上角 “设为星标”

比别人更快接收好文章

## Flink

在 Flink 中需要端到端精准一次处理的位置有三个：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjoIKmqrlryqiaKBk2N1ICZbPUZXmGFEG0nbICGmSnbGUq56TSJoZrKWgQ/640?wx_fmt=png)

Flink 端到端精准一次处理

-   **Source 端**：数据从上一阶段进入到 Flink 时，需要保证消息精准一次消费。
-   **Flink 内部端**：这个我们已经了解，利用 Checkpoint 机制，把状态存盘，发生故障的时候可以恢复，保证内部的状态一致性。不了解的小伙伴可以看下我之前的文章：

    [Flink 可靠性的基石 - checkpoint 机制详细解析](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247483947&idx=1&sn=adae434f4e32b31be51627888e7d9f76&chksm=ce77f4faf9007decd2f78a788a89e6777bb7bec79f4e59093474532ca5cf774284e2fe35e1bd&token=1679639512&lang=zh_CN&scene=21#wechat_redirect)
-   **Sink 端**：将处理完的数据发送到下一阶段时，需要保证数据能够准确无误发送到下一阶段。

在 Flink 1.4 版本之前，精准一次处理只限于 Flink 应用内，也就是所有的 Operator 完全由 Flink 状态保存并管理的才能实现精确一次处理。但 Flink 处理完数据后大多需要将结果发送到外部系统，比如 Sink 到 Kafka 中，这个过程中 Flink 并不保证精准一次处理。

在 Flink 1.4 版本正式引入了一个里程碑式的功能：两阶段提交 Sink，即 TwoPhaseCommitSinkFunction 函数。该 SinkFunction 提取并封装了**两阶段提交协议**中的公共逻辑，自此 Flink 搭配特定 Source 和 Sink（如  Kafka 0.11 版）**实现精确一次处理语义**(英文简称：EOS，即 Exactly-Once Semantics)。

## Flink 端到端精准一次处理语义（EOS）

注：以下内容适用于 Flink 1.4 及之后版本

**对于 Source 端**：Source 端的精准一次处理比较简单，毕竟数据是落到 Flink 中，所以 Flink 只需要保存消费数据的偏移量即可， 如消费 Kafka 中的数据，Flink 将 Kafka Consumer 作为 Source，可以将偏移量保存下来，如果后续任务出现了故障，恢复的时候可以由连接器重置偏移量，重新消费数据，保证一致性。

**对于 Sink 端**：**Sink 端是最复杂的**，因为数据是落地到其他系统上的，数据一旦离开 Flink 之后，Flink 就监控不到这些数据了，所以精准一次处理语义必须也要应用于 Flink 写入数据的外部系统，故这些外部系统必须提供一种手段允许提交或回滚这些写入操作，同时还要保证与 Flink Checkpoint 能够协调使用（Kafka 0.11 版本已经实现精确一次处理语义）。

我们以 Flink 与 Kafka 组合为例，Flink 从 Kafka 中读数据，处理完的数据在写入 Kafka 中。

为什么以 Kafka 为例，第一个原因是目前大多数的 Flink 系统读写数据都是与 Kafka 系统进行的。第二个原因，也是**最重要的原因 Kafka 0.11 版本正式发布了对于事务的支持，这是与 Kafka 交互的 Flink 应用要实现端到端精准一次语义的必要条件**。

当然，Flink 支持这种精准一次处理语义并不只是限于与 Kafka 的结合，可以使用任何 Source/Sink，只要它们提供了必要的协调机制。

## Flink 与 Kafka 组合

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjot0o4asqwLv5xrMiaCkogx8O9SdibMLTRPIfKicLKzociaZPEDMphAFHfXA/640?wx_fmt=png)

Flink 应用示例

如上图所示，Flink 中包含以下组件：

1.  一个 Source，从 Kafka 中读取数据（即 KafkaConsumer）
2.  一个时间窗口化的聚会操作（Window）
3.  一个 Sink，将结果写入到 Kafka（即 KafkaProducer）

**若要 Sink 支持精准一次处理语义 (EOS)，它必须以事务的方式写数据到 Kafka**，这样当提交事务时两次 Checkpoint 间的所有写入操作当作为一个事务被提交。这确保了出现故障或崩溃时这些写入操作能够被回滚。

当然了，在一个分布式且含有多个并发执行 Sink 的应用中，仅仅执行单次提交或回滚是不够的，因为所有组件都必须对这些提交或回滚达成共识，这样才能保证得到一个一致性的结果。Flink 使用**两阶段提交协议以及预提交 (Pre-commit) 阶段**来解决这个问题。

## 两阶段提交协议（2PC）

**两阶段提交协议（Two-Phase Commit，2PC）是很常用的解决分布式事务问题的方式，它可以保证在分布式事务中，要么所有参与进程都提交事务，要么都取消，即实现 ACID 中的 A （原子性）**。

在数据一致性的环境下，其代表的含义是：要么所有备份数据同时更改某个数值，要么都不改，以此来达到数据的**强一致性**。

**两阶段提交协议中有两个重要角色，协调者（Coordinator）和参与者（Participant），其中协调者只有一个，起到分布式事务的协调管理作用，参与者有多个**。

顾名思义，两阶段提交将提交过程划分为连续的两个阶段：**表决阶段（Voting）和提交阶段（Commit）**。

两阶段提交协议过程如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjovNTfc0piav5T46NiaVPjslAHcVF5Sg8Clpgq27SfibHHIejeDpJ76L1vQ/640?wx_fmt=png)

两阶段提交协议

**第一阶段：表决阶段**

1.  协调者向所有参与者发送一个 VOTE_REQUEST 消息。
2.  当参与者接收到 VOTE_REQUEST 消息，向协调者发送 VOTE_COMMIT 消息作为回应，告诉协调者自己已经做好准备提交准备，如果参与者没有准备好或遇到其他故障，就返回一个 VOTE_ABORT 消息，告诉协调者目前无法提交事务。

**第二阶段：提交阶段**

1.  协调者收集来自各个参与者的表决消息。如果**所有参与者一致认为可以提交事务，那么协调者决定事务的最终提交**，在此情形下协调者向所有参与者发送一个 GLOBAL_COMMIT 消息，通知参与者进行本地提交；如果所有参与者中有**任意一个返回消息是 VOTE_ABORT，协调者就会取消事务**，向所有参与者广播一条 GLOBAL_ABORT 消息通知所有的参与者取消事务。
2.  每个提交了表决信息的参与者等候协调者返回消息，如果参与者接收到一个 GLOBAL_COMMIT 消息，那么参与者提交本地事务，否则如果接收到 GLOBAL_ABORT 消息，则参与者取消本地事务。

## 两阶段提交协议在 Flink 中的应用

**Flink 的两阶段提交思路**：

我们从 Flink 程序启动到消费 Kafka 数据，最后到 Flink 将数据 Sink 到 Kafka 为止，来分析 Flink 的精准一次处理。

1.  当 Checkpoint 启动时，JobManager 会将检查点分界线（checkpoint battier）注入数据流，checkpoint barrier 会在算子间传递下去，如下如所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjoBp3VIQhnQr4vso6H8gHq0M2WnupG3nfTKEiaiavWiaGHvUc4DjI4NSNlg/640?wx_fmt=png)

Flink 精准一次处理：Checkpoint 启动

2.  **Source 端**：**Flink Kafka Source 负责保存 Kafka 消费 offset**，当 Chckpoint 成功时 Flink 负责提交这些写入，否则就终止取消掉它们，当 Chckpoint 完成位移保存，它会将 checkpoint barrier（检查点分界线） 传给下一个 Operator，然后每个算子会对当前的状态做个快照，**保存到状态后端**（State Backend）。

    **对于 Source 任务而言，就会把当前的 offset 作为状态保存起来。下次从 Checkpoint 恢复时，Source 任务可以重新提交偏移量，从上次保存的位置开始重新消费数据**，如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjovDibJChq3gy0PFaDfQp39w4CFBZ3FUKNJMohibYg7VDUWIvkgribdUglQ/640?wx_fmt=png)

Flink 精准一次处理：checkpoint barrier 及 offset 保存

3.  **Slink 端**：从 Source 端开始，每个内部的 transform 任务遇到 checkpoint barrier（检查点分界线）时，都会把状态存到 Checkpoint 里。数据处理完毕到 Sink 端时，Sink 任务首先把数据写入外部 Kafka，这些数据都属于预提交的事务（还不能被消费），**此时的 Pre-commit 预提交阶段下 Data Sink 在保存状态到状态后端的同时还必须预提交它的外部事务**，如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjomiaZ2ONibt9Gvia2ndfwcEQM1oOwkzROlygUXqy5p9qmHPibyuD7UrickOg/640?wx_fmt=png)

Flink 精准一次处理：预提交到外部系统

4.  **当所有算子任务的快照完成**（所有创建的快照都被视为是 Checkpoint 的一部分），**也就是这次的 Checkpoint 完成时，JobManager 会向所有任务发通知，确认这次 Checkpoint 完成，此时 Pre-commit 预提交阶段才算完成**。才正式到**两阶段提交协议的第二个阶段：commit 阶段**。该阶段中 JobManager 会为应用中每个 Operator 发起 Checkpoint 已完成的回调逻辑。

    本例中的 Data Source 和窗口操作无外部状态，因此在该阶段，这两个 Opeartor 无需执行任何逻辑，但是 **Data Sink 是有外部状态的，此时我们必须提交外部事务**，当 Sink 任务收到确认通知，就会正式提交之前的事务，Kafka 中未确认的数据就改为 “已确认”，数据就真正可以被消费了，如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjoY1ccAFibrJRYulibYovDYnl3nMiab0eggWyuW1loibUTxJXWLdC5KpiafuQ/640?wx_fmt=png)

Flink 精准一次处理：数据精准被消费

> 注：Flink 由 JobManager 协调各个 TaskManager 进行 Checkpoint 存储，Checkpoint 保存在 StateBackend（状态后端） 中，默认 StateBackend 是内存级的，也可以改为文件级的进行持久化保存。

最后，一张图总结下 Flink 的 EOS：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zFrStTricbsMvvLvTO5hFEjoiaibr0hWk3UBa3S356KyKOsDzMpwUuicmD6sWgqJ0Ou35zdNM7MYEyVoQ/640?wx_fmt=png)

Flink 端到端精准一次处理

**此图建议保存，总结全面且简明扼要，再也不怂面试官！**

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

非常欢迎大家加我**个人微信**，有关大数据的问题我们在**群内**一起讨论

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zE6jg8pGqTM8bodQoPTqicfQUcAzbIQl9RmicTcXT7ecVRuEV0ZeicTU9nbpb0ggJhUc15E5Ly7OE5OA/640?wx_fmt=jpeg)

长按上方扫码二维码，加我微信，拉你进群

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zG2nW9msUBGp3lmbrXyyQPD4keYgsgVhnvmT3zBc5J9qQIxJNMxrzg6Laso7PoPGuQaMStSglsnibA/640?wx_fmt=png)

点个**在看**，支持一下 
 [https://mp.weixin.qq.com/s/0dOQi9LwcscbJziWaBSwDQ](https://mp.weixin.qq.com/s/0dOQi9LwcscbJziWaBSwDQ)
