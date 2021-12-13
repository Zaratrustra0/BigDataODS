# 上万字详解Spark Core（好文建议收藏）
进入主页，点击右上角 “设为星标”

比别人更快接收好文章

先来一个问题，也是面试中常问的：

**Spark 为什么会流行？**

原因 1：优秀的数据模型和丰富计算抽象

Spark 产生之前，已经有 MapReduce 这类非常成熟的计算系统存在了，并提供了高层次的 API(map/reduce)，把计算运行在集群中并提供容错能力，从而实现分布式计算。

虽然 MapReduce 提供了对数据访问和计算的抽象，但是对于数据的复用就是简单的将中间数据写到一个稳定的文件系统中 (例如 HDFS)，所以会产生数据的复制备份，磁盘的 I/O 以及数据的序列化，所以在遇到需要在多个计算之间复用中间结果的操作时效率就会非常的低。而这类操作是非常常见的，例如迭代式计算，交互式数据挖掘，图计算等。

认识到这个问题后，学术界的 AMPLab 提出了一个新的模型，叫做 RDD。**RDD 是一个可以容错且并行的数据结构**(其实可以理解成分布式的集合，操作起来和操作本地集合一样简单)，它可以让用户显式的将中间结果数据集保存在内存中，并且通过控制数据集的分区来达到数据存放处理最优化。同时 RDD 也提供了丰富的 API (map、reduce、filter、foreach、redeceByKey...) 来操作数据集。后来 RDD 被 AMPLab 在一个叫做 Spark 的框架中提供并开源。

简而言之，Spark 借鉴了 MapReduce 思想发展而来，保留了其分布式并行计算的优点并改进了其明显的缺陷。让中间数据存储在内存中提高了运行速度、并提供丰富的操作数据的 API 提高了开发速度。

原因 2：完善的生态圈 - fullstack

目前，Spark 已经发展成为一个包含多个子项目的集合，其中包含 SparkSQL、Spark Streaming、GraphX、MLlib 等子项目。

Spark Core：实现了 Spark 的基本功能，包含 RDD、任务调度、内存管理、错误恢复、与存储系统交互等模块。

Spark SQL：Spark 用来操作结构化数据的程序包。通过 Spark SQL，我们可以使用 SQL 操作数据。

Spark Streaming：Spark 提供的对实时数据进行流式计算的组件。提供了用来操作数据流的 API。

Spark MLlib：提供常见的机器学习 (ML) 功能的程序库。包括分类、回归、聚类、协同过滤等，还提供了模型评估、数据导入等额外的支持功能。

GraphX(图计算)：Spark 中用于图计算的 API，性能良好，拥有丰富的功能和运算符，能在海量数据上自如地运行复杂的图算法。

集群管理器：Spark 设计为可以高效地在一个计算节点到数千个计算节点之间伸缩计算。

StructuredStreaming：处理结构化流, 统一了离线和实时的 API。

## Spark VS Hadoop

\|  
 | Hadoop | Spark |
\| --- \| --- \| --- \|
| 类型 | 基础平台, 包含计算, 存储, 调度 | 分布式计算工具 |
| 场景 | 大规模数据集上的批处理 | 迭代计算, 交互式计算, 流计算 |
| 价格 | 对机器要求低, 便宜 | 对内存有要求, 相对较贵 |
| 编程范式 | Map+Reduce, API 较为底层, 算法适应性差 | RDD 组成 DAG 有向无环图, API 较为顶层, 方便使用 |
| 数据存储结构 | MapReduce 中间计算结果存在 HDFS 磁盘上, 延迟大 | RDD 中间运算结果存在内存中 , 延迟小 |
| 运行方式 | Task 以进程方式维护, 任务启动慢 | Task 以线程方式维护, 任务启动快 |

> ❣️注意：  
> 尽管 Spark 相对于 Hadoop 而言具有较大优势，但 Spark 并不能完全替代 Hadoop，Spark 主要用于替代 Hadoop 中的 MapReduce 计算模型。存储依然可以使用 HDFS，但是中间结果可以存放在内存中；调度可以使用 Spark 内置的，也可以使用更成熟的调度系统 YARN 等。  
> 实际上，Spark 已经很好地融入了 Hadoop 生态圈，并成为其中的重要一员，它可以借助于 YARN 实现资源调度管理，借助于 HDFS 实现分布式存储。  
> 此外，Hadoop 可以使用廉价的、异构的机器来做分布式存储与计算，但是，Spark 对硬件的要求稍高一些，对内存与 CPU 有一定的要求。

## 一、RDD 详解

### 1. 为什么要有 RDD?

在许多迭代式算法 (比如机器学习、图算法等) 和交互式数据挖掘中，不同计算阶段之间会重用中间结果，即一个阶段的输出结果会作为下一个阶段的输入。但是，之前的 MapReduce 框架采用非循环式的数据流模型，把中间结果写入到 HDFS 中，带来了大量的数据复制、磁盘 IO 和序列化开销。且这些框架只能支持一些特定的计算模式(map/reduce)，并没有提供一种通用的数据抽象。

AMP 实验室发表的一篇关于 RDD 的论文:《Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing》就是为了解决这些问题的。

RDD 提供了一个抽象的数据模型，让我们不必担心底层数据的分布式特性，只需将具体的应用逻辑表达为一系列转换操作 (函数)，不同 RDD 之间的转换操作之间还可以形成依赖关系，进而实现管道化，从而避免了中间结果的存储，大大降低了数据复制、磁盘 IO 和序列化开销，并且还提供了更多的 API(map/reduec/filter/groupBy...)。

### 2. RDD 是什么?

RDD(Resilient Distributed Dataset) 叫做弹性分布式数据集，是 Spark 中最基本的数据抽象，代表一个不可变、可分区、里面的元素可并行计算的集合。单词拆解：

-   Resilient ：它是弹性的，RDD 里面的中的数据可以保存在内存中或者磁盘里面
-   Distributed ：它里面的元素是分布式存储的，可以用于分布式计算
-   Dataset: 它是一个集合，可以存放很多元素

### 3. RDD 主要属性

进入 RDD 的源码中看下：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLKQYPAJI7QTTzkZ7OlgCmSxb57jcb4jxpGEDEWLWMiaqAFlq9nB3A4LUw/640?wx_fmt=png)

RDD 源码

在源码中可以看到有对 RDD 介绍的注释，我们来翻译下：

1.  A list of partitions ：一组分片 (Partition)/ 一个分区(Partition) 列表，即数据集的基本组成单位。对于 RDD 来说，每个分片都会被一个计算任务处理，分片数决定并行度。用户可以在创建 RDD 时指定 RDD 的分片个数，如果没有指定，那么就会采用默认值。
2.  A function for computing each split ：一个函数会被作用在每一个分区。Spark 中 RDD 的计算是以分片为单位的，compute 函数会被作用到每个分区上。
3.  A list of dependencies on other RDDs ：一个 RDD 会依赖于其他多个 RDD。RDD 的每次转换都会生成一个新的 RDD，所以 RDD 之间就会形成类似于流水线一样的前后依赖关系。在部分分区数据丢失时，Spark 可以通过这个依赖关系重新计算丢失的分区数据，而不是对 RDD 的所有分区进行重新计算。(Spark 的容错机制)
4.  Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)：可选项，对于 KV 类型的 RDD 会有一个 Partitioner，即 RDD 的分区函数，默认为 HashPartitioner。
5.  Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)：可选项, 一个列表，存储存取每个 Partition 的优先位置 (preferred location)。对于一个 HDFS 文件来说，这个列表保存的就是每个 Partition 所在的块的位置。按照 "移动数据不如移动计算" 的理念，Spark 在进行任务调度的时候，会尽可能选择那些存有数据的 worker 节点来进行任务计算。

**总结**

RDD 是一个数据集的表示，不仅表示了数据集，还表示了这个数据集从哪来，如何计算，主要属性包括：

1.  分区列表
2.  计算函数
3.  依赖关系
4.  分区函数 (默认是 hash)
5.  最佳位置

分区列表、分区函数、最佳位置，这三个属性其实说的就是数据集在哪，在哪计算更合适，如何分区；  
计算函数、依赖关系，这两个属性其实说的是数据集怎么来的。

## 二、RDD-API

### 1. RDD 的创建方式

1.  由外部存储系统的数据集创建，包括本地的文件系统，还有所有 Hadoop 支持的数据集，比如 HDFS、Cassandra、HBase 等：  
    `val rdd1 = sc.textFile("hdfs://node1:8020/wordcount/input/words.txt")`
2.  通过已有的 RDD 经过算子转换生成新的 RDD：  
    `val rdd2=rdd1.flatMap(_.split(" "))`
3.  由一个已经存在的 Scala 集合创建：  
    `val rdd3 = sc.parallelize(Array(1,2,3,4,5,6,7,8))`或  
    `val rdd4 = sc.makeRDD(List(1,2,3,4,5,6,7,8))`

makeRDD 方法底层调用了 parallelize 方法：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLKcNzxgf9o4z0ynhY0jHEElWJu0AIMbc9sKmbeQwspjfqj7OJZ4GtvNw/640?wx_fmt=png)

RDD 源码

### 2. RDD 的算子分类

RDD 的算子分为两类:

1.  Transformation 转换操作:**返回一个新的 RDD**
2.  Action 动作操作:**返回值不是 RDD(无返回值或返回其他的)**

> ❣️注意:  
> 1、RDD 不实际存储真正要计算的数据，而是记录了数据的位置在哪里，数据的转换关系 (调用了什么方法，传入什么函数)。  
> 2、RDD 中的所有转换都是惰性求值 / 延迟执行的，也就是说并不会直接计算。只有当发生一个要求返回结果给 Driver 的 Action 动作时，这些转换才会真正运行。  
> 3、之所以使用惰性求值 / 延迟执行，是因为这样可以在 Action 时对 RDD 操作形成 DAG 有向无环图进行 Stage 的划分和并行优化，这种设计让 Spark 更加有效率地运行。

### 3. Transformation 转换算子

| 转换算子                                                  | 含义                                                                                                               |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **map**(func)                                         | 返回一个新的 RDD，该 RDD 由每一个输入元素经过 func 函数转换后组成                                                                         |
| **filter**(func)                                      | 返回一个新的 RDD，该 RDD 由经过 func 函数计算后返回值为 true 的输入元素组成                                                                 |
| **flatMap**(func)                                     | 类似于 map，但是每一个输入元素可以被映射为 0 或多个输出元素 (所以 func 应该返回一个序列，而不是单一元素)                                                     |
| **mapPartitions**(func)                               | 类似于 map，但独立地在 RDD 的每一个分片上运行，因此在类型为 T 的 RDD 上运行时，func 的函数类型必须是 Iterator\[T] => Iterator\[U]                       |
| **mapPartitionsWithIndex**(func)                      | 类似于 mapPartitions，但 func 带有一个整数参数表示分片的索引值，因此在类型为 T 的 RDD 上运行时，func 的函数类型必须是 (Int, Interator\[T]) => Iterator\[U] |
| sample(withReplacement, fraction, seed)               | 根据 fraction 指定的比例对数据进行采样，可以选择是否使用随机数进行替换，seed 用于指定随机数生成器种子                                                       |
| **union**(otherDataset)                               | 对源 RDD 和参数 RDD 求并集后返回一个新的 RDD                                                                                    |
| intersection(otherDataset)                            | 对源 RDD 和参数 RDD 求交集后返回一个新的 RDD                                                                                    |
| **distinct**(\[numTasks]))                            | 对源 RDD 进行去重后返回一个新的 RDD                                                                                           |
| **groupByKey**(\[numTasks])                           | 在一个 (K,V) 的 RDD 上调用，返回一个(K, Iterator\[V]) 的 RDD                                                                  |
| **reduceByKey**(func, \[numTasks])                    | 在一个 (K,V) 的 RDD 上调用，返回一个 (K,V) 的 RDD，使用指定的 reduce 函数，将相同 key 的值聚合到一起，与 groupByKey 类似，reduce 任务的个数可以通过第二个可选的参数来设置 |
| aggregateByKey(zeroValue)(seqOp, combOp, \[numTasks]) | 对 PairRDD 中相同的 Key 值进行聚合操作，在聚合过程中同样使用了一个中立的初始值。和 aggregate 函数类似，aggregateByKey 返回值的类型不需要和 RDD 中 value 的类型一致      |
| **sortByKey**(\[ascending], \[numTasks])              | 在一个 (K,V) 的 RDD 上调用，K 必须实现 Ordered 接口，返回一个按照 key 进行排序的 (K,V) 的 RDD                                               |
| sortBy(func,\[ascending], \[numTasks])                | 与 sortByKey 类似，但是更灵活                                                                                             |
| **join**(otherDataset, \[numTasks])                   | 在类型为 (K,V) 和(K,W)的 RDD 上调用，返回一个相同 key 对应的所有元素对在一起的 (K,(V,W)) 的 RDD                                               |
| cogroup(otherDataset, \[numTasks])                    | 在类型为 (K,V) 和(K,W)的 RDD 上调用，返回一个 (K,(Iterable,Iterable)) 类型的 RDD                                                  |
| cartesian(otherDataset)                               | 笛卡尔积                                                                                                             |
| pipe(command, \[envVars])                             | 对 rdd 进行管道操作                                                                                                     |
| **coalesce**(numPartitions)                           | 减少 RDD 的分区数到指定值。在过滤大量数据之后，可以执行此操作                                                                                |
| **repartition**(numPartitions)                        | 重新给 RDD 分区                                                                                                       |

### 4. Action 动作算子

| 动作算子                                     | 含义                                                                                      |
| ---------------------------------------- | --------------------------------------------------------------------------------------- |
| reduce(func)                             | 通过 func 函数聚集 RDD 中的所有元素，这个功能必须是可交换且可并联的                                                 |
| collect()                                | 在驱动程序中，以数组的形式返回数据集的所有元素                                                                 |
| count()                                  | 返回 RDD 的元素个数                                                                            |
| first()                                  | 返回 RDD 的第一个元素 (类似于 take(1))                                                             |
| take(n)                                  | 返回一个由数据集的前 n 个元素组成的数组                                                                   |
| takeSample(withReplacement,num, \[seed]) | 返回一个数组，该数组由从数据集中随机采样的 num 个元素组成，可以选择是否用随机数替换不足的部分，seed 用于指定随机数生成器种子                     |
| takeOrdered(n, \[ordering])              | 返回自然顺序或者自定义顺序的前 n 个元素                                                                   |
| **saveAsTextFile**(path)                 | 将数据集的元素以 textfile 的形式保存到 HDFS 文件系统或者其他支持的文件系统，对于每个元素，Spark 将会调用 toString 方法，将它装换为文件中的文本 |
| **saveAsSequenceFile**(path)             | 将数据集中的元素以 Hadoop sequencefile 的格式保存到指定的目录下，可以使 HDFS 或者其他 Hadoop 支持的文件系统                 |
| saveAsObjectFile(path)                   | 将数据集的元素，以 Java 序列化的方式保存到指定的目录下                                                          |
| **countByKey**()                         | 针对 (K,V) 类型的 RDD，返回一个 (K,Int) 的 map，表示每一个 key 对应的元素个数                                   |
| foreach(func)                            | 在数据集的每一个元素上，运行函数 func 进行更新                                                              |
| **foreachPartition**(func)               | 在数据集的每一个分区上，运行函数 func                                                                   |

**统计操作：** 

| 算子             | 含义             |
| -------------- | -------------- |
| count          | 个数             |
| mean           | 均值             |
| sum            | 求和             |
| max            | 最大值            |
| min            | 最小值            |
| variance       | 方差             |
| sampleVariance | 从采样中计算方差       |
| stdev          | 标准差: 衡量数据的离散程度 |
| sampleStdev    | 采样的标准差         |
| stats          | 查看统计结果         |

## 三、RDD 的持久化 / 缓存

在实际开发中某些 RDD 的计算或转换可能会比较耗费时间，如果这些 RDD 后续还会频繁的被使用到，那么可以将这些 RDD 进行持久化 / 缓存，这样下次再使用到的时候就不用再重新计算了，提高了程序运行的效率。

    val rdd1 = sc.textFile("hdfs://node01:8020/words.txt")val rdd2 = rdd1.flatMap(x=>x.split(" ")).map((_,1)).reduceByKey(_+_)rdd2.cache //缓存/持久化rdd2.sortBy(_._2,false).collect//触发action,会去读取HDFS的文件,rdd2会真正执行持久化rdd2.sortBy(_._2,false).collect//触发action,会去读缓存中的数据,执行速度会比之前快,因为rdd2已经持久化到内存中了

### 持久化 / 缓存 API 详解

-   ersist 方法和 cache 方法

RDD 通过 persist 或 cache 方法可以将前面的计算结果缓存，但是**并不是这两个方法被调用时立即缓存**，**而是 \*\***触发后面的 action\***\* 时**，该 RDD 将会被缓存在计算节点的内存中，并供后面重用。  
通过查看 RDD 的源码发现 cache 最终也是调用了 persist 无参方法 (默认存储只存在内存中)：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLKM8UR0c71sfMkpSlg5gSV6ibcs6b3icX7fRfIlqpF6Oy8QjPrQvrDggJA/640?wx_fmt=png)

RDD 源码

-   存储级别

默认的存储级别都是仅在内存存储一份，Spark 的存储级别还有好多种，存储级别在 object StorageLevel 中定义的。

| 持久化级别                                | 说明                                                                                                                   |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| **MORY_ONLY(默认)**                    | 将 RDD 以非序列化的 Java 对象存储在 JVM 中。如果没有足够的内存存储 RDD，则某些分区将不会被缓存，每次需要时都会重新计算。这是默认级别                                         |
| **MORY_AND_DISK(开发中可以使用这个)**         | 将 RDD 以非序列化的 Java 对象存储在 JVM 中。如果数据在内存中放不下，则溢写到磁盘上．需要时则会从磁盘上读取                                                        |
| MEMORY_ONLY_SER (Java and Scala)     | 将 RDD 以序列化的 Java 对象 (每个分区一个字节数组) 的方式存储．这通常比非序列化对象 (deserialized objects) 更具空间效率，特别是在使用快速序列化的情况下，但是这种方式读取数据会消耗更多的 CPU |
| MEMORY_AND_DISK_SER (Java and Scala) | 与 MEMORY_ONLY_SER 类似，但如果数据在内存中放不下，则溢写到磁盘上，而不是每次需要重新计算它们                                                              |
| DISK_ONLY                            | 将 RDD 分区存储在磁盘上                                                                                                       |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2 等   | 与上面的储存级别相同，只不过将持久化数据存为两份，备份每个分区存储在两个集群节点上                                                                            |
| OFF_HEAP(实验中)                        | 与 MEMORY_ONLY_SER 类似，但将数据存储在堆外内存中。(即不是直接存储在 JVM 内存中)                                                                 |

**总结：** 

1.  RDD 持久化 / 缓存的目的是为了提高后续操作的速度
2.  缓存的级别有很多，默认只存在内存中, 开发中使用 memory_and_disk
3.  只有执行 action 操作的时候才会真正将 RDD 数据进行持久化 / 缓存
4.  实际开发中如果某一个 RDD 后续会被频繁的使用，可以将该 RDD 进行持久化 / 缓存

## 四、RDD 容错机制 Checkpoint

-   **持久化的局限：** 

持久化 / 缓存可以把数据放在内存中，虽然是快速的，但是也是最不可靠的；也可以把数据放在磁盘上，也不是完全可靠的！例如磁盘会损坏等。

-   **问题解决：** 

Checkpoint 的产生就是为了更加可靠的数据持久化，在 Checkpoint 的时候一般把数据放在在 HDFS 上，这就天然的借助了 HDFS 天生的高容错、高可靠来实现数据最大程度上的安全，实现了 RDD 的容错和高可用。

用法：

    SparkContext.setCheckpointDir("目录") //HDFS的目录RDD.checkpoint

-   **总结：** 
-   开发中如何保证数据的安全性性及读取效率：可以对频繁使用且重要的数据，先做缓存 / 持久化，再做 checkpint 操作。
-   持久化和 Checkpoint 的区别：

1.  位置：Persist 和 Cache 只能保存在本地的磁盘和内存中 (或者堆外内存 -- 实验中) Checkpoint 可以保存数据到 HDFS 这类可靠的存储上。
2.  生命周期：Cache 和 Persist 的 RDD 会在程序结束后会被清除或者手动调用 unpersist 方法 Checkpoint 的 RDD 在程序结束后依然存在，不会被删除。

## 五、RDD 依赖关系

### 1. 宽窄依赖

-   两种依赖关系类型：RDD 和它依赖的父 RDD 的关系有两种不同的类型，即**宽依赖**(wide dependency/shuffle dependency)**窄依赖**(narrow dependency)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLKqWDSaUQfNvcicBb98tfRjQ7EN7NhEY12zz7w0DF03fajEoj6eOZkdww/640?wx_fmt=png)

-   图解：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLK0u41E6nOvdEY6Jcs2xmanBvXn5XaRiaWVO8vibpIY17aHiaiaXqsyto6Cg/640?wx_fmt=png)

宽窄依赖

-   如何区分宽窄依赖：

窄依赖: 父 RDD 的一个分区只会被子 RDD 的一个分区依赖；  
宽依赖: 父 RDD 的一个分区会被子 RDD 的多个分区依赖 (涉及到 shuffle)。

### 2. 为什么要设计宽窄依赖

1.  对于窄依赖：

窄依赖的多个分区可以并行计算；  
窄依赖的一个分区的数据如果丢失只需要重新计算对应的分区的数据就可以了。

2.  对于宽依赖：

划分 Stage(阶段) 的依据: 对于宽依赖, 必须等到上一阶段计算完成才能计算下一阶段。

### 六、DAG 的生成和划分 Stage

### 1. DAG 介绍

-   DAG 是什么：

DAG(Directed Acyclic Graph 有向无环图) 指的是数据转换执行的过程，有方向，无闭环 (其实就是 RDD 执行的流程)；  
原始的 RDD 通过一系列的转换操作就形成了 DAG 有向无环图，任务执行时，可以按照 DAG 的描述，执行真正的计算 (数据被操作的一个过程)。

-   DAG 的边界

开始: 通过 SparkContext 创建的 RDD；  
结束: 触发 Action，一旦触发 Action 就形成了一个完整的 DAG。

### 2.DAG 划分 Stage

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zGTicrmicKXOEMgWDibX1oDFLKfnJxlsRQSf4gfhNvpvP5iczAM3hx2jXJLknWPe6PicnibgAMZ7yDtO1ZA/640?wx_fmt=png)

DAG 划分 Stage

**一个 Spark 程序可以有多个 DAG(有几个 Action，就有几个 DAG，上图最后只有一个 Action（图中未表现）, 那么就是一个 DAG)**。

一个 DAG 可以有多个 Stage(根据宽依赖 / shuffle 进行划分)。

**同一个 Stage 可以有多个 Task 并行执行**(**task 数 = 分区数**，如上图，Stage1 中有三个分区 P1、P2、P3，对应的也有三个 Task)。

可以看到这个 DAG 中只 reduceByKey 操作是一个宽依赖，Spark 内核会以此为边界将其前后划分成不同的 Stage。

同时我们可以注意到，在图中 Stage1 中，**从 textFile 到 flatMap 到 map 都是窄依赖，这几步操作可以形成一个流水线操作，通过 flatMap 操作生成的 partition 可以不用等待整个 RDD 计算结束，而是继续进行 map 操作，这样大大提高了计算的效率**。

-   为什么要划分 Stage? -- 并行计算

一个复杂的业务逻辑如果有 shuffle，那么就意味着前面阶段产生结果后，才能执行下一个阶段，即下一个阶段的计算要依赖上一个阶段的数据。那么我们按照 shuffle 进行划分 (也就是按照宽依赖就行划分)，就可以将一个 DAG 划分成多个 Stage / 阶段，在同一个 Stage 中，会有多个算子操作，可以形成一个 pipeline 流水线，流水线内的多个平行的分区可以并行执行。

-   如何划分 DAG 的 stage？

对于窄依赖，partition 的转换处理在 stage 中完成计算，不划分 (将窄依赖尽量放在在同一个 stage 中，可以实现流水线计算)。

对于宽依赖，由于有 shuffle 的存在，只能在父 RDD 处理完成后，才能开始接下来的计算，也就是说需要要划分 stage。

**总结：** 

Spark 会根据 shuffle / 宽依赖使用回溯算法来对 DAG 进行 Stage 划分，从后往前，遇到宽依赖就断开，遇到窄依赖就把当前的 RDD 加入到当前的 stage / 阶段中

具体的划分算法请参见 AMP 实验室发表的论文：《Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing》  
`http://xueshu.baidu.com/usercenter/paper/show?paperid=b33564e60f0a7e7a1889a9da10963461&site=xueshu_se`

* * *

**文章推荐**：  
[Spark 底层执行原理详细解析](https://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247483979&idx=1&sn=087bc70b936a8e3ec840fef28f21501f&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

    扫描二维码

   收获更多技术

五分钟学大数据

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zHo5ICR7ia2IAFRuQhnq3AfhnzfXwmHuJNZownrjcpWPHK77tGHw9NbnV5keRXVy5mpwSaabwN6icwg/640?wx_fmt=jpeg)

点个**在看**，支持一下 
 [https://mp.weixin.qq.com/s/V08jUJ4cMCtxVJQ8JbUdCA](https://mp.weixin.qq.com/s/V08jUJ4cMCtxVJQ8JbUdCA)
