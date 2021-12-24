# 三万字，Spark学习笔记
## Spark 基础

### Spark 特性

Spark 使用简练优雅的 Scala 语言编写，基于 Scala 提供了交互式编程体验，同时提供多种方便易用的 API。Spark 遵循 “一个软件栈满足不同应用场景” 的设计理念，逐渐形成了一套完整的生态系统（包括 Spark 提供内存计算框架、SQL 即席查询（Spark  SQL）、流式计算（Spark  Streaming）、机器学习（MLlib）、图计算（Graph X）等），Spark 可以部署在 yarn 资源管理器上，提供一站式大数据解决方案，可以同时支持批处理、流处理、交互式查询。

MapReduce 计算模型延迟高，无法胜任实时、快速计算的需求，因而只适用于离线场景，Spark 借鉴 MapReduce 计算模式，但与之相比有以下几个优势（快、易用、全面）：

-   Spark 提供更多种数据集操作类型，编程模型比 MapReduce 更加灵活；
-   Spark 提供内存计算，将计算结果直接放在内存中，减少了迭代计算的 IO 开销，有更高效的运算效率。
-   Spark 基于 DAG 的任务调度执行机制，迭代效率更高；在实际开发中 MapReduce 需要编写很多底层代码，不够高效，Spark 提供了多种高层次、简洁的 API 实现相同功能的应用程序，实现代码量比 MapReduce 少很多。

Spark 作为计算框架只是取代了 Hadoop 生态系统中的 MapReduce 计算框架，它任需要 HDFS 来实现数据的分布式存储，Hadoop 中的其他组件依然在企业大数据系统中发挥着重要作用。

**Spark 的不足：** 虽然 Spark 很快，但现在在生产环境中仍然不尽人意，无论扩展性、稳定性、管理性等方面都需要进一步增强；同时 Spark 在流处理领域能力有限，如果要实现亚秒级或大容量的数据获取或处理需要其他流处理产品。

Cloudera 旨在让 Spark 流数据技术适用于 80% 的使用场合，就考虑到这一缺陷，在实时分析（而非简单数据过滤或分发）场景中，很多以前使用 S4 或 Storm 等流式处理引擎的实现已经逐渐被 Kafka+Spark Streaming 代替；

Hadoop 现在分三块 HDFS/MR/YARN，Spark 的流行将逐渐让 MapReduce、Tez 走进博物馆；Spark 只是作为一个计算引擎比 MR 的性能要好，但它的存储和调度框架还是依赖于 HDFS/YARN，Spark 也有自己的调度框架，但不成熟，基本不可商用。

### Spark 部署 (on Yarn)

YARN 实现了一个集群多个框架”，即在一个集群上部署一个统一的资源调度管理框架，并部署其他各种计算框架，YARN 为这些计算框架提供统一的资源调度管理服务，并且能够根据各种计算框架的负载需求调整各自占用的资源，实现集群资源共享和资源弹性收缩；

并且，YARN 实现集群上的不同应用负载混搭，有效提高了集群的利用率；不同计算框架可以共享底层存储，避免了数据集跨集群移动 ；

这里使用 Spark on Yarn 模式部署，配置 on yarn 模式只需要修改很少配置，也不用使用启动 spark 集群命令，需要提交任务时候须指定在 yarn 上。

Spark 运行需要 Scala 语言，须下载 Scala 和 Spark 并解压到家目录，设置当前用户的环境变量（~/.bash_profile），增加 SCALA_HOME 和 SPARK_HOME 路径并立即生效；启动 scala 命令和 spark-shell 命令验证是否成功；Spark 的配置文件修改按照官网教程不好理解，这里完成的配置参照博客及调试。

Spark 的需要修改两个配置文件：spark-env.sh 和 spark-default.conf，前者需要指明 Hadoop 的 hdfs 和 yarn 的配置文件路径及 Spark.master.host 地址，后者需要指明 jar 包地址；

spark-env.sh 配置文件修改如下：

\`  
export JAVA_HOME=/home/stream/jdk1.8.0_144

export SCALA_HOME=/home/stream/scala-2.11.12

export HADOOP_HOME=/home/stream/hadoop-3.0.3

export HADOOP_CONF_DIR=/home/stream/hadoop-3.0.3/etc/hadoop

export YARN_CONF_DIR=/home/stream/hadoop-3.0.3/etc/hadoop

export SPARK_MASTER_HOST=xx

export SPARK_LOCAL_IP=xxx

spark-default.conf 配置修改如下：

// 增加 jar 包地址

spark.yarn.jars=hdfs://1xxx/spark_jars/\*

\`

该设置表明将 jar 地址定义在 hdfs 上，必须将~/spark/jars 路径下所有的 jar 包都上传到 hdfs 的 / spark_jars / 路径 (hadoop hdfs –put ~/spark/jars/\*)，否则会报错无法找到编译 jar 包错误；

### Spark 启动和验证

直接无参数启动./spark-shell ，运行的是本地模式：

启动./spark-shell –master yarn，运行的是 on yarn 模式，前提是 yarn 配置成功并可用：

在 hdfs 文件系统中创建文件 README.md，并读入 RDD 中，使用 RDD 自带的参数转换，RDD 默认每行为一个值：

使用./spark-shell --master  yarn 启动 spark 后运行命令：val textFile=sc.textFile(“README.md”) 读取 hdfs 上的 README.md 文件到 RDD，并使用内置函数测试如下，说明 spark on yarn 配置成功.

### 常见问题

在启动 spark-shell 时候，报错 Yarn-site.xml 中配置的最大分配内存不足，调大这个值为 2048M，需重启 yarn 后生效。

设置的 hdfs 地址冲突，hdfs 的配置文件中 hdfs-site.xml 设置没有带端口，但是 spark-default.conf 中的 spark.yarn.jars 值带有端口，修改 spark-default.conf 的配置地址同前者一致：

### Spark 基本原理

在实际应用中，大数据处理主要包括以下三个类型：复杂的批量数据处理：通常时间跨度在数十分钟到数小时之间；基于历史数据的交互式查询：通常时间跨度在数十秒到数分钟之间；基于实时数据流的数据处理：通常时间跨度在数百毫秒到数秒之间；

同时存在以上场景需要同时部署多个组件，如: MapReduce/Impala/Storm，这样做难免会带来一些问题：不同场景之间输入输出数据无法做到无缝共享，通常需要进行数据格式的转换，不同的软件需要不同的开发和维护团队，带来了较高的使用成本，比较难以对同一个集群中的各个系统进行统一的资源协调和分配；

Spark 的设计遵循 “一个软件栈满足不同应用场景” 的理念，逐渐形成了一套完整的生态系统，其生态系统包含了 Spark Core、Spark SQL、Spark Streaming（ Structured Streaming）、MLLib 和 GraphX 等组件，既能够提供内存计算框架，也可以支持 SQL 即席查询、实时流式计算、机器学习和图计算等。

而且 Spark 可以部署在资源管理器 YARN 之上，提供一站式的大数据解决方案；因此，Spark 所提供的生态系统足以应对上述三种场景，即批处理、交互式查询和流数据处理。

### Spark 概念 / 架构设计

RDD：是 Resilient Distributed Dataset（弹性分布式数据集）的简称，是分布式内存的一个抽象概念，提供了一种高度受限的共享内存模型 ；

DAG：是 Directed Acyclic Graph（有向无环图）的简称，反映 RDD 之间的依赖关系 ；

Executor：是运行在工作节点（WorkerNode）的一个进程，负责运行 Task ；

应用（Application）：用户编写的 Spark 应用程序；

任务（ Task ）：运行在 Executor 上的工作单元 ；

作业（ Job ）：一个作业包含多个 RDD 及作用于相应 RDD 上的各种操作；

阶段（ Stage ）：是作业的基本调度单位，一个作业会分为多组任务，每组任务被称为阶段，或者也被称为任务集合，代表了一组关联的、相互之间没有 Shuffle 依赖关系的任务组成的任务集；

Spark 运行架构包括集群资源管理器（Cluster Manager）、运行作业任务的工作节点（Worker Node）、每个应用的任务控制节点（Driver）和每个工作节点上负责具体任务的执行进程（Executor），资源管理器可以自带或使用 Mesos/YARN；

一个应用由一个 Driver 和若干个作业构成，一个作业由多个阶段构成，一个阶段由多个没有 Shuffle 关系的任务组成；

当执行一个应用时，Driver 会向集群管理器申请资源，启动 Executor，并向 Executor 发送应用程序代码和文件，然后在 Executor 上执行任务，运行结束后，执行结果会返回给 Driver，或者写到 HDFS 或者其他数据库中。

### Spark 运行流程

SparkContext 对象代表了和一个集群的连接：

（1）首先为应用构建起基本的运行环境，即由 Driver 创建一个 SparkContext，进行资源的申请、任务的分配和监控；

（2）资源管理器为 Executor 分配资源，并启动 Executor 进程；

（3）SparkContext 根据 RDD 的依赖关系构建 DAG 图，DAG 图提交给 DAGScheduler 解析成 Stage，然后把一个个 TaskSet 提交给底层调度器 TaskScheduler 处理；Executor 向 SparkContext 申请 Task，Task Scheduler 将 Task 发放给 Executor 运行，并提供应用程序代码；

（4）Task 在 Executor 上运行，把执行结果反馈给 TaskScheduler，然后反馈给 DAGScheduler，运行完毕后写入数据并释放所有资源；

## Spark RDD

### RDD 概念 / 特性

许多迭代式算法（比如机器学习、图算法等）和交互式数据挖掘工具，共同之处是不同计算阶段之间会重用中间结果， MapReduce 框架把中间结果写入到稳定存储（如磁盘）中，带来大量的数据复制、磁盘 IO 和序列化开销。

RDD 就是为了满足这种需求而出现的，它提供了一个抽象的数据架构，开发者不必担心底层数据的分布式特性，只需将具体的应用逻辑表达为一系列转换处理，不同 RDD 之间的转换操作形成依赖关系，可以实现管道化，避免中间数据存储。一个 RDD 就是一个分布式对象集合，本质上是一个只读的分区记录集合，每个 RDD 可分成多个分区，每个分区就是一个数据集片段，并且一个 RDD 的不同分区可以被保存到集群中不同的节点上，从而可以在集群中的不同节点上进行并行计算。

RDD 提供了一种高度受限的共享内存模型，即 RDD 是只读的记录分区的集合，不能直接修改，只能基于稳定的物理存储中的数据集创建 RDD，或者通过在其他 RDD 上执行确定的转换操作（如 map、join 和 group by）而创建得到新的 RDD。

RDD 提供了丰富的操作以支持常见数据运算，分 “转换”（Transformation）和 “动作”（Action）两种类型；RDD 提供的转换接口都非常简单，都是类似 map、filter、groupBy、join 等粗粒度的数据转换操作，而不是针对某个数据项的细粒度修改（不适合网页爬虫），表面上 RDD 的功能很受限、不够强大，实际上 RDD 已经被实践证明可以高效地表达许多框架的编程模型（比如 MapReduce、SQL、Pregel）；Spark 用 Scala 语言实现了 RDD 的 API，程序员可以通过调用 API 实现对 RDD 的各种操作

RDD 典型的执行过程如下，这一系列处理称为一个 Lineage（血缘关系），即 DAG 拓扑排序的结果：

-   RDD 读入外部数据源进行创建；
-   RDD 经过一系列的转换（Transformation）操作，每一次都会产生不同的 RDD，供给下一个转换操作使用；
-   最后一个 RDD 经过 “动作” 操作进行转换，并输出到外部数据源**优点：惰性调用、管道化、避免同步等待、不需要保存中间结果、操作简单；**

Spark 采用 RDD 以后能够实现高效计算的原因主要在于：

（1）高容错性：血缘关系、重新计算丢失分区、无需回滚系统、重算过程在不同节点之间并行、只记录粗粒度的操作；

（2）中间结果持久化到内存：数据在内存中的多个 RDD 操作之间进行传递，避免了不必要的读写磁盘开销；

（3）存放的数据是 Java 对象：避免了不必要的对象序列化和反序列化；

### RDD 依赖关系

Spark 通过分析各个 RDD 的依赖关系生成了 DAG，并根据 RDD 依赖关系把一个作业分成多个阶段，阶段划分的依据是窄依赖和宽依赖，窄依赖可以实现流水线优化，宽依赖包含 Shuffle 过程，无法实现流水线方式处理。

窄依赖表现为一个父 RDD 的分区对应于一个子 RDD 的分区或多个父 RDD 的分区对应于一个子 RDD 的分区；宽依赖则表现为存在一个父 RDD 的一个分区对应一个子 RDD 的多个分区。

逻辑上每个 RDD 操作都是一个 fork/join（一种用于并行执行任务的框架），把计算 fork 到每个 RDD 分区，完成计算后对各个分区得到的结果进行 join 操作，然后 fork/join 下一个 RDD 操作；

RDD Stage 划分：Spark 通过分析各个 RDD 的依赖关系生成了 DAG，再通过分析各个 RDD 中的分区之间的依赖关系来决定如何划分 Stage，具体方法：

-   在 DAG 中进行反向解析，遇到宽依赖就断开；
-   遇到窄依赖就把当前的 RDD 加入到 Stage 中；
-   将窄依赖尽量划分在同一个 Stage 中，可以实现流水线计算；

### RDD 运行过程

通过上述对 RDD 概念、依赖关系和 Stage 划分的介绍，结合之前介绍的 Spark 运行基本流程，总结一下 RDD 在 Spark 架构中的运行过程：

（1）创建 RDD 对象；

（2）SparkContext 负责计算 RDD 之间的依赖关系，构建 DAG；

（3）DAG Scheduler 负责把 DAG 图分解成多个 Stage，每个 Stage 中包含了多个 Task，每个 Task 会被 TaskScheduler 分发给各个 WorkerNode 上的 Executor 去执行；

### RDD 创建

RDD 的创建可以从从文件系统中加载数据创建得到，或者通过并行集合（数组）创建 RDD。Spark 采用 textFile() 方法来从文件系统中加载数据创建 RDD，该方法把文件的 URI 作为参数，这个 URI 可以是本地文件系统的地址，或者是分布式文件系统 HDFS 的地址；

从文件系统中加载数据：

`scala> val lines = sc.textFile("file:///usr/local/spark/mycode/rdd/word.txt")  
`

从 HDFS 中加载数据：

`scala> val lines = sc.textFile("hdfs://localhost:9000/user/hadoop/word.txt")  
`

可以调用 SparkContext 的 parallelize 方法，在 Driver 中一个已经存在的集合（数组）上创建。

\`scala>val array = Array(1,2,3,4,5)

scala>val rdd = sc.parallelize(array)

\`

或者从列表中创建：

\`scala>val list = List(1,2,3,4,5)

scala>val rdd = sc.parallelize(list)

\`

### RDD 操作

对于 RDD 而言，每一次转换操作都会产生不同的 RDD，供给下一个 “转换” 使用，转换得到的 RDD 是惰性求值的，也就是说，整个转换过程只是记录了转换的轨迹，并不会发生真正的计算，只有遇到行动操作时，才会发生真正的计算，开始从血缘关系源头开始，进行物理的转换操作；

常用的 RDD 转换操作，总结如下：

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mqKD9ibibfMVa8oOg0EUC1lnNb4tnKy7NmptFZfTpKFv6V6Dnm2jf4uyw/640?wx_fmt=png)
filter(func) 操作：筛选出满足函数 func 的元素，并返回一个新的数据集

\`scala>  val  lines =sc.textFile(file:///usr/local/spark/mycode/rdd/word.txt)

scala>  val  linesWithSpark=lines.filter(line => line.contains("Spark"))

\`

map(func) 操作：map(func) 操作将每个元素传递到函数 func 中，并将结果返回为一个新的数据集

\`scala> data=Array(1,2,3,4,5)

scala> val  rdd1= sc.parallelize(data)

scala> val  rdd2=rdd1.map(x=>x+10)

\`

另一个实例：

\`scala> val  lines = sc.textFile("file:///usr/local/spark/mycode/rdd/word.txt")

scala> val  words=lines.map(line => line.split(" "))

\`

flatMap(func) 操作：拍扁操作

\`scala> val  lines = sc.textFile("file:///usr/local/spark/mycode/rdd/word.txt")

scala> val  words=lines.flatMap(line => line.split(" "))

\`

groupByKey()操作：应用于 (K,V) 键值对的数据集时，返回一个新的 (K, Iterable) 形式的数据集；

reduceByKey(func)操作：应用于 (K,V) 键值对的数据集返回新 (K, V) 形式数据集，其中每个值是将每个 key 传递到函数 func 中进行聚合后得到的结果：

行动操作是真正触发计算的地方。Spark 程序执行到行动操作时，才会执行真正的计算，这就是惰性机制，“惰性机制”是指，整个转换过程只是记录了转换的轨迹，并不会发生真正的计算，只有遇到行动操作时，才会触发 “从头到尾” 的真正的计算，常用的行动操作：

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mH30Za5gT2x1ICyCZZfATCcL8hevia0Aguaygm6icTlCPf1sOhL6xCwlQ/640?wx_fmt=png)

### RDD 持久

Spark RDD 采用惰性求值的机制，但是每次遇到行动操作都会从头开始执行计算，每次调用行动操作都会触发一次从头开始的计算，这对于迭代计算而言代价是很大的，迭代计算经常需要多次重复使用同一组数据：

\`scala> val  list = List("Hadoop","Spark","Hive")

scala> val  rdd = sc.parallelize(list)

scala> println(rdd.count())  // 行动操作，触发一次真正从头到尾的计算

scala> println(rdd.collect().mkString(","))  // 行动操作，触发一次真正从头到尾的计算

\`

可以通过持久化（缓存）机制避免这种重复计算的开销，可以使用 persist()方法对一个 RDD 标记为持久化，之所以说 “标记为持久化”，是因为出现 persist() 语句的地方，并不会马上计算生成 RDD 并把它持久化，而是要等到遇到第一个行动操作触发真正计算以后，才会把计算结果进行持久化，持久化后的 RDD 将会被保留在计算节点的内存中被后面的行动操作重复使用；

persist() 的圆括号中包含的是持久化级别参数，persist(MEMORY_ONLY) 表示将 RDD 作为反序列化的对象存储于 JVM 中，如果内存不足，就要按照 LRU 原则替换缓存中的内容；persist(MEMORY_AND_DISK) 表示将 RDD 作为反序列化的对象存储在 JVM 中，如果内存不足，超出的分区将会被存放在硬盘上；一般而言，使用 cache() 方法时，会调用 persist(MEMORY_ONLY)，同时可以使用 unpersist() 方法手动地把持久化的 RDD 从缓存中移除。

针对上面的实例，增加持久化语句以后的执行过程如下：

\`scala> val  list = List("Hadoop","Spark","Hive")

scala> val  rdd = sc.parallelize(list)

scala> rdd.cache()  // 会调用 persist(MEMORY_ONLY)，但是，语句执行到这里，并不会缓存 rdd，因为这时 rdd 还没有被计算生成

scala> println(rdd.count()) // 第一次行动操作，触发一次真正从头到尾的计算，这时上面的 rdd.cache() 才会被执行，把这个 rdd 放到缓存中

scala> println(rdd.collect().mkString(",")) // 第二次行动操作，不需要触发从头到尾的计算，只需要重复使用上面缓存中的 rdd

\`

### RDD 分区

RDD 是弹性分布式数据集，通常 RDD 很大，会被分成很多个分区分别保存在不同的节点上，分区的作用：（1）增加并行度（2）减少通信开销。RDD 分区原则是使得分区的个数尽量等于集群中的 CPU 核心（core）数目，对于不同的 Spark 部署模式而言（本地模式、Standalone 模式、YARN 模式、Mesos 模式），都可以通过设置 spark.default.parallelism 这个参数的值，来配置默认的分区数目，一般而言：

本地模式：默认为本地机器的 CPU 数目，若设置了 local\[N], 则默认为 N；

Standalone 或 YARN：在 “集群中所有 CPU 核心数目总和” 和 2 二者中取较大值作为默认值；

设置分区的个数有两种方法：创建 RDD 时手动指定分区个数，使用 reparititon 方法重新设置分区个数；

创建 RDD 时手动指定分区个数：在调用 textFile() 和 parallelize() 方法的时候手动指定分区个数即可，语法格式如 sc.textFile(path, partitionNum)，其中 path 参数用于指定要加载的文件的地址，partitionNum 参数用于指定分区个数。

\`scala> val  array = Array(1,2,3,4,5)

scala> val  rdd = sc.parallelize(array,2)  // 设置两个分区

\`

reparititon 方法重新设置分区个数：通过转换操作得到新 RDD 时，直接调用 repartition 方法即可，如：

\`scala> val  data = sc.textFile("file:///usr/local/spark/mycode/rdd/word.txt",2)

scala> data.partitions.size  // 显示 data 这个 RDD 的分区数量

scala> val  rdd = data.repartition(1)  // 对 data 这个 RDD 进行重新分区

scala> rdd.partitions.size

res4: Int = 1

\`

## Spark-shell 批处理

完成 Spark 部署后，使用 spark-shell 指令进入 Scala 交互编程界面，spark-shell 默认创建一个 sparkContext（sc），在 spark-shell 启动时候可以查看运行模式是 on yarn 还是 local 模式，使用交互式界面可以直接引用 sc 变量使用；

使用 Spark-shell 处理数据实例：读取 HDFS 文件系统中文件实现 WordCount 单词计数：

`sc.textFile("hdfs://172.22.241.183:8020/user/spark/yzg_test.txt").flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_).collect()  
`

其中，map((\_,1)) 等同于 map(x => (x, 1))

使用 saveAsText File() 函数可以将结果保存到文件系统中。

## Scala 及函数式编程

Spark 采用 Scala 语言编写，在开发中需要熟悉函数式编程思想，并熟练使用 Scala 语言，使用 Scala 进行 Spark 开发的代码量大大少于 Java 开发的代码量；

### 函数式编程特性

函数式编程属于声明式编程的一种，将计算描述为数学函数的求值，但函数式编程没有准确的定义，只是一系列理念，并不需要严格准守，可以理解为函数式编程把程序看做是数学函数，输入的是自变量，输出因变量，通过表达式完成计算，当前越来越多的命令式语言支持部分的函数式编程特性。

在函数式编程中，函数作为一等公民，就是说函数的行为和普通变量没有区别，可以作为函参进行传递，也可以在函数内部声明一个函数，那么外层的函数就被称作高阶函数。

函数式编程的 curry 化：把接受多个参数的函数变换成接受一个单一参数的函数，返回接受余下的参数并且返回结果的新函数。

函数式编程要求所有的变量都是常量（这里所用的变量这个词并不准确，只是为了便于理解），erlang 是其中的典型语言，虽然许多语言支持部分函数式编程的特性，但是并不要求变量必须是常量。这样的特性提高了编程的复杂度，但是使代码没有副作用，并且带来了很大的一个好处，那就是大大简化了并发编程。

Java 中最常用的并发模式是共享内存模型，依赖于线程与锁，若代码编写不当，会发生死锁和竞争条件，并且随着线程数的增加，会占用大量的系统资源。在函数式编程中，因为都是常量，所以根本就不用考虑死锁等情况。为什么说一次赋值提高了编程的复杂度，既然所有变量都是常量，那么我们没办法更改一个变量的值，循环的意义也就不大，所以 haskell 与 erlang 中使用递归代替了循环。

### Scala 语法

Scala 即可伸缩的语言（Scalable Language），是一种多范式的编程语言，类似于 java 的编程，设计初衷是要集成面向对象编程和函数式编程的各种特性。

### Scala 函数地位：一等公民

在 Scala 中函数是一等公民，像变量一样既可以作为函参使用，也可以将函数赋值给一个变量；而且函数的创建不用依赖于类、或对象，在 Java 中函数的创建则要依赖于类、抽象类或者接口。Scala 函数有两种定义：

Scala 的函数定义规范化写法，最后一行代码是它的返回值：![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mgvDJUCu3f9b2X8r2ibCnBYhn5ribyWu5yB8zUkyhspLuQcgPbCYmH5vg/640?wx_fmt=png)

精简后函数定义可以只有一行：![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mV4rrlA6dfa7ekhoA0MSRo46OAb1TYygbRIibATpVwgUhaQJ8Jvxlbibg/640?wx_fmt=png)
也可以直接使用 val 将函数定义成变量，表示定义函数 addInt, 输入参数有两个，分别为 x,y, 均为 Int 类型，返回值为两者的和，类型 Int：

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mPKVNrO1xpOwcecsoeNpe7Yu3zr89p6yxjTDXQ4aibibEH2zoexgaWFeg/640?wx_fmt=png)

### Scala 匿名函数 (函数字面量)

Scala 中的匿名函数也叫做函数字面量，既可以作为函数的参数使用，也可以将其赋值给一个变量，在匿名函数的定义中 “=>” 可理解为一个转换器，它使用右侧的算法，将左侧的输入数据转换为新的输出数据，使用匿名函数后，我们的代码变得更简洁了。

`val test = (x:Int) => x + 1  
`

### Scala 高阶函数

Scala 使用术语 “高阶函数” 来表示那些把函数作为参数或函数作为返回结果的方法和函数。比如常见的有 map,filter,reduce 等函数，它们可以接受一个函数作为参数。

### Scala 闭包

Scala 中的闭包指的是当函数的变量超出它的有效作用域的时候，还能对函数内部的变量进行访问；Scala 中的闭包捕获到的是变量的本身而不仅仅是变量的数值，当自由变量发生变化时，Scala 中的闭包能够捕获到这个变化；如果自由变量在闭包内部发生变化，也会反映到函数外面定义的自由变量的数值。

### Scala 部分应用函数

部分应用函数只是在 “已有函数” 的基础上，提供部分默认参数，未提供默认参数的地方使用下划线替代，从而创建出一个“函数值”，在使用这个函数值（部分应用函数）的时候，只需提供下划线部分对应的参数即可；部分应用函数本质上是一种值类型的表达式, 在使用的时候不需要提供所有的参数, 只需要提供部分参数。

### Scala 柯里化函数

scala 中的柯里化指的是将原来接受两个参数的函数变成新的接受一个参数的函数的过程，新的函数返回一个以原有第二个参数作为参数的函数；

`def someAction(f:(Double)=>Double) = f(10)  
`

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mcQve4d9KQA2uIWbARFKzLyqDuYiaLbBeaHjkbyoG8BnyAaUHMryS1lg/640?wx_fmt=png)
只要满足：函数参数是一个 double、返回值也是一个 double，这个函数就可以作为 f 值；

## Spark SQL

### Shark 和 Spark SQL

Shark 的出现，使得 SQL-on-Hadoop 的性能比 Hive 有了 10-100 倍的提高, 但 Shark 的设计导致了两个问题：

-   一是执行计划优化完全依赖于 Hive，不方便添加新的优化策略
-   二是因为 Spark 是线程级并行，而 MapReduce 是进程级并行，因此，Spark 在兼容 Hive 的实现上存在线程安全问题，导致 Shark 不得不使用另外一套独立维护的打了补丁的 Hive 源码分支 ；

Spark SQL 在 Hive 兼容层面仅依赖 HiveQL 解析、Hive 元数据，也就是说，从 HQL 被解析成抽象语法树（AST）起，就全部由 Spark SQL 接管了。Spark SQL 执行计划生成和优化都由 Catalyst（函数式关系查询优化框架）负责 ；

### DataFrame 和 RDD

Spark SQL 增加了 DataFrame（即带有 Schema 信息的 RDD），使用户可以在 Spark SQL 中执行 SQL 语句，数据既可以来自 RDD，也可以是 Hive、HDFS、Cassandra 等外部数据源，还可以是 JSON 格式的数据，Spark SQL 目前支持 Scala、Java、Python 三种语言，支持 SQL-92 规范 ；

-   DataFrame 的推出，让 Spark 具备了处理大规模结构化数据的能力，不仅比原有的 RDD 转化方式更加简单易用，且获得了更高的计算性能；
-   Spark 可轻松实现从 MySQL 到 DataFrame 的转化，且支持 SQL 查询；![](https://mmbiz.qpic.cn/mmbiz_jpg/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mEiafM2VAMndibcV5Ve8USa4q6tot1EvZc8xZApYGamIJz4Uukgn4NAuw/640?wx_fmt=jpeg)
    RDD 是分布式的 Java 对象的集合，但是，对象内部结构对于 RDD 而言却是不可知的；DataFrame 是一种以 RDD 为基础的分布式数据集，提供了详细的结构信息。

RDD 就像一个屋子，找东西要把这个屋子翻遍才能找到；DataFrame 相当于在你的屋子里面打上了货架，只要告诉他你是在第几个货架的第几个位置， DataFrame 就是在 RDD 基础上加入了列，处理数据就像处理二维表一样。

DataFrame 与 RDD 的主要区别在于，前者带 schema 元信息，即 DataFrame 表示的二维表数据集的每一列都带有名称和类型。这使得 Spark SQL 得以洞察更多的结构信息，从而对藏于 DataFrame 背后的数据源以及作用于 DataFrame 之上的变换进行了针对性的优化，最终达到大幅提升运行时效率的目标。反观 RDD，由于无从得知所存数据元素的具体内部结构，Spark Core 只能在 stage 层面进行简单、通用的流水线优化。

### DataFrame 的创建

Spark2.0 版本开始，Spark 使用全新的 SparkSession 接口替代 Spark1.6 中的 SQLContext 及 HiveContext 接口来实现其对数据加载、转换、处理等功能。SparkSession 实现了 SQLContext 及 HiveContext 所有功能；

SparkSession 支持从不同的数据源加载数据，并把数据转换成 DataFrame，支持把 DataFrame 转换成 SQLContext 自身中的表，然后使用 SQL 语句来操作数据。SparkSession 亦提供了 HiveQL 以及其他依赖于 Hive 的功能的支持；可以通过如下语句创建一个 SparkSession 对象：

\`scala> import org.apache.spark.sql.SparkSession

scala> val spark=SparkSession.builder().getOrCreate()

\`

在创建 DataFrame 前，为支持 RDD 转换为 DataFrame 及后续的 SQL 操作，需通过 import 语句（即 import spark.implicits.\_）导入相应包，启用隐式转换。

在创建 DataFrame 时，可使用 spark.read 操作从不同类型的文件中加载数据创建 DataFrame，如：spark.read.json("people.json")：读取 people.json 文件创建 DataFrame；在读取本地文件或 HDFS 文件时，要注意给出正确的文件路径；spark.read.csv("people.csv")：读取 people.csv 文件创建 DataFrame；

读取 hdfs 上的 json 文件，并打印，json 文件为：

\`{"name":"Michael"}

{"name":"Andy", "age":30}

{"name":"Justin", "age":19}

\`

读取代码：

\`import org.apache.spark.sql.SparkSession

val spark=SparkSession.builder().getOrCreate()

import spark.implicits.\_

val df =spark.read.json("hdfs://172.22.241.183:8020/user/spark/json_sparksql.json")

df.show()

\`

### RDD 转换 DataFrame

Spark 官网提供了两种方法来实现从 RDD 转换得到 DataFrame：① 利用反射来推断包含特定类型对象的 RDD 的 schema，适用对已知数据结构的 RDD 转换；② 使用编程接口，构造一个 schema 并将其应用在已知的 RDD 上；

### Spark-sql 即席查询

SparkSQL 的元数据的状态有两种：①  in_memory，用完了元数据也就丢了；② 通过 hive 保存，hive 的元数据存在哪儿，它的元数据也就存在哪，SparkSQL 数据仓库建立在 Hive 之上实现的，使用 SparkSQL 去构建数据仓库的时候，必须依赖于 Hive。

Spark-sql 命令行提供了即席查询能力，可以使用类 sql 方式操作数据源，效率高于 hive，常用语句：[https://www.cnblogs.com/BlueSkyyj/p/9640626.html；](https://www.cnblogs.com/BlueSkyyj/p/9640626.html；)

spark-sql 导入数据到数仓：[https://www.cnblogs.com/chenfool/p/4502212.html；](https://www.cnblogs.com/chenfool/p/4502212.html；)

## Spark Streaming

Spark Streaming 是 Spark Core 扩展而来的一个高吞吐、高容错的实时处理引擎，同 Storm 的最大区别在于无法实现毫秒级计算，而 Storm 可以实现毫秒级响应，Spark Streaming 实现方式是批量计算，按照时间片对 stream 切割形成静态数据，并且基于 RDD 数据集更容易做高效的容错处理。Spark Streaming 的输入和输出数据源可以是多种。Spark  Streaming 实时读取数据并将数据分为小批量的 batch，然后在 spark 引擎中处理生成批量的结果集。Spark Streaming 提供了称为离散流或 DStream 的高级抽象概念，它表示连续的数据流。DStreams 既可以从 Kafka、Flume 等源的输入数据流创建，也可以通过在其他 DStreams 上应用高级操作创建。在内部 DStream 表示为 RDD 序列。

在这里从一个例子开始介绍，StreamingContext 是所有的流式计算的主要实体，创建含有两个执行线程的本地 StreamingContext 和 1 秒钟的 batch，然后创建一个 Dstream（lines）用于监听 TCP 端口，lines 中的每一行就是一个 RDD，flatMap 函数将一个 RDD 分解成多个记录，是一对多的 Dstream 操作，这里使用空格将 lines 分解成单词，words 被映射成 (word, 1) 对，随后进行词频统计，例子的代码如下：

\`import org.apache.spark.\_

import org.apache.spark.streaming.\_

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")

val ssc = new StreamingContext(conf, Seconds(1))

val lines = ssc.socketTextStream("localhost", 9999)

val words = lines.flatMap(\_.split(" "))

val pairs = words.map(word => (word, 1))

val wordCounts = pairs.reduceByKey(_ + _)

wordCounts.print()

ssc.start()

ssc.awaitTermination()

\`

### Streaming 原理

可以参考官网教程：[http://spark.apache.org/docs/latest/streaming-programming-guide.html，Spark](http://spark.apache.org/docs/latest/streaming-programming-guide.html，Spark) Streaming 提供了称为离散流或 DStream 的高级抽象，它表示连续的数据流，在内部 DStream 表示为 RDD 序列，每个 RDD 包含一定间隔的数据，如下图所示：

所有对于 DStream 的操作都会相应地转换成对 RDDs 的操作，在上面的例子中，flatMap 操作被应用到 lines 中的每个 RDD 中生成了一组 RDD（即 words）

总结编写 Spark Streaming 程序的基本步骤是：

1\. 通过创建输入 DStream 来定义输入源

2\. 通过对 DStream 应用转换操作和输出操作来定义流计算

3\. 用 streamingContext.start() 来开始接收数据和处理流程

4\. 通过 streamingContext.awaitTermination() 方法来等待处理结束（手动结束或因为错误而结束）

5\. 可以通过 streamingContext.stop() 来手动结束流计算进程

### StreamingContext

有两种创建 StreamingContext 的方式：通过 SparkContext 创建和通过 SparkConf 创建；

Spark conf 创建：

\`val conf = new SparkConf().setAppName(appName).setMaster(master);

val ssc = new StreamingContext(conf, Seconds(1));

\`

appName 是用来在 Spark UI 上显示的应用名称。master 是 Spark、Mesos 或 Yarn 集群的 URL，或者是 local\[\*]。batch interval 可以根据你的应用程序的延迟要求以及可用的集群资源情况来设置。

SparkContext 创建：

\`val sc = new SparkContext(conf)

val ssc = new StreamingContext(sc, Seconds(1))

\`

### 输入 DStreams 和 Receiver

在前面的例子中 lines 就是从源得到的输入 DStream，输入 DStream 对应一个接收器对象，可以从源接收消息并存储到 Spark 内存中进行处理。Spark Streaming 提供两种 streaming 源：

-   基础源：直接可以使用 streaming 上下文 API 的源，比如 files 和 socket；
-   高级源：通过引用额外实体类得到的 Kafka，Flume 源；可以在应用中创建使用多个输入 DStreams 来实现同时读取多种数据流，worker/executor 是持久运行的任务，因此它将占用一个分给该应用的 core，因此 Spark Streaming 需要分配足够的 core 去运行接收器和处理接收的数据；

在本地运行 Spark Streaming 程序时，不要使用 “local” 或“local\[1]” 作为主 URL。这两者中的任何一个都意味着在本地运行任务只使用一个线程。如果使用基于 receiver 的输入 DStream（如 Kafka、Flume 等），这表明将使用单个线程运行 receiver，而不留下用于处理所接收数据的线程。因此在本地运行时，始终使用 “local\[n]” 作为主 URL，其中 n 必须大于运行的 receiver 数量，否则系统将接收数据，但不能处理它。

Kafka 和 Flume 这类源需要外部依赖包，其中一些库具有复杂的依赖关系，Spark shell 中没有这些高级源代码，因此无法在 spark-shell 中测试基于这些高级源代码的应用程序，但可以手动将包引入；

基于可靠性的考虑，可以将数据源分为两类：可靠的接收器的数据被 Receiver 接收后发送确认到源头（如 Kafka ,Flume）并将数据存储在 spark，不可靠的接收器不会向源发送确认。

### DStreams 转换

与 RDD 类似，转换操作允许修改来自输入 DStream 的数据，转换操作包括无状态转换操作和有状态转换操作。

无状态转换操作实例：下节 spark-shell 中 “套接字流” 词频统计采用无状态转换，每次统计都只统计当前批次到达的单词的词频，和之前批次无关，不会进行累计。

有状态转换操作实例：滑动窗口转换操作和 updateStateByKey 操作；

一些常见的转换如下：

### 窗口操作

每次窗口在源 DStream 上滑动，窗口内的源 RDD 被组合 / 操作生成了窗口 RDD，在图例中，过去 3 个时间单位的数据将被操作，并按 2 个时间单位滑动。

任何窗口操作都需要指定两个参数：窗口长度：窗口的持续时间（图中值是 3）；滑动间隔：执行窗口操作的间隔（图中值是 2）。这两个参数必须是源 DStream 的批处理间隔的倍数（图中值是 1）

举例说明窗口操作：希望通过每隔 10 秒在最近 30 秒数据中生成字数来扩展前面的示例。为此，我们必须在最近的 30 秒数据上对（word,1）的 DStream 键值对应用 reduceByKey 操作。这是使用 reduceByKeyAndWindow 操作完成的。

`val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))  
`

所有的滑动窗口操作都需要使用参数：windowLength（窗口长度）和 slideInterval（滑动间隔），常见窗口操作总结如下，对应的含义可参照 RDD 的转换操作：

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3moFBT3tlT2JqdCXAxpUYAo7DaMIqoRgUKuNyBcWsrTez4yOTdsoz79w/640?wx_fmt=png)

Window：基于源 DStream 产生的窗口化的批数据计算得到新的 Dstream；

countByWindow: 返回 DStream 中元素的滑动窗口计数；

reduceByWindow：返回一个单元素流。利用函数 func 聚集滑动时间间隔的流的元素创建这个单元素流。函数 func 必须满足结合律从而支持并行计算；

reduceByKeyAndWindow(三参数)：应用到一个 (K,V) 键值对组成的 DStream 上时，会返回一个由 (K,V) 键值对组成的新的 DStream。每一个 key 的值均由给定的 reduce 函数 (func 函数) 进行聚合计算。注意：在默认情况下，这个算子利用了 Spark 默认的并发任务数去分组。可以通过 numTasks 参数的设置来指定不同的任务数；

reduceByKeyAndWindow(四参数)：比上述 reduceByKeyAndWindow（三参数）更高效的 reduceByKeyAndWindow，每个窗口的 reduce 值是基于先前窗口的 reduce 值进行增量计算得到的；它会对进入滑动窗口的新数据进行 reduce 操作，并对离开窗口的老数据进行 “逆向 reduce” 操作。但是，只能用于“可逆 reduce 函数”，即那些 reduce 函数都有一个对应的“逆向 reduce 函数”（以 InvFunc 参数传入）。

countByValueAndWindow：当应用到一个 (K,V) 键值对组成的 DStream 上，返回一个由 (K,V) 键值对组成的新的 DStream。每个 key 的值都是它们在滑动窗口中出现的频率。

updateStateByKey：需要在跨批次之间维护状态时，必须使用 updateStateByKey 操作；

### 多流关联

窗口计算上 join 操作非常有用，在 Spark Streaming 中可以轻松实现不同类型的 join，包括 leftouterjoin、rightouterjoin 和 fulloterjoin。每个批处理间隔中 stream1 生成的 RDD 与 stream2 生成的 RDD 关联如下：

\`val stream1: DStream[String, String] = ...

val stream2: DStream[String, String] = ...

val joinedStream = stream1.join(stream2)

\`

### Dstream 的输出

输出操作允许将 DStream 的数据推送到外部系统，如数据库或 files，由于输出操作触发所有 DStream 转换的实际执行（类似于 RDD 的操作），并允许外部系统使用转换后的数据，输出操作有以下几种：

![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mBOgPmZxiaKQJFV5Z0ibniaMnNZtBv8OW2DMZN2ZuWJNtcCLibVP0ZYmSaA/640?wx_fmt=png)
在输出 DStream 中，Dstream.foreachRDD 是一个功能强大的原语.

### DataFrame 和 SQL 操作

可以轻松地对流数据使用 DataFrames 和 SQL 操作，但必须使用 StreamingContext 正在用的 SparkContext 创建 SparkSession。下面例子使用 DataFrames 和 SQL 生成单词计数。每个 RDD 都转换为 DataFrame，注册为临时表后使用 SQL 进行查询：

\`val words: DStream[String] =

words.foreachRDD {rdd =>

val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()

import spark.implicits.\_

val wordsDataFrame = rdd.toDF("word")

wordsDataFrame.createOrReplaceTempView("words")

val wordCountsDataFrame = spark.sql("select word, count(\*) as total from words group by word")

wordCountsDataFrame.show()

}

\`

### Spark-shell 流处理

进入 spark-shell 后就默认获得了的 SparkConext，即 sc，从 SparkConf 对象创建 StreamingContext 对象，spark-shell 中创建 StreamingContext 对象如下：

\`scala> import org.apache.spark.streaming.\_

scala> val ssc = new StreamingContext(sc, Seconds(1))

\`

如果是编写一个独立的 Spark Streaming 程序，而不是在 spark-shell 中运行，则需要通过如下方式创建 StreamingContext 对象：

\`import org.apache.spark.\_

import org.apache.spark.streaming.\_

val conf = new SparkConf().setAppName("TestDStream").setMaster("local[2]")

val ssc = new StreamingContext(conf, Seconds(1))

\`

### 文件流

文件流可以读取本机文件，也可以读取读取 HDFS 上文件，如果部署的 on yarn 模式的 Spark，则启动 spark-shell 默认读取 HDFS 上对应的: hdfs:xxxx/user/xx/ 下的文件；

\`scala> import org.apache.spark.streaming.\_

scala> val ssc = new StreamingContext(sc, Seconds(5))

scala> val lines = ssc.textFileStream("hdfs://xxx/yzg_test.txt")

scala> val Counts = lines.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)

scala> Counts.saveAsTextFiles("hdfs://xxx/bendi"))

scala> ssc.start()

scala> ssc.awaitTermination()

scala> ssc.stop()

\`

以上代码在 spark-shell 中运行后，每隔 5 秒读取 hdfs 上的文件并进行词频统计后写入到 hdfs 中的 “bendi - 时间戳” 文件夹下，直到 ssc.stop()；Counts.saveAsTextFiles("file://xxx/bendi"))和 Counts.print 分别写本地和 std 输出；

### Socket 套接字流

Spark Streaming 可以通过 Socket 端口实时监听并接收数据计算，步骤如下：

driver 端创建 StreamingContext 对象，启动上下文时依次创建 JobScheduler 和 ReceiverTracker，并调用他们的 start 方法。ReceiverTracker 在 start 方法中发送启动接收器消息给远程 Executor，消息内部含有 ServerSocket 的地址信息。在 executor 一侧，由 Receiver TrackerEndpoint 终端接受消息，抽取消息内容，利用 sparkContext 结合消息内容创建 ReceiverRDD 对象，最后提交 rdd 给 spark 集群。在代码实现上，使用 nc –lk 9999 开启 地址 172.22.241.184 主机的 9999 监听端口，并持续往里面写数据；使用 spark-shell 实现监听端口代码如下，输入源为 socket 源，进行简单的词频统计后，统计结果输出到 HDFS 文件系统；

\`scala> import org.apache.spark.\_

scala> import org.apache.spark.streaming.\_

scala> import org.apache.spark.storage.StorageLevel

scala> val ssc = new StreamingContext(sc, Seconds(5))

scala> val lines = ssc.socketTextStream("172.22.241.184", 9999, StorageLevel.MEMORY_AND_DISK_SER)

scala> val wordCounts = lines.flatMap(_.split(" ")).map((_,1)).reduceByKey(_ + _)

scala> wordCounts.saveAsTextFiles("hdfs://xxx/bendi-socket"))

scala> ssc.start()

scala> ssc.awaitTermination()

scala> ssc.stop()

\`

### Kafka 流（窗口）

Kafka 和 Flume 等高级输入源需要依赖独立的库（jar 文件），如果使用 spark-shell 读取 kafka 等高级输入源，需要将对应的依赖 jar 包放在 spark 的依赖文件夹 lib 下。

根据当前使用的 kafka 版本，适配所需要的 spark-streaming-kafka 依赖的版本，在 maven 仓库下载，地址如下：[https://mvnrepository.com/artifact/org.apache.spark/spark-streaming-kafka_2.10/1.2.1](https://mvnrepository.com/artifact/org.apache.spark/spark-streaming-kafka_2.10/1.2.1)

将对应的依赖 jar 包放在 CDH 的 spark 的依赖文件夹 lib 下，通过引入包内依赖验证是否成功：

\`  
scala> import org.apache.spark.\_

scala> import org.apache.spark.streaming.\_

scala> import org.apache.spark.streaming.kafka.\_

scala> val ssc = new StreamingContext(sc, Seconds(5))

scala> ssc.checkpoint("hdfs://usr/spark/kafka/checkpoint")

scala> val zkQuorum = "172.22.241.186:2181"

scala> val group = "test-consumer-group"

scala> val topics = "yzg_spark"

scala> val numThreads = 1

scala> val topicMap =topics.split(",").map((\_,numThreads.toInt)).toMap

scala> val lineMap = KafkaUtils.createStream(ssc,zkQuorum,group,topicMap)

scala> val pair = lineMap.map(_.\_2).flatMap(_.split(" ")).map((\_,1))

scala> val wordCounts = pair.reduceByKeyAndWindow(_ + _,_ -_,Minutes(2),Seconds(10),2)

scala> wordCounts.print

scala> ssc.start

\`

### updateStateByKey 操作

当 Spark Streaming 需要跨批次间维护状态时，就必须使用 updateStateByKey 操作。以词频统计为例，对于有状态转换操作而言，当前批次的词频统计是在之前批次的词频统计结果的基础上进行不断累加，所以最终统计得到的词频是所有批次的单词总的词频统计结果。

\`val updateFunc = (values: Seq[Int], state: Option[Int]) => {  
val currentCount = values.foldLeft(0)(_ + _)

val previousCount = state.getOrElse(0)

Some(currentCount + previousCount)

}

\`

实现：

\`import org.apache.spark.\_

import org.apache.spark.streaming.\_

import org.apache.spark.storage.StorageLevel

val ssc = new StreamingContext(sc, Seconds(5))

ssc.checkpoint("hdfs:172.22.241.184:8020//usr/spark/checkpoint")

val lines = ssc.socketTextStream("172.22.241.184", 9999, StorageLevel.MEMORY_AND_DISK_SER)

val wordCounts = lines.flatMap(_.split(" ")).map((_,1)).updateStateByKey[Int](updateFunc)

wordCounts.saveAsTextFiles("hdfs:172.22.241.184:8020//user/spark/bendi-socket")

ssc.start()

ssc.awaitTermination()

ssc.stop()

\`

## Streaming 同 Kafka 交互

### Dstream 创建

关于 SparkStreaming 实时计算框架实时地读取 kafka 中的数据然后进行计算，在 spark1.3 版本后 kafkaUtils 提供两种 Dstream 创建方法，一种为 KafkaUtils.createDstream，另一种为 KafkaUtils.createDirectStream。

KafkaUtils.createDstream 方式

其构造函数为 KafkaUtils.createDstream(ssc,\[zk], \[consumer group id], \[per-topic,partitions] )，使用 receivers 来接收数据，利用的是 Kafka 高层次的消费者 api，对于所有的 receivers 接收到的数据将会保存在 Spark executors 中，然后通过 Spark Streaming 启动 job 来处理这些数据，默认会丢失，可启用 WAL 日志，它同步将接受到数据保存到分布式文件系统上比如 HDFS。所以数据在出错的情况下可以恢复出来。![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mib7OUvyA6EARvvp0ChZgpQgln6hZCj5nmhUTzFF5zDvPIJLibTdVzQTA/640?wx_fmt=png)
A、创建一个 receiver 来对 kafka 进行定时拉取数据，ssc 的 RDD 分区和 Kafka 的 topic 分区不是一个概念，故如果增加特定主消费的线程数仅仅是增加一个 receiver 中消费 topic 的线程数，并不增加 spark 的并行处理数据数量。

B、对于不同的 group 和 topic 可以使用多个 receivers 创建不同的 DStream

C、如果启用了 WAL(spark.streaming.receiver.writeAheadLog.enable=true)

同时需要设置存储级别 (默认 StorageLevel.MEMORY_AND_DISK_SER_2)，即 KafkaUtils.createStream(….,StorageLevel.MEMORY_AND_DISK_SER)

KafkaUtils.createDirectStream 方式

在 spark1.3 之后，引入了 Direct 方式，不同于 Receiver 的方式，Direct 方式没有 receiver 这一层，其会周期性的获取 Kafka 中每个 topic 的每个 partition 中的最新 offsets，之后根据设定的 maxRatePerPartition 来处理每个 batch。如图：![](https://mmbiz.qpic.cn/mmbiz_png/z2DApiaibzMicicOqmeSH8neNob7fheLeK3mNbXgZeJJtTicIrrrcvbjCNYLicoOiak6hMbKbT5dls4YU3g7ChrTxHuYA/640?wx_fmt=png)
这种方法相较于 Receiver 方式的优势在于：

简化的并行：在 Receiver 的方式中我们提到创建多个 Receiver 之后利用 union 来合并成一个 Dstream 的方式提高数据传输并行度。而在 Direct 方式中，Kafka 中的 partition 与 RDD 中的 partition 是一一对应的并行读取 Kafka 数据，这种映射关系也更利于理解和优化。

高效：在 Receiver 的方式中，为了达到 0 数据丢失需要将数据存入 Write Ahead Log 中，这样在 Kafka 和日志中就保存了两份数据，浪费！而第二种方式不存在这个问题，只要我们 Kafka 的数据保留时间足够长，我们都能够从 Kafka 进行数据恢复。

精确一次：在 Receiver 的方式中，使用的是 Kafka 的高阶 API 接口从 Zookeeper 中获取 offset 值，这也是传统的从 Kafka 中读取数据的方式，但由于 Spark Streaming 消费的数据和 Zookeeper 中记录的 offset 不同步，这种方式偶尔会造成数据重复消费。而第二种方式，直接使用了简单的低阶 Kafka API，Offsets 则利用 Spark Streaming 的 checkpoints 进行记录，消除了这种不一致性。

此方法缺点是它不会更新 Zookeeper 中的偏移量，因此基于 Zookeeper 的 Kafka 监视工具将不会显示进度。但是您可以在每个批处理中访问此方法处理的偏移量，并自行更新 Zookeeper。

### 位置策略

参照官方的 API 文档地址：[http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html，位置策略是用来控制特定的主题分区在哪个执行器上消费的，在 executor 针对主题分区如何对消费者进行调度，并且位置的选择是相对的，位置策略有三种方案：](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html，位置策略是用来控制特定的主题分区在哪个执行器上消费的，在executor针对主题分区如何对消费者进行调度，并且位置的选择是相对的，位置策略有三种方案：)

1、PreferBrokers：首选 kafka 服务器，只有在 kafka 服务器和 executor 位于同一主机可以使用该策略。

2、PreferConsistent：首选一致性，多数时候采用该方式，在所有可用的执行器上均匀分配 kakfa 的主题的所有分区，能够综合利用集群的计算资源。

3、PreferFixed：首选固定模式，如果负载不均衡可以使用该策略放置在特定节点使用指定的主题分区；该配置是手动控制方案，若没有显式指定的分区仍然采用 (2) 方案。

### 消费策略

消费者策略是控制如何创建和配制消费者对象或者如何对 Kafka 上的消息进行消费界定，比如 t1 主题的分区 0 和 1，或者消费特定分区上的特定消息段。该类可扩展，自行实现。

1、ConsumerStrategies.Assign：指定固定的分区集合；

\`def Assign[K, V]\(

      topicPartitions: Iterable[TopicPartition],

      kafkaParams: collection.Map[String, Object],

      offsets: collection.Map[TopicPartition, Long])

\`

2、ConsumerStrategies.Subscribe：允许消费订阅固定的主题集合；

3、ConsumerStrategies.SubscribePattern：使用正则表达式指定感兴趣的主题集合。

## Spark Streaming 开发

IDEA 作为常用的开发工具使用 maven 进行依赖包的统一管理，配置 Scala 的开发环境，进行 Spark Streaming 的 API 开发；

下载并破解 IDEA，并加入汉化的包到 lib，重启生效；

在 IDEA 中导入离线的 Scala 插件：需要确保当前 win 主机上已经下载安装 Scala 并设置环境变量，首先下载 IDEA 的 Scala 插件，无须解压，然后将其添加到 IDEA 中，具体为 new---setting--plugins--"输入 scala"--install plugin from disk；

### Maven 快捷键

\`shift 键多次 ------ 查找类和插件；

shift+ctrl+enter------- 结束当前行，自动补全分号；

shift+alter+s-----------setting 设置

alter+enter----------- 补全抛出的异常

alter+insert--------- 自动生成 get、set、构造函数等；

Ctrl+X -------------- 删除当前行

ctrl+r---------------- 替换

ctrl+/---------------- 多行代码分行注释，每行一个注释符号

ctrl+shift+/--------- 多行代码注释在一个块里，只在开头和结尾有注释符号

\`

### 任务提交

新建 maven 工程：file--new--project--maven(选择 quickstart 框架模型新建)，groupId 和 ArtifactID 用来区分该 java 工程；

maven 自动生成 pom.xml 配置文件，用于各种包的依赖和引入，如果使用 maven 打包，需要引入 maven 的打包插件：使用 maven-compiler-plugin、maven-jar-plugin 插件，并在 prom.xml 中增加指定程序入口的配置；具体可参照：[https://blog.csdn.net/qq_17348297/article/details/79092383](https://blog.csdn.net/qq_17348297/article/details/79092383)

将 mainClass 设置为 HelloWorld（主类），点击右边窗口 maven -> package，生成 jar 包，打包完成后使用 spark-submit 指令提交 jar 包并执行。

`spark-submit --class "JSONRead" /usr/local/spark/mycode/json/target/scala-2.11/json-project_2.11-1.0.jar  
`

若有 cannot find main class 错误，需要删除 - class xx.jar 选项；若出现 “Invalid signature file digest for Manifest main” 错误，则运行 zip -d xxx.jar  'META-INF/.SF'  'META-INF/.RSA'  'META-INF/\*SF' 指令，删除所属 jar 包中. SF/.RSA / 相关文件。任务 yarn 管理器查看任务运行情况；

## Structured Streaming

在 Spark2.x 中，spark 新开放了一个基于 DataFrame 的无下限的流式处理组件 Structured Streaming，在过去使用 streaming 时一次处理是当前 batch 的所有数据，针对这波数据进行各种处理，如果要做一些类似 pv，uv 的统计，需要借助有状态的 state 的 DStream，或者借助一些分布式缓存系统，如 Redis，做一些类似 Group by 的操作 Streaming 是非常不便的，在面对复杂的流式处理场景时捉襟见肘，且无法支持基于 event_time 的时间窗口做聚合逻辑。

在 Structured Streaming 中，把源源不断到来的数据通过固定的模式 “追加” 或者 “更新” 到无下限的 DataFrame 中。剩余的工作跟普通的 DataFrame 一样，可以去 map、filter，也可以去 groupby().count()，甚至还可以把流处理的 dataframe 跟其他的“静态”DataFrame 进行 join。另外，还提供了基于 window 时间的流式处理。总之，Structured Streaming 提供了快速、可扩展、高可用、高可靠的流式处理。

Structured Streaming 构建于 sparksql 引擎之上，可以用处理静态数据的方式去处理你的流计算，随着流数据的不断流入，Sparksql 引擎会增量的连续不断的处理并且更新结果。可以使用 DataSet/DataFrame 的 API 进行 streaming aggregations, event-time windows, stream-to-batch joins 等，计算的执行也是基于优化后的 sparksql 引擎。通过 checkpointing and Write Ahead Logs 该系统可以保证点对点，一次处理，容错担保。

**--END--**

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

非常欢迎大家加我**个人微信**，有关大数据的问题我们在**群内**一起讨论

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zE6jg8pGqTM8bodQoPTqicfQUcAzbIQl9RmicTcXT7ecVRuEV0ZeicTU9nbpb0ggJhUc15E5Ly7OE5OA/640?wx_fmt=jpeg)

长按上方扫码二维码，加我微信，拉你进群 
 [https://mp.weixin.qq.com/s/LnmeBdjc8uYrrKiVKAzL-A](https://mp.weixin.qq.com/s/LnmeBdjc8uYrrKiVKAzL-A)
