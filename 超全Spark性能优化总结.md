# 超全Spark性能优化总结
Spark 是大数据分析的利器，在工作中用到 spark 的地方也比较多，这篇总结是希望能将自己使用 spark 的一些调优经验分享出来。

#### 一、常用参数说明

    --driver-memory 4g : driver内存大小，一般没有广播变量(broadcast)时，设置4g足够，如果有广播变量，视情况而定，可设置6G，8G，12G等均可--executor-memory 4g : 每个executor的内存，正常情况下是4g足够，但有时处理大批量数据时容易内存不足，再多申请一点，如6G--num-executors 15 : 总共申请的executor数目，普通任务十几个或者几十个足够了，若是处理海量数据如百G上T的数据时可以申请多一些，100，200等--executor-cores 2  : 每个executor内的核数，即每个executor中的任务task数目，此处设置为2，即2个task共享上面设置的6g内存，每个map或reduce任务的并行度是executor数目*executor中的任务数yarn集群中一般有资源申请上限，如，executor-memory*num-executors < 400G 等，所以调试参数时要注意这一点—-spark.default.parallelism 200 ：Spark作业的默认为500~1000个比较合适,如果不设置，spark会根据底层HDFS的block数量设置task的数量，这样会导致并行度偏少，资源利用不充分。该参数设为num-executors * executor-cores的2~3倍比较合适。-- spark.storage.memoryFraction 0.6 : 设置RDD持久化数据在Executor内存中能占的最大比例。默认值是0.6—-spark.shuffle.memoryFraction 0.2 ：设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2，如果shuffle聚合时使用的内存超出了这个20%的限制，多余数据会被溢写到磁盘文件中去，降低shuffle性能—-spark.yarn.executor.memoryOverhead 1G ：executor执行的时候，用的内存可能会超过executor-memory，所以会为executor额外预留一部分内存，spark.yarn.executor.memoryOverhead即代表这部分内存

#### 二、Spark 常用编程建议

1.  避免创建重复的 RDD，尽量复用同一份数据。
2.  尽量避免使用 shuffle 类算子，因为 shuffle 操作是 spark 中最消耗性能的地方，reduceByKey、join、distinct、repartition 等算子都会触发 shuffle 操作，尽量使用 map 类的非 shuffle 算子
3.  用 aggregateByKey 和 reduceByKey 替代 groupByKey, 因为前两个是预聚合操作，会在每个节点本地对相同的 key 做聚合，等其他节点拉取所有节点上相同的 key 时，会大大减少磁盘 IO 以及网络开销。
4.  repartition 适用于 RDD\[V], partitionBy 适用于 RDD\[K, V]
5.  mapPartitions 操作替代普通 map，foreachPartitions 替代 foreach
6.  filter 操作之后进行 coalesce 操作，可以减少 RDD 的 partition 数量
7.  如果有 RDD 复用，尤其是该 RDD 需要花费比较长的时间，建议对该 RDD 做 cache，若该 RDD 每个 partition 需要消耗很多内存，建议开启 Kryo 序列化机制 (据说可节省 2 到 5 倍空间), 若还是有比较大的内存开销，可将 storage_level 设置为 MEMORY_AND_DISK_SER
8.  尽量避免在一个 Transformation 中处理所有的逻辑，尽量分解成 map、filter 之类的操作
9.  多个 RDD 进行 union 操作时，避免使用 rdd.union(rdd).union(rdd).union(rdd) 这种多重 union，rdd.union 只适合 2 个 RDD 合并，合并多个时采用 SparkContext.union(Array(RDD))，避免 union 嵌套层数太多，导致的调用链路太长，耗时太久，且容易引发 StackOverFlow
10. spark 中的 Group/join/XXXByKey 等操作，都可以指定 partition 的个数，不需要额外使用 repartition 和 partitionBy 函数
11. 尽量保证每轮 Stage 里每个 task 处理的数据量 > 128M
12. 如果 2 个 RDD 做 join，其中一个数据量很小，可以采用 Broadcast Join，将小的 RDD 数据 collect 到 driver 内存中，将其 BroadCast 到另外以 RDD 中，其他场景想优化后面会讲
13. 2 个 RDD 做笛卡尔积时，把小的 RDD 作为参数传入，如 BigRDD.certesian(smallRDD)
14. 若需要 Broadcast 一个大的对象到远端作为字典查询，可使用多 executor-cores，大 executor-memory。若将该占用内存较大的对象存储到外部系统，executor-cores=1， executor-memory=m(默认值 2g), 可以正常运行，那么当大字典占用空间为 size(g) 时，executor-memory 为 2\*size，executor-cores=size/m(向上取整)

15\. 如果对象太大无法 BroadCast 到远端，且需求是根据大的 RDD 中的 key 去索引小 RDD 中的 key，可使用 zipPartitions 以 hash join 的方式实现，具体原理参考下一节的 shuffle 过程

16. 如果需要在 repartition 重分区之后还要进行排序，可直接使用 repartitionAndSortWithinPartitions，比分解操作效率高，因为它可以一边 shuffle 一边排序

#### 三、shuffle 性能优化

3.1 什么是 shuffle 操作 spark 中的 shuffle 操作功能：将分布在集群中多个节点上的同一个 key，拉取到同一个节点上，进行聚合或 join 操作，类似洗牌的操作。这些分布在各个存储节点上的数据重新打乱然后汇聚到不同节点的过程就是 shuffle 过程。

3.2 哪些操作中包含 shuffle 操作

RDD 的特性是不可变的带分区的记录集合，Spark 提供了 Transformation 和 Action 两种操作 RDD 的方式。Transformation 是生成新的 RDD，包括 map, flatMap, filter, union, sample, join, groupByKey, cogroup, ReduceByKey, cros, sortByKey, mapValues 等；Action 只是返回一个结果，包括 collect，reduce，count，save，lookupKey 等

Spark 所有的算子操作中是否使用 shuffle 过程要看计算后对应多少分区：

-   若一个操作执行过程中，结果 RDD 的每个分区只依赖上一个 RDD 的同一个分区，即属于窄依赖，如 map、filter、union 等操作，这种情况是不需要进行 shuffle 的，同时还可以按照 pipeline 的方式，把一个分区上的多个操作放在同一个 Task 中进行
-   若结果 RDD 的每个分区需要依赖上一个 RDD 的全部分区，即属于宽依赖，如 repartition 相关操作（repartition，coalesce）、 \* ByKey 操作（groupByKey，ReduceByKey，combineByKey、aggregateByKey 等）、join 相关操作（cogroup，join）、distinct 操作，这种依赖是需要进行 shuffle 操作的

3.3 shuffle 操作过程

shuffle 过程分为 shuffle write 和 shuffle read 两部分

-   shuffle write：分区数由上一阶段的 RDD 分区数控制，shuffle write 过程主要是将计算的中间结果按某种规则临时放到各个 executor 所在的本地磁盘上（当前 stage 结束之后，每个 task 处理的数据按 key 进行分类，数据先写入内存缓冲区，缓冲区满，溢写 spill 到磁盘文件，最终相同 key 被写入同一个磁盘文件）创建的磁盘文件数量 = 当前 stage 中 task 数量 \* 下一个 stage 的 task 数量
-   shuffle read：从上游 stage 的所有 task 节点上拉取属于自己的磁盘文件，每个 read task 会有自己的 buffer 缓冲，每次只能拉取与 buffer 缓冲相同大小的数据，然后聚合，聚合完一批后拉取下一批，边拉取边聚合。分区数由 Spark 提供的一些参数控制，如果这个参数值设置的很小，同时 shuffle read 的数据量很大，会导致一个 task 需要处理的数据非常大，容易发生 JVM crash，从而导致 shuffle 数据失败，同时 executor 也丢失了，就会看到 Failed to connect to host 的错误 (即 executor lost)。

shuffle 过程中，各个节点会通过 shuffle write 过程将相同 key 都会先写入本地磁盘文件中，然后其他节点的 shuffle read 过程通过网络传输拉取各个节点上的磁盘文件中的相同 key。这其中大量数据交换涉及到的网络传输和文件读写操作是 shuffle 操作十分耗时的根本原因

3.4 spark 的 shuffle 类型

参数 spark.shuffle.manager 用于设置 ShuffleManager 的类型。Spark1.5 以后，该参数有三个可选项：hash、sort 和 tungsten-sort。

HashShuffleManager 是 Spark1.2 以前的默认值，Spark1.2 之后的默认值都是 SortShuffleManager。tungsten-sort 与 sort 类似，但是使用了 tungsten 计划中的堆外内存管理机制，内存使用效率更高。

由于 SortShuffleManager 默认会对数据进行排序，因此如果业务需求中需要排序的话，使用默认的 SortShuffleManager 就可以；但如果不需要排序，可以通过 bypass 机制或设置 HashShuffleManager 避免排序，同时也能提供较好的磁盘读写性能。

HashShuffleManager 流程：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MH8CibLA4RwDe4gVIfTMcacYic9MGgbC6R2TVPkU8QH1vDSnSAiaWauaHx3G5WWgDjELQVIZibpRb0RQ/640?wx_fmt=png)

SortShuffleManager 流程：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MH8CibLA4RwDe4gVIfTMcacoeUpHqgXIvE8iaYSaK6PxCtgy2pxZnnkc4iaIGMqFWgmichH6mdxmJxNQ/640?wx_fmt=png)

3.5 如何开启 bypass 机制

bypass 机制通过参数 spark.shuffle.sort.bypassMergeThreshold 设置，默认值是 200，表示当 ShuffleManager 是 SortShuffleManager 时，若 shuffle read task 的数量小于这个阈值（默认 200）时，则 shuffle write 过程中不会进行排序操作，而是直接按照未经优化的 HashShuffleManager 的方式写数据，但最后会将每个 task 产生的所有临时磁盘文件合并成一个文件，并创建索引文件。

这里给出的调优建议是，当使用 SortShuffleManager 时，如果的确不需要排序，可以将这个参数值调大一些，大于 shuffle read task 的数量。那么此时就会自动开启 bypass 机制，map-side 就不会进行排序了，减少排序的性能开销，提升 shuffle 操作效率。但这种方式并没有减少 shuffle write 过程产生的磁盘文件数量，所以写的性能没有改变。

3.6 HashShuffleManager 优化建议

如果使用 HashShuffleManager，可以设置 spark.shuffle.consolidateFiles 参数。该参数默认为 false，只有当使用 HashShuffleManager 且该参数设置为 True 时，才会开启 consolidate 机制，大幅度合并 shuffle write 过程产生的输出文件，对于 shuffle read task 数量特别多的情况下，可以极大地减少磁盘 IO 开销，提升 shuffle 性能。参考社区同学给出的数据，consolidate 性能比开启 bypass 机制的 SortShuffleManager 高出 10% ~ 30%。

3.7 shuffle 调优建议

除了上述的几个参数调优，shuffle 过程还有一些参数可以提高性能：

    - spark.shuffle.file.buffer : 默认32M，shuffle Write阶段写文件时的buffer大小，若内存资源比较充足，可适当将其值调大一些（如64M），减少executor的IO读写次数，提高shuffle性能- spark.shuffle.io.maxRetries ：默认3次，Shuffle Read阶段取数据的重试次数，若shuffle处理的数据量很大，可适当将该参数调大。

3.8 shuffle 操作过程中的常见错误

SparkSQL 中的 shuffle 错误：

    org.apache.spark.shuffle.MetadataFetchFailedException: Missing an output location for shuffle 0org.apache.spark.shuffle.FetchFailedException:Failed to connect to hostname/192.168.xx.xxx:50268

RDD 中的 shuffle 错误：

    WARN TaskSetManager: Lost task 17.1 in stage 4.1 (TID 1386, spark050013): java.io.FileNotFoundException: /data04/spark/tmp/blockmgr-817d372f-c359-4a00-96dd-8f6554aa19cd/2f/temp_shuffle_e22e013a-5392-4edb-9874-a196a1dad97c (没有那个文件或目录)FetchFailed(BlockManagerId(6083b277-119a-49e8-8a49-3539690a2a3f-S155, spark050013, 8533), shuffleId=1, mapId=143, reduceId=3, message=org.apache.spark.shuffle.FetchFailedException: Error in opening FileSegmentManagedBuffer{file=/data04/spark/tmp/blockmgr-817d372f-c359-4a00-96dd-8f6554aa19cd/0e/shuffle_1_143_0.data, offset=997061, length=112503}

处理 shuffle 类操作的注意事项：

-   减少 shuffle 数据量：在 shuffle 前过滤掉不必要的数据，只选取需要的字段处理
-   针对 SparkSQL 和 DataFrame 的 join、group by 等操作：可以通过 spark.sql.shuffle.partitions 控制分区数，默认设置为 200，可根据 shuffle 的量以及计算的复杂度提高这个值，如 2000 等
-   RDD 的 join、group by、reduceByKey 等操作：通过 spark.default.parallelism 控制 shuffle read 与 reduce 处理的分区数，默认为运行任务的 core 总数，官方建议为设置成运行任务的 core 的 2~3 倍
-   提高 executor 的内存：即 spark.executor.memory 的值
-   分析数据验证是否存在数据倾斜的问题：如空值如何处理，异常数据（某个 key 对应的数据量特别大）时是否可以单独处理，可以考虑自定义数据分区规则，如何自定义可以参考下面的 join 优化环节

#### 四、join 性能优化

Spark 所有的操作中，join 操作是最复杂、代价最大的操作，也是大部分业务场景的性能瓶颈所在。所以针对 join 操作的优化是使用 spark 必须要学会的技能。

spark 的 join 操作也分为 Spark SQL 的 join 和 Spark RDD 的 join。

4.1 Spark SQL 的 join 操作

4.1.1 Hash Join

Hash Join 的执行方式是先将小表映射成 Hash Table 的方式，再将大表使用相同方式映射到 Hash Table，在同一个 hash 分区内做 join 匹配。

hash join 又分为 broadcast hash join 和 shuffle hash join 两种。其中 Broadcast hash join，顾名思义，就是把小表广播到每一个节点上的内存中，大表按 Key 保存到各个分区中，小表和每个分区的大表做 join 匹配。这种情况适合一个小表和一个大表做 join 且小表能够在内存中保存的情况。如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MH8CibLA4RwDe4gVIfTMcacDIymh5oareGMcTPqJrWxb8kNoVDNzlacVStehoAwB1jicgeS5wh6pTA/640?wx_fmt=png)

当 Hash Join 不能适用的场景就需要 Shuffle Hash Join 了，Shuffle Hash Join 的原理是按照 join Key 分区，key 相同的数据必然分配到同一分区中，将大表 join 分而治之，变成小表的 join，可以提高并行度。执行过程也分为两个阶段：

-   shuffle 阶段：分别将两个表按照 join key 进行分区，将相同的 join key 数据重分区到同一节点
-   hash join 阶段：每个分区节点上的数据单独执行单机 hash join 算法

Shuffle Hash Join 的过程如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MH8CibLA4RwDe4gVIfTMcactQB9sV0kwrS0s8g8fpGJKpwEOBRvy5DrCvBx2YwruBTZ7DedmkmIiaQ/640?wx_fmt=png)

4.1.2 Sort-Merge Join

SparkSQL 针对两张大表 join 的情况提供了全新的算法——Sort-merge join，整个过程分为三个步骤：

-   Shuffle 阶段：将两张大表根据 join key 进行重新分区，两张表数据会分布到整个集群，以便分布式进行处理
-   sort 阶段：对单个分区节点的两表数据，分别进行排序
-   merge 阶段：对排好序的两张分区表数据执行 join 操作。分别遍历两个有序序列，遇到相同的 join key 就 merge 输出，否则继续取更小一边的 key，即合并两个有序列表的方式。

sort-merge join 流程如下图所示。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MH8CibLA4RwDe4gVIfTMcac56GFSTNkKf5wMMf67bwGdV6d5ColjML1Up1UmFspOpPHoxUqiaKyTZw/640?wx_fmt=png)

4.2 Spark RDD 的 join 操作

Spark 的 RDD join 没有上面这么多的分类，但是面临的业务需求是一样的。如果是大表 join 小表的情况，则可以将小表声明为 broadcast 变量，使用 map 操作快速实现 join 功能，但又不必执行 Spark core 中的 join 操作。

如果是两个大表 join，则必须依赖 Spark Core 中的 join 操作了。Spark RDD Join 的过程可以自行阅读源码了解，这里只做一个大概的讲解。

spark 的 join 过程中最核心的函数是 cogroup 方法，这个方法中会判断 join 的两个 RDD 所使用的 partitioner 是否一样，如果分区相同，即存在 OneToOneDependency 依赖，不用进行 hash 分区，可直接 join；如果要关联的 RDD 和当前 RDD 的分区不一致时，就要对 RDD 进行重新 hash 分区，分到正确的分区中，即存在 ShuffleDependency，需要先进行 shuffle 操作再 join。因此提升 join 效率的一个思路就是使得两个 RDD 具有相同的 partitioners。

所以针对 Spark RDD 的 join 操作的优化建议是：

-   如果需要 join 的其中一个 RDD 比较小，可以直接将其存入内存，使用 broadcast hash join
-   在对两个 RDD 进行 join 操作之前，使其使用同一个 partitioners，避免 join 操作的 shuffle 过程
-   如果两个 RDD 其一存在重复的 key 也会导致 join 操作性能变低，因此最好先进行 key 值的去重处理

4.3 数据倾斜优化

均匀数据分布的情况下，前面所说的优化建议就足够了。但存在数据倾斜时，仍然会有性能问题。主要体现在绝大多数 task 执行得都非常快，个别 task 执行很慢，拖慢整个任务的执行进程，甚至可能因为某个 task 处理的数据量过大而爆出 OOM 错误。

4.3.1 分析数据分布

如果是 Spark SQL 中的 group by、join 语句导致的数据倾斜，可以使用 SQL 分析执行 SQL 中的表的 key 分布情况；如果是 Spark RDD 执行 shuffle 算子导致的数据倾斜，可以在 Spark 作业中加入分析 Key 分布的代码，使用 countByKey() 统计各个 key 对应的记录数。

4.3.2 数据倾斜的解决方案

这里参考美团技术博客中给出的几个方案。

1）针对 hive 表中的数据倾斜，可以尝试通过 hive 进行数据预处理，如按照 key 进行聚合，或是和其他表 join，Spark 作业中直接使用预处理后的数据。

2）如果发现导致倾斜的 key 就几个，而且对计算本身的影响不大，可以考虑过滤掉少数导致倾斜的 key

3）设置参数 spark.sql.shuffle.partitions，提高 shuffle 操作的并行度，增加 shuffle read task 的数量，降低每个 task 处理的数据量

4）针对 RDD 执行 reduceByKey 等聚合类算子或是在 Spark SQL 中使用 group by 语句时，可以考虑两阶段聚合方案，即局部聚合 + 全局聚合。第一阶段局部聚合，先给每个 key 打上一个随机数，接着对打上随机数的数据执行 reduceByKey 等聚合操作，然后将各个 key 的前缀去掉。第二阶段全局聚合即正常的聚合操作。

5）针对两个数据量都比较大的 RDD/hive 表进行 join 的情况，如果其中一个 RDD/hive 表的少数 key 对应的数据量过大，另一个比较均匀时，可以先分析数据，将数据量过大的几个 key 统计并拆分出来形成一个单独的 RDD，得到的两个 RDD/hive 表分别和另一个 RDD/hive 表做 join，其中 key 对应数据量较大的那个要进行 key 值随机数打散处理，另一个无数据倾斜的 RDD/hive 表要 1 对 n 膨胀扩容 n 倍，确保随机化后 key 值仍然有效。

6）针对 join 操作的 RDD 中有大量的 key 导致数据倾斜，对有数据倾斜的整个 RDD 的 key 值做随机打散处理，对另一个正常的 RDD 进行 1 对 n 膨胀扩容，每条数据都依次打上 0~n 的前缀。处理完后再执行 join 操作

#### 五、其他错误总结

(1) 报错信息

    java.lang.OutOfMemory, unable to create new native threadCaused by: java.lang.OutOfMemoryError: unable to create new native thread        at java.lang.Thread.start0(Native Method)        at java.lang.Thread.start(Thread.java:640)

解决方案：

上面这段错误提示的本质是 Linux 操作系统无法创建更多进程，导致出错，并不是系统的内存不足。因此要解决这个问题需要修改 Linux 允许创建更多的进程，就需要修改 Linux 最大进程数

（2）报错信息

由于 Spark 在计算的时候会将中间结果存储到 / tmp 目录，而目前 linux 又都支持 tmpfs，其实就是将 / tmp 目录挂载到内存当中, 那么这里就存在一个问题，中间结果过多导致 / tmp 目录写满而出现如下错误 No Space Left on the device（Shuffle 临时文件过多）

解决方案：

修改配置文件 spark-env.sh, 把临时文件引入到一个自定义的目录中去, 即:

    export SPARK_LOCAL_DIRS=/home/utoken/datadir/spark/tmp

（3）报错信息 Worker 节点中的 work 目录占用许多磁盘空间, 这些是 Driver 上传到 worker 的文件, 会占用许多磁盘空间 需要定时做手工清理 work 目录

（4）spark-shell 提交 Spark Application 如何解决依赖库 解决方案：

（5）内存不足或数据倾斜导致 Executor Lost，shuffle fetch 失败，Task 重试失败等（spark-submit 提交）

    TaskSetManager: Lost task 1.0 in stage 6.0 (TID 100, 192.168.10.37): java.lang.OutOfMemoryError: Java heap spaceINFO BlockManagerInfo: Added broadcast_8_piece0 in memory on 192.168.10.37:57139 (size: 42.0 KB, free: 24.2 MB)INFO BlockManagerInfo: Added broadcast_8_piece0 in memory on 192.168.10.38:53816 (size: 42.0 KB, free: 24.2 MB)INFO TaskSetManager: Starting task 3.0 in stage 6.0 (TID 102, 192.168.10.37, ANY, 2152 bytes)

解决方案：增加 worker 内存，或者相同资源下增加 partition 数目，这样每个 task 要处理的数据变少，占用内存变少

如果存在 shuffle 过程，设置 shuffle read 阶段的并行数。 
 [https://mp.weixin.qq.com/s/M3YCKtbZt97jXrPFXqafAA](https://mp.weixin.qq.com/s/M3YCKtbZt97jXrPFXqafAA)
