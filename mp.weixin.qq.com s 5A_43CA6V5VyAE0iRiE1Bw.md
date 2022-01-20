# mp.weixin.qq.com/s/5A_43CA6V5VyAE0iRiE1Bw
Spark 是一个快速的大数据处理引擎，在实际的生产环境中，应用十分广泛。目前，Spark 仍然是大数据开发非常重要的一个工具，所以在面试的过程中，Spark 也会是被重点考察的对象。对于初学者而言，面对繁多的 Spark 相关概念，一时会难以厘清头绪，对于使用 Spark 开发的同学而言，有时候也会对这些概念感到模糊。本文主要梳理了几个关于 Spark 的比较重要的几个概念，在面试的过程中如果被问到 Spark 相关的问题，具体可以从以下几个方面展开即可，希望对你有所帮助。本文主要包括以下内容：  

-   运行架构
-   运行流程
-   执行模式
-   驱动程序
-   共享变量
-   宽依赖窄依赖
-   持久化
-   分区
-   综合实践案例

## 组成

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GWbh4py4rQ5xlTzNKAvGCibX7vVLVg10ibTdCMV3p4DXbzfOojVMmHNbA/640?wx_fmt=png)

Spark 栈包括 SQL 和 DataFrames，MLlib 机器学习， GraphX 和 SparkStreaming。用户可以在同一个应用程序中无缝组合使用这些库。

## 架构

![](https://mmbiz.qpic.cn/mmbiz_jpg/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GoGIchQ6WJROUhH8uc8MonWuhWnIhHq6L7UWItZtQslUibgSnOpdzDuQ/640?wx_fmt=jpeg)

Spark 运行架构包括集群资源管理器（Cluster Manager）、运行作业任务的工作节点（Worker Node）、每个应用的任务控制节点（Driver）和每个工作节点上负责具体任务的执行进程（Executor）。其中，集群资源管理器可以是 Spark 自带的资源管理器，也可以是 YARN 或 Mesos 等资源管理框架。

## 运行流程

![](https://mmbiz.qpic.cn/mmbiz_jpg/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GBF2ic2h9rOYxzf4cQG0KgLw4xKwLnqUmKebjPOFjvwp0urDyrVa0qoQ/640?wx_fmt=jpeg)

-   当一个 Spark 应用被提交时，首先需要为这个应用构建起基本的运行环境，即由任务控制节点（Driver）创建一个 SparkContext，由 SparkContext 负责和资源管理器（Cluster Manager）的通信以及进行资源的申请、任务的分配和监控等。SparkContext 会向资源管理器注册并申请运行 Executor 的资源；
-   资源管理器为 Executor 分配资源，并启动 Executor 进程，Executor 运行情况将随着 “心跳” 发送到资源管理器上；
-   SparkContext 根据 RDD 的依赖关系构建 DAG 图，DAG 图提交给 DAG 调度器（DAGScheduler）进行解析，将 DAG 图分解成多个 “阶段”（每个阶段都是一个任务集），并且计算出各个阶段之间的依赖关系，然后把一个个“任务集” 提交给底层的任务调度器（TaskScheduler）进行处理；Executor 向 SparkContext 申请任务，任务调度器将任务分发给 Executor 运行，同时，SparkContext 将应用程序代码发放给 Executor；
-   任务在 Executor 上运行，把执行结果反馈给任务调度器，然后反馈给 DAG 调度器，运行完毕后写入数据并释放所有资源。

## MapReduce VS Spark

与 Spark 相比，MapReduce 具有以下缺点：

-   表达能力有限
-   磁盘 IO 开销大
-   延迟高


-   任务之间的衔接涉及 IO 开销
-   在前一个任务执行完成之前，其他任务就无法开始，难以胜任复杂、多阶段的计算任务

与 MapReduce 相比，Spark 具有以下优点：具体包括两个方面

-   一是利用多线程来执行具体的任务（Hadoop MapReduce 采用的是进程模型），减少任务的启动开销；
-   二是 Executor 中有一个 BlockManager 存储模块，会将内存和磁盘共同作为存储设备，当需要多轮迭代计算时，可以将中间结果存储到这个存储模块里，下次需要时，就可以直接读该存储模块里的数据，而不需要读写到 HDFS 等文件系统里，因而有效减少了 IO 开销；或者在交互式查询场景下，预先将表缓存到该存储系统上，从而可以提高读写 IO 性能。

## 驱动程序 (Driver) 和 Executor

运行 main 函数的**驱动程序**进程位于集群中的一个节点上，负责三件事：

-   维护有关 Spark 应用程序的信息。
-   响应用户的程序或输入。
-   跨 Executor 分析、分配和调度作业。

驱动程序进程是绝对必要的——它是 Spark 应用程序的核心，并在应用程序的生命周期内维护所有相关信息。

Executor 负责实际执行驱动程序分配给他们的任务。这意味着每个 Executor 只负责两件事：

-   执行驱动程序分配给它的代码。
-   向 Driver 节点汇报该 Executor 的计算状态

## 分区

为了让每个 executor 并行执行工作，Spark 将数据分解成称为**partitions 的**块。分区是位于集群中一台物理机器上的行的集合。Dataframe 的分区表示数据在执行期间如何在机器集群中物理分布。

如果你有一个分区，即使你有数千个 Executor，Spark 的并行度也只有一个。如果你有很多分区但只有一个执行器，Spark 仍然只有一个并行度，因为只有一个计算资源。

## 执行模式：**Client VS Cluster VS Local**

执行模式能够在运行应用程序时确定 Driver 和 Executor 的物理位置。

有三种模式可供选择：

-   集群模式 (Cluster)。
-   客户端模式 (Client)。
-   本地模式 (Local)。

**集群模式** 可能是运行 Spark 应用程序最常见的方式。在集群模式下，用户将预编译的代码提交给集群管理器。除了启动 Executor 之外，集群管理器会在集群内的工作节点 (work) 上启动驱动程序 (Driver) 进程。这意味着集群管理器负责管理与 Spark 应用程序相关的所有进程。

**客户端模式** 与集群模式几乎相同，只是 Spark 驱动程序保留在提交应用程序的客户端节点上。这意味着客户端机器负责维护 Spark driver 进程，集群管理器维护 executor 进程。通常将这个节点称之为网关节点。

**本地模式**可以被认为是在你的计算机上运行一个程序，spark 会在同一个 JVM 中运行驱动程序和执行程序。

## RDD VS DataFrame VS DataSet

#### RDD

一个 RDD 是一个分布式对象集合，其本质是一个只读的、分区的记录集合。每个 RDD 可以分成多个分区，不同的分区保存在不同的集群节点上 (具体如下图所示)。RDD 是一种高度受限的共享内存模型，即 RDD 是只读的分区记录集合，所以也就不能对其进行修改。只能通过两种方式创建 RDD，一种是基于物理存储的数据创建 RDD，另一种是通过在其他 RDD 上作用转换操作(transformation，比如 map、filter、join 等) 得到新的 RDD。

-   基于内存

RDD 是位于内存中的对象集合。RDD 可以存储在内存、磁盘或者内存加磁盘中，但是，Spark 之所以速度快，是基于这样一个事实：数据存储在内存中，并且每个算子不会从磁盘上提取数据。

-   分区

分区是对逻辑数据集划分成不同的独立部分，分区是分布式系统性能优化的一种技术手段，可以减少网络流量传输，将相同的 key 的元素分布在相同的分区中可以减少 shuffle 带来的影响。RDD 被分成了多个分区，这些分区分布在集群中的不同节点。

-   强类型

RDD 中的数据是强类型的，当创建 RDD 的时候，所有的元素都是相同的类型，该类型依赖于数据集的数据类型。

-   懒加载

Spark 的转换操作是懒加载模式，这就意味着只有在执行了 action(比如 count、collect 等) 操作之后，才会去执行一些列的算子操作。

-   不可修改

RDD 一旦被创建，就不能被修改。只能从一个 RDD 转换成另外一个 RDD。

-   并行化

RDD 是可以被并行操作的，由于 RDD 是分区的，每个分区分布在不同的机器上，所以每个分区可以被并行操作。

-   持久化

由于 RDD 是懒加载的，只有 action 操作才会导致 RDD 的转换操作被执行，进而创建出相对应的 RDD。对于一些被重复使用的 RDD，可以对其进行持久化操作 (比如将其保存在内存或磁盘中，Spark 支持多种持久化策略)，从而提高计算效率。

#### DataFrame

DataFrame 代表一个不可变的分布式数据集合，其核心目的是让开发者面对数据处理时，只关心要做什么，而不用关心怎么去做，将一些优化的工作交由 Spark 框架本身去处理。DataFrame 是具有 Schema 信息的，也就是说可以被看做具有字段名称和类型的数据，类似于关系型数据库中的表，但是底层做了很多的优化。创建了 DataFrame 之后，就可以使用 SQL 进行数据处理。

用户可以从多种数据源中构造 DataFrame，例如：结构化数据文件，Hive 中的表，外部数据库或现有 RDD。DataFrame API 支持 Scala，Java，Python 和 R，在 Scala 和 Java 中，row 类型的 DataSet 代表 DataFrame，即 Dataset\[Row]等同于 DataFrame。

#### DataSet

DataSet 是 Spark 1.6 中添加的新接口，是 DataFrame 的扩展，它具有 RDD 的优点（强类型输入，支持强大的 lambda 函数）以及 Spark SQL 的优化执行引擎的优点。可以通过 JVM 对象构建 DataSet，然后使用函数转换（map，flatMap，filter)。值得注意的是，Dataset API 在 Scala 和 Java 中可用，Python 不支持 Dataset API。

另外，DataSet API 可以减少内存的使用，由于 Spark 框架知道 DataSet 的数据结构，因此在持久化 DataSet 时可以节省很多的内存空间。

## 共享变量

Spark 提供了两种类型的共享变量：广播变量和累加器。广播变量 (Broadcast variables) 是一个只读的变量，并且在每个节点都保存一份副本，而不需要在集群中发送数据。累加器 (Accumulators) 可以将所有任务的数据累加到一个共享结果中。

#### 广播变量

广播变量允许用户在集群中共享一个不可变的值，该共享的、不可变的值被持计划到集群的每台节点上。通常在需要将一份小数据集 (比如维表) 复制到集群中的每台节点时使用，比如日志分析的应用，web 日志通常只包含 pageId，而每个 page 的标题保存在一张表中，如果要分析日志(比如哪些 page 被访问的最多)，则需要将两者 join 在一起，这时就可以使用广播变量，将该表广播到集群的每个节点。具体如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GsCzR7geibqsAocQicGUJNFWgtL3maa4tcT8pgFgmsRD9H1zjiccsHdE1A/640?wx_fmt=png)

如上图，首先 Driver 将序列化对象分割成小的数据库，然后将这些数据块存储在 Driver 节点的 BlockManager 上。当 ececutor 中执行具体的 task 时，每个 executor 首先尝试从自己所在节点的 BlockManager 提取数据，如果之前已经提取的该广播变量的值，就直接使用它。如果没有找到，则会向远程的 Driver 或者其他的 Executor 中提取广播变量的值，一旦获取该值，就将其存储在自己节点的 BlockManager 中。这种机制可以避免 Driver 端向多个 executor 发送数据而造成的性能瓶颈。

#### 累加器

累加器 (Accumulator) 是 Spark 提供的另外一个共享变量，与广播变量不同，累加器是可以被修改的，是可变的。每个 transformation 会将修改的累加器值传输到 Driver 节点，累加器可以实现一个累加的功能，类似于一个计数器。Spark 本身支持数字类型的累加器，用户也可以自定义累加器的类型。

## 宽依赖和窄依赖

RDD 中不同的操作会使得不同 RDD 中的分区产不同的依赖，主要有两种依赖：**宽依赖**和**窄依赖**。宽依赖是指一个父 RDD 的一个分区对应一个子 RDD 的多个分区，窄依赖是指一个父 RDD 的分区对应与一个子 RDD 的分区，或者多个父 RDD 的分区对应一个子 RDD 分区。

窄依赖会被划分到同一个 stage 中，这样可以以管道的形式迭代执行。宽依赖所依赖的分区一般有多个，所以需要跨节点传输数据。从容灾方面看，两种依赖的计算结果恢复的方式是不同的，窄依赖只需要恢复父 RDD 丢失的分区即可，而宽依赖则需要考虑恢复所有父 RDD 丢失的分区。

DAGScheduler 会将 Job 的 RDD 划分到不同的 stage 中，并构建一个 stage 的依赖关系，即 DAG。这样划分的目的是既可以保障没有依赖关系的 stage 可以并行执行，又可以保证存在依赖关系的 stage 顺序执行。stage 主要分为两种类型，一种是 ShuffleMapStage，另一种是 ResultStage。其中 ShuffleMapStage 是属于上游的 stage，而 ResulStage 属于最下游的 stage，这意味着上游的 stage 先执行，最后执行 ResultStage。

## 持久化

#### 方式

在 Spark 中，RDD 采用惰性求值的机制，每次遇到 action 操作，都会从头开始执行计算。每次调用 action 操作，都会触发一次从头开始的计算。对于需要被重复使用的 RDD，spark 支持对其进行持久化，通过调用 persist() 或者 cache() 方法即可实现 RDD 的持计划。通过持久化机制可以避免重复计算带来的开销。值得注意的是，当调用持久化的方法时，只是对该 RDD 标记为了持久化，需要等到第一次执行 action 操作之后，才会把计算结果进行持久化。持久化后的 RDD 将会被保留在计算节点的内存中被后面的行动操作重复使用。

Spark 提供的两个持久化方法的主要区别是：cache() 方法默认使用的是内存级别，其底层调用的是 persist() 方法。

#### 持久化级别的选择

Spark 提供的持久化存储级别是在内存使用与 CPU 效率之间做权衡，通常推荐下面的选择方式：

-   如果内存可以容纳 RDD，可以使用默认的持久化级别，即 MEMORY_ONLY。这是 CPU 最有效率的选择，可以使作用在 RDD 上的算子尽可能第快速执行。
-   如果内存不够用，可以尝试使用 MEMORY_ONLY_SER，使用一个快速的序列化库可以节省很多空间，比如 Kryo 。

> tips：在一些 shuffle 算子中，比如 reduceByKey，即便没有显性调用 persist 方法，Spark 也会自动将中间结果进行持久化，这样做的目的是避免在 shuffle 期间发生故障而造成重新计算整个输入。即便如此，还是推荐对需要被重复使用的 RDD 进行持久化处理。

## coalesce VS repartition

repartition 算法对数据进行了 shuffle 操作，并创建了大小相等的数据分区。coalesce 操作合并现有分区以避免 shuffle，除此之外 coalesce 操作仅能用于减少分区，不能用于增加分区。

值得注意的是：使用 coalesce 在减少分区时，并没有对所有数据进行了移动，仅仅是在原来分区的基础之上进行了合并而已，所以效率较高，但是可能会引起数据倾斜。

## 综合案例

#### 一种数仓技术架构

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GTTAR2WeGXBhZ42SxI00NnVjOepHERGpVicdIxho5FjQL5Lzpv4QkZcQ/640?wx_fmt=png)

#### SparkStreaming 实时同步

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GacNmZy8ibtYomQgBNRYlcqc4w74WSDuN9iaT5Yd2IUYVRErtyyyriaKtQ/640?wx_fmt=png)

-   **订阅消费：** 

SparkStreaming 消费 kafka 埋点数据

-   **数据写入：** 

将解析的数据同时写入 HDFS 上的某个临时目录下及 Hive 表对应的分区目录下

-   **小文件合并：** 

由于是实时数据抽取，所以会生成大量的小文件，小文件的生成取决于 SparkStreaming 的 Batch Interval，比如一分钟一个 batch，那么一分钟就会生成一个小文件

#### 基于 SparkSQL 的批处理

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GZxoFqV6fc9ydPlT41z6dE0icnHnxITeD9IVZ9mQ3MJKlibHia0J9HaRQg/640?wx_fmt=png)

-   **ODS 层到 DWD 层数据去重**

SparkStreaming 数据输出是 At Least Once，可能会存在数据重复。在 ODS 层到 DWD 层进行明细数据处理时，需要对数据使用 row_number 去重。

-   **JDBC 写入 MySQL**

数据量大时，需要对数据进行重分区，并且为 DataSet 分区级别建立连接，采用批量提交的方式。

-   **使用 DISTRIBUTE BY 子句避免生成大量小文件**

**spark.sql.shuffle.partitions**的默认值为**200**，会导致以下问题

-   对于较小的数据，200 是一个过大的选择，由于调度开销，通常会导致处理速度变慢，同时会造成小文件的产生。
-   对于大数据集，200 很小，无法有效利用集群中的资源

使用 DISTRIBUTE BY cast(rand \* N as int) 这里的 N 是指具体最后落地生成多少个文件数。

#### 手动维护 offset 至 HBase

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GojXZ1qZo31Nu8YibWBHENibHPGo6GM15nZ7rbWLxgCrj6cw0CgibkUsYA/640?wx_fmt=png)

当作业发生故障或重启时，要保障从当前的消费位点去处理数据，单纯的依靠 SparkStreaming 本身的机制是不太理想，生产环境中通常借助手动管理来维护 kafka 的 offset。

#### 流应用监控告警

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsj3l7qEibdsNZFlZESBZicl4GWD7O0X3cX4cpaQYM09w5hGCodnIvTaA9ZcKK2ur0dInz13JPwsBFzw/640?wx_fmt=png)

-   实现**StreamingListener** 接口，重写**onBatchStarted**与**onBatchCompleted**方法
-   获取 batch 执行完成之后的时间，写入 Redis，数据类型的 key 为自定义的具体字符串，value 是 batch 处理完的结束时间
-   加入流作业监控
-   启动定时任务监控上述步骤写入 redis 的 kv 数据，一旦超出给定的阈值，则报错，并发出告警通知
-   使用 Azkaban 定时调度该任务 
    [https://mp.weixin.qq.com/s/5A_43CA6V5VyAE0iRiE1Bw](https://mp.weixin.qq.com/s/5A_43CA6V5VyAE0iRiE1Bw)
