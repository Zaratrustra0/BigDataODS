# Spark性能优化指南——高级篇
[Spark 性能优化指南——基础篇](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247494904&idx=1&sn=ec31482d43ef125277c8b6516c1803c5&chksm=f9ed15d0ce9a9cc61c8390e22bdd80d198ef8072b21476c2dd589cba154dab0f41a08e0e31ea&scene=21#wechat_redirect)  

继基础篇讲解了每个 Spark 开发人员都必须熟知的开发调优与资源调优之后，本文作为《Spark 性能优化指南》的高级篇，将深入分析数据倾斜调优与 shuffle 调优，以解决更加棘手的性能问题。

## 调优概述

有的时候，我们可能会遇到大数据计算中一个最棘手的问题——数据倾斜，此时 Spark 作业的性能会比期望差很多。数据倾斜调优，就是使用各种技术方案解决不同类型的数据倾斜问题，以保证 Spark 作业的性能。

## 数据倾斜发生时的现象

-   绝大多数 task 执行得都非常快，但个别 task 执行极慢。比如，总共有 1000 个 task，997 个 task 都在 1 分钟之内执行完了，但是剩余两三个 task 却要一两个小时。这种情况很常见。
-   原本能够正常执行的 Spark 作业，某天突然报出 OOM（内存溢出）异常，观察异常栈，是我们写的业务代码造成的。这种情况比较少见。

## 数据倾斜发生的原理

数据倾斜的原理很简单：在进行 shuffle 的时候，必须将各个节点上相同的 key 拉取到某个节点上的一个 task 来进行处理，比如按照 key 进行聚合或 join 等操作。此时如果某个 key 对应的数据量特别大的话，就会发生数据倾斜。比如大部分 key 对应 10 条数据，但是个别 key 却对应了 100 万条数据，那么大部分 task 可能就只会分配到 10 条数据，然后 1 秒钟就运行完了；但是个别 task 可能分配到了 100 万数据，要运行一两个小时。因此，整个 Spark 作业的运行进度是由运行时间最长的那个 task 决定的。

因此出现数据倾斜的时候，Spark 作业看起来会运行得非常缓慢，甚至可能因为某个 task 处理的数据量过大导致内存溢出。

下图就是一个很清晰的例子：hello 这个 key，在三个节点上对应了总共 7 条数据，这些数据都会被拉取到同一个 task 中进行处理；而 world 和 you 这两个 key 分别才对应 1 条数据，所以另外两个 task 只要分别处理 1 条数据即可。此时第一个 task 的运行时间可能是另外两个 task 的 7 倍，而整个 stage 的运行速度也由运行最慢的那个 task 所决定。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM994q4o0Fqm0NazBsy7eYlJuEXTNUtca7gABW5mib1KR8IVErxTrC4Krw/640?wx_fmt=png)

## 如何定位导致数据倾斜的代码

数据倾斜只会发生在 shuffle 过程中。这里给大家罗列一些常用的并且可能会触发 shuffle 操作的算子：distinct、groupByKey、reduceByKey、aggregateByKey、join、cogroup、repartition 等。出现数据倾斜时，可能就是你的代码中使用了这些算子中的某一个所导致的。

### 某个 task 执行特别慢的情况

首先要看的，就是数据倾斜发生在第几个 stage 中。

如果是用 yarn-client 模式提交，那么本地是直接可以看到 log 的，可以在 log 中找到当前运行到了第几个 stage；如果是用 yarn-cluster 模式提交，则可以通过 Spark Web UI 来查看当前运行到了第几个 stage。此外，无论是使用 yarn-client 模式还是 yarn-cluster 模式，我们都可以在 Spark Web UI 上深入看一下当前这个 stage 各个 task 分配的数据量，从而进一步确定是不是 task 分配的数据不均匀导致了数据倾斜。

比如下图中，倒数第三列显示了每个 task 的运行时间。明显可以看到，有的 task 运行特别快，只需要几秒钟就可以运行完；而有的 task 运行特别慢，需要几分钟才能运行完，此时单从运行时间上看就已经能够确定发生数据倾斜了。此外，倒数第一列显示了每个 task 处理的数据量，明显可以看到，运行时间特别短的 task 只需要处理几百 KB 的数据即可，而运行时间特别长的 task 需要处理几千 KB 的数据，处理的数据量差了 10 倍。此时更加能够确定是发生了数据倾斜。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9wicgs88wuica8xQXJaRVocViaibVdqbNpCLWefXfacsPuyiaeOuRO7h6P6Q/640?wx_fmt=png)

知道数据倾斜发生在哪一个 stage 之后，接着我们就需要根据 stage 划分原理，推算出来发生倾斜的那个 stage 对应代码中的哪一部分，这部分代码中肯定会有一个 shuffle 类算子。精准推算 stage 与代码的对应关系，需要对 Spark 的源码有深入的理解，这里我们可以介绍一个相对简单实用的推算方法：只要看到 Spark 代码中出现了一个 shuffle 类算子或者是 Spark SQL 的 SQL 语句中出现了会导致 shuffle 的语句（比如 group by 语句），那么就可以判定，以那个地方为界限划分出了前后两个 stage。  

这里我们就以 Spark 最基础的入门程序——单词计数来举例，如何用最简单的方法大致推算出一个 stage 对应的代码。如下示例，在整个代码中，只有一个 reduceByKey 是会发生 shuffle 的算子，因此就可以认为，以这个算子为界限，会划分出前后两个 stage。_ stage0，主要是执行从 textFile 到 map 操作，以及执行 shuffle write 操作。shuffle write 操作，我们可以简单理解为对 pairs RDD 中的数据进行分区操作，每个 task 处理的数据中，相同的 key 会写入同一个磁盘文件内。_ stage1，主要是执行从 reduceByKey 到 collect 操作，stage1 的各个 task 一开始运行，就会首先执行 shuffle read 操作。执行 shuffle read 操作的 task，会从 stage0 的各个 task 所在节点拉取属于自己处理的那些 key，然后对同一个 key 进行全局性的聚合或 join 等操作，在这里就是对 key 的 value 值进行累加。stage1 在执行完 reduceByKey 算子之后，就计算出了最终的 wordCounts RDD，然后会执行 collect 算子，将所有数据拉取到 Driver 上，供我们遍历和打印输出。

\`val conf = new SparkConf()  
val sc = new SparkContext(conf)

 val lines = sc.textFile("hdfs://...")  
val words = lines.flatMap(_.split(" "))  
val pairs = words.map((_, 1))  
val wordCounts = pairs.reduceByKey(_ + _)

 wordCounts.collect().foreach(println(\_))

\`

通过对单词计数程序的分析，希望能够让大家了解最基本的 stage 划分的原理，以及 stage 划分后 shuffle 操作是如何在两个 stage 的边界处执行的。然后我们就知道如何快速定位出发生数据倾斜的 stage 对应代码的哪一个部分了。比如我们在 Spark Web UI 或者本地 log 中发现，stage1 的某几个 task 执行得特别慢，判定 stage1 出现了数据倾斜，那么就可以回到代码中定位出 stage1 主要包括了 reduceByKey 这个 shuffle 类算子，此时基本就可以确定是由 educeByKey 算子导致的数据倾斜问题。比如某个单词出现了 100 万次，其他单词才出现 10 次，那么 stage1 的某个 task 就要处理 100 万数据，整个 stage 的速度就会被这个 task 拖慢。

### 某个 task 莫名其妙内存溢出的情况

这种情况下去定位出问题的代码就比较容易了。我们建议直接看 yarn-client 模式下本地 log 的异常栈，或者是通过 YARN 查看 yarn-cluster 模式下的 log 中的异常栈。一般来说，通过异常栈信息就可以定位到你的代码中哪一行发生了内存溢出。然后在那行代码附近找找，一般也会有 shuffle 类算子，此时很可能就是这个算子导致了数据倾斜。

但是大家要注意的是，不能单纯靠偶然的内存溢出就判定发生了数据倾斜。因为自己编写的代码的 bug，以及偶然出现的数据异常，也可能会导致内存溢出。因此还是要按照上面所讲的方法，通过 Spark Web UI 查看报错的那个 stage 的各个 task 的运行时间以及分配的数据量，才能确定是否是由于数据倾斜才导致了这次内存溢出。

## 查看导致数据倾斜的 key 的数据分布情况

知道了数据倾斜发生在哪里之后，通常需要分析一下那个执行了 shuffle 操作并且导致了数据倾斜的 RDD/Hive 表，查看一下其中 key 的分布情况。这主要是为之后选择哪一种技术方案提供依据。针对不同的 key 分布与不同的 shuffle 算子组合起来的各种情况，可能需要选择不同的技术方案来解决。

此时根据你执行操作的情况不同，可以有很多种查看 key 分布的方式：

1.  如果是 Spark SQL 中的 group by、join 语句导致的数据倾斜，那么就查询一下 SQL 中使用的表的 key 分布情况。
2.  如果是对 Spark RDD 执行 shuffle 算子导致的数据倾斜，那么可以在 Spark 作业中加入查看 key 分布的代码，比如 RDD.countByKey()。然后对统计出来的各个 key 出现的次数，collect/take 到客户端打印一下，就可以看到 key 的分布情况。

举例来说，对于上面所说的单词计数程序，如果确定了是 stage1 的 reduceByKey 算子导致了数据倾斜，那么就应该看看进行 reduceByKey 操作的 RDD 中的 key 分布情况，在这个例子中指的就是 pairs RDD。如下示例，我们可以先对 pairs 采样 10% 的样本数据，然后使用 countByKey 算子统计出每个 key 出现的次数，最后在客户端遍历和打印样本数据中各个 key 的出现次数。

`val sampledPairs = pairs.sample(false, 0.1)  
val sampledWordCounts = sampledPairs.countByKey()  
sampledWordCounts.foreach(println(_))`

## 数据倾斜的解决方案

### 解决方案一：使用 Hive ETL 预处理数据

**方案适用场景：** 导致数据倾斜的是 Hive 表。如果该 Hive 表中的数据本身很不均匀（比如某个 key 对应了 100 万数据，其他 key 才对应了 10 条数据），而且业务场景需要频繁使用 Spark 对 Hive 表执行某个分析操作，那么比较适合使用这种技术方案。

**方案实现思路：** 此时可以评估一下，是否可以通过 Hive 来进行数据预处理（即通过 Hive ETL 预先对数据按照 key 进行聚合，或者是预先和其他表进行 join），然后在 Spark 作业中针对的数据源就不是原来的 Hive 表了，而是预处理后的 Hive 表。此时由于数据已经预先进行过聚合或 join 操作了，那么在 Spark 作业中也就不需要使用原先的 shuffle 类算子执行这类操作了。

**方案实现原理：** 这种方案从根源上解决了数据倾斜，因为彻底避免了在 Spark 中执行 shuffle 类算子，那么肯定就不会有数据倾斜的问题了。但是这里也要提醒一下大家，这种方式属于治标不治本。因为毕竟数据本身就存在分布不均匀的问题，所以 Hive ETL 中进行 group by 或者 join 等 shuffle 操作时，还是会出现数据倾斜，导致 Hive ETL 的速度很慢。我们只是把数据倾斜的发生提前到了 Hive ETL 中，避免 Spark 程序发生数据倾斜而已。

**方案优点：** 实现起来简单便捷，效果还非常好，完全规避掉了数据倾斜，Spark 作业的性能会大幅度提升。

**方案缺点：** 治标不治本，Hive ETL 中还是会发生数据倾斜。

**方案实践经验：** 在一些 Java 系统与 Spark 结合使用的项目中，会出现 Java 代码频繁调用 Spark 作业的场景，而且对 Spark 作业的执行性能要求很高，就比较适合使用这种方案。将数据倾斜提前到上游的 Hive ETL，每天仅执行一次，只有那一次是比较慢的，而之后每次 Java 调用 Spark 作业时，执行速度都会很快，能够提供更好的用户体验。

**项目实践经验：** 在美团 · 点评的交互式用户行为分析系统中使用了这种方案，该系统主要是允许用户通过 Java Web 系统提交数据分析统计任务，后端通过 Java 提交 Spark 作业进行数据分析统计。要求 Spark 作业速度必须要快，尽量在 10 分钟以内，否则速度太慢，用户体验会很差。所以我们将有些 Spark 作业的 shuffle 操作提前到了 Hive ETL 中，从而让 Spark 直接使用预处理的 Hive 中间表，尽可能地减少 Spark 的 shuffle 操作，大幅度提升了性能，将部分作业的性能提升了 6 倍以上。

### 解决方案二：过滤少数导致倾斜的 key

**方案适用场景：** 如果发现导致倾斜的 key 就少数几个，而且对计算本身的影响并不大的话，那么很适合使用这种方案。比如 99% 的 key 就对应 10 条数据，但是只有一个 key 对应了 100 万数据，从而导致了数据倾斜。

**方案实现思路：** 如果我们判断那少数几个数据量特别多的 key，对作业的执行和计算结果不是特别重要的话，那么干脆就直接过滤掉那少数几个 key。比如，在 Spark SQL 中可以使用 where 子句过滤掉这些 key 或者在 Spark Core 中对 RDD 执行 filter 算子过滤掉这些 key。如果需要每次作业执行时，动态判定哪些 key 的数据量最多然后再进行过滤，那么可以使用 sample 算子对 RDD 进行采样，然后计算出每个 key 的数量，取数据量最多的 key 过滤掉即可。

**方案实现原理：** 将导致数据倾斜的 key 给过滤掉之后，这些 key 就不会参与计算了，自然不可能产生数据倾斜。

**方案优点：** 实现简单，而且效果也很好，可以完全规避掉数据倾斜。

**方案缺点：** 适用场景不多，大多数情况下，导致倾斜的 key 还是很多的，并不是只有少数几个。

**方案实践经验：** 在项目中我们也采用过这种方案解决数据倾斜。有一次发现某一天 Spark 作业在运行的时候突然 OOM 了，追查之后发现，是 Hive 表中的某一个 key 在那天数据异常，导致数据量暴增。因此就采取每次执行前先进行采样，计算出样本中数据量最大的几个 key 之后，直接在程序中将那些 key 给过滤掉。

### 解决方案三：提高 shuffle 操作的并行度

**方案适用场景：** 如果我们必须要对数据倾斜迎难而上，那么建议优先使用这种方案，因为这是处理数据倾斜最简单的一种方案。

**方案实现思路：** 在对 RDD 执行 shuffle 算子时，给 shuffle 算子传入一个参数，比如 reduceByKey(1000)，该参数就设置了这个 shuffle 算子执行时 shuffle read task 的数量。对于 Spark SQL 中的 shuffle 类语句，比如 group by、join 等，需要设置一个参数，即 spark.sql.shuffle.partitions，该参数代表了 shuffle read task 的并行度，该值默认是 200，对于很多场景来说都有点过小。

**方案实现原理：** 增加 shuffle read task 的数量，可以让原本分配给一个 task 的多个 key 分配给多个 task，从而让每个 task 处理比原来更少的数据。举例来说，如果原本有 5 个 key，每个 key 对应 10 条数据，这 5 个 key 都是分配给一个 task 的，那么这个 task 就要处理 50 条数据。而增加了 shuffle read task 以后，每个 task 就分配到一个 key，即每个 task 就处理 10 条数据，那么自然每个 task 的执行时间都会变短了。具体原理如下图所示。

**方案优点：** 实现起来比较简单，可以有效缓解和减轻数据倾斜的影响。

**方案缺点：** 只是缓解了数据倾斜而已，没有彻底根除问题，根据实践经验来看，其效果有限。

**方案实践经验：** 该方案通常无法彻底解决数据倾斜，因为如果出现一些极端情况，比如某个 key 对应的数据量有 100 万，那么无论你的 task 数量增加到多少，这个对应着 100 万数据的 key 肯定还是会分配到一个 task 中去处理，因此注定还是会发生数据倾斜的。所以这种方案只能说是在发现数据倾斜时尝试使用的第一种手段，尝试去用嘴简单的方法缓解数据倾斜而已，或者是和其他方案结合起来使用。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9t6gTickK97U28x7wuH848HeCkxuEibUTIgIWicZJpaNC0dkmhfs0UyROw/640?wx_fmt=png)

### 解决方案四：两阶段聚合（局部聚合 + 全局聚合）

**方案适用场景：** 对 RDD 执行 reduceByKey 等聚合类 shuffle 算子或者在 Spark SQL 中使用 group by 语句进行分组聚合时，比较适用这种方案。

**方案实现思路：** 这个方案的核心实现思路就是进行两阶段聚合。第一次是局部聚合，先给每个 key 都打上一个随机数，比如 10 以内的随机数，此时原先一样的 key 就变成不一样的了，比如 (hello, 1) (hello, 1) (hello, 1) (hello, 1)，就会变成 (1_hello, 1) (1_hello, 1) (2_hello, 1) (2_hello, 1)。接着对打上随机数后的数据，执行 reduceByKey 等聚合操作，进行局部聚合，那么局部聚合结果，就会变成了 (1_hello, 2) (2_hello, 2)。然后将各个 key 的前缀给去掉，就会变成 (hello,2)(hello,2)，再次进行全局聚合操作，就可以得到最终结果了，比如 (hello, 4)。

**方案实现原理：** 将原本相同的 key 通过附加随机前缀的方式，变成多个不同的 key，就可以让原本被一个 task 处理的数据分散到多个 task 上去做局部聚合，进而解决单个 task 处理数据量过多的问题。接着去除掉随机前缀，再次进行全局聚合，就可以得到最终的结果。具体原理见下图。

**方案优点：** 对于聚合类的 shuffle 操作导致的数据倾斜，效果是非常不错的。通常都可以解决掉数据倾斜，或者至少是大幅度缓解数据倾斜，将 Spark 作业的性能提升数倍以上。

**方案缺点：** 仅仅适用于聚合类的 shuffle 操作，适用范围相对较窄。如果是 join 类的 shuffle 操作，还得用其他的解决方案。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9yicWkec3tnWHmCZBZzROJaaDmaZ24p2WfuCEldk06z4MUllhib1ibIPdQ/640?wx_fmt=png)

\`// 第一步，给 RDD 中的每个 key 都打上一个随机前缀。  
JavaPairRDD&lt;String, Long> randomPrefixRdd = rdd.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,Long>, String, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, Long> call(Tuple2&lt;Long, Long> tuple)  
                    throws Exception {  
                Random random = new Random();  
                int prefix = random.nextInt(10);  
                return new Tuple2&lt;String, Long>(prefix + "\_" + tuple.\_1, tuple.\_2);  
            }  
        });

  // 第二步，对打上随机前缀的 key 进行局部聚合。  
JavaPairRDD&lt;String, Long> localAggrRdd = randomPrefixRdd.reduceByKey(  
        new Function2&lt;Long, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Long call(Long v1, Long v2) throws Exception {  
                return v1 + v2;  
            }  
        });

  // 第三步，去除 RDD 中每个 key 的随机前缀。  
JavaPairRDD&lt;Long, Long> removedRandomPrefixRdd = localAggrRdd.mapToPair(  
        new PairFunction&lt;Tuple2&lt;String,Long>, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;Long, Long> call(Tuple2&lt;String, Long> tuple)  
                    throws Exception {  
                long originalKey = Long.valueOf(tuple._1.split("_")[1]);  
                return new Tuple2&lt;Long, Long>(originalKey, tuple.\_2);  
            }  
        });

  // 第四步，对去除了随机前缀的 RDD 进行全局聚合。  
JavaPairRDD&lt;Long, Long> globalAggrRdd = removedRandomPrefixRdd.reduceByKey(  
        new Function2&lt;Long, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Long call(Long v1, Long v2) throws Exception {  
                return v1 + v2;  
            }  
        });

\`

### 解决方案五：将 reduce join 转为 map join

**方案适用场景：** 在对 RDD 使用 join 类操作，或者是在 Spark SQL 中使用 join 语句时，而且 join 操作中的一个 RDD 或表的数据量比较小（比如几百 M 或者一两 G），比较适用此方案。

**方案实现思路：** 不使用 join 算子进行连接操作，而使用 Broadcast 变量与 map 类算子实现 join 操作，进而完全规避掉 shuffle 类的操作，彻底避免数据倾斜的发生和出现。将较小 RDD 中的数据直接通过 collect 算子拉取到 Driver 端的内存中来，然后对其创建一个 Broadcast 变量；接着对另外一个 RDD 执行 map 类算子，在算子函数内，从 Broadcast 变量中获取较小 RDD 的全量数据，与当前 RDD 的每一条数据按照连接 key 进行比对，如果连接 key 相同的话，那么就将两个 RDD 的数据用你需要的方式连接起来。

**方案实现原理：** 普通的 join 是会走 shuffle 过程的，而一旦 shuffle，就相当于会将相同 key 的数据拉取到一个 shuffle read task 中再进行 join，此时就是 reduce join。但是如果一个 RDD 是比较小的，则可以采用广播小 RDD 全量数据 + map 算子来实现与 join 同样的效果，也就是 map join，此时就不会发生 shuffle 操作，也就不会发生数据倾斜。具体原理如下图所示。

**方案优点：** 对 join 操作导致的数据倾斜，效果非常好，因为根本就不会发生 shuffle，也就根本不会发生数据倾斜。

**方案缺点：** 适用场景较少，因为这个方案只适用于一个大表和一个小表的情况。毕竟我们需要将小表进行广播，此时会比较消耗内存资源，driver 和每个 Executor 内存中都会驻留一份小 RDD 的全量数据。如果我们广播出去的 RDD 数据比较大，比如 10G 以上，那么就可能发生内存溢出了。因此并不适合两个都是大表的情况。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9xXGpzSkVcjVHhzI32QJEibe9IkrX1WRNyPftWZUR3N86Jb6iaD5CUneQ/640?wx_fmt=png)

\`// 首先将数据量比较小的 RDD 的数据，collect 到 Driver 中来。  
List&lt;Tuple2&lt;Long, Row>> rdd1Data = rdd1.collect()  
// 然后使用 Spark 的广播功能，将小 RDD 的数据转换成广播变量，这样每个 Executor 就只有一份 RDD 的数据。  
// 可以尽可能节省内存空间，并且减少网络传输性能开销。  
final Broadcast&lt;List&lt;Tuple2&lt;Long, Row>>> rdd1DataBroadcast = sc.broadcast(rdd1Data);

  // 对另外一个 RDD 执行 map 类操作，而不再是 join 类操作。  
JavaPairRDD&lt;String, Tuple2&lt;String, Row>> joinedRdd = rdd2.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, String, Tuple2&lt;String, Row>>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, Tuple2&lt;String, Row>> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                // 在算子函数中，通过广播变量，获取到本地 Executor 中的 rdd1 数据。  
                List&lt;Tuple2&lt;Long, Row>> rdd1Data = rdd1DataBroadcast.value();  
                // 可以将 rdd1 的数据转换为一个 Map，便于后面进行 join 操作。  
                Map&lt;Long, Row> rdd1DataMap = new HashMap&lt;Long, Row>();  
                for(Tuple2&lt;Long, Row> data : rdd1Data) {  
                    rdd1DataMap.put(data.\_1, data.\_2);  
                }  
                // 获取当前 RDD 数据的 key 以及 value。  
                String key = tuple.\_1;  
                String value = tuple.\_2;  
                // 从 rdd1 数据 Map 中，根据 key 获取到可以 join 到的数据。  
                Row rdd1Value = rdd1DataMap.get(key);  
                return new Tuple2&lt;String, String>(key, new Tuple2&lt;String, Row>(value, rdd1Value));  
            }  
        });

  // 这里得提示一下。  
// 上面的做法，仅仅适用于 rdd1 中的 key 没有重复，全部是唯一的场景。  
// 如果 rdd1 中有多个相同的 key，那么就得用 flatMap 类的操作，在进行 join 的时候不能用 map，而是得遍历 rdd1 所有数据进行 join。  
// rdd2 中每条数据都可能会返回多条 join 后的数据。

\`

### 解决方案六：采样倾斜 key 并分拆 join 操作

**方案适用场景：** 两个 RDD/Hive 表进行 join 的时候，如果数据量都比较大，无法采用 “解决方案五”，那么此时可以看一下两个 RDD/Hive 表中的 key 分布情况。如果出现数据倾斜，是因为其中某一个 RDD/Hive 表中的少数几个 key 的数据量过大，而另一个 RDD/Hive 表中的所有 key 都分布比较均匀，那么采用这个解决方案是比较合适的。

**方案实现思路：** 

-   对包含少数几个数据量过大的 key 的那个 RDD，通过 sample 算子采样出一份样本来，然后统计一下每个 key 的数量，计算出来数据量最大的是哪几个 key。
-   然后将这几个 key 对应的数据从原来的 RDD 中拆分出来，形成一个单独的 RDD，并给每个 key 都打上 n 以内的随机数作为前缀，而不会导致倾斜的大部分 key 形成另外一个 RDD。
-   接着将需要 join 的另一个 RDD，也过滤出来那几个倾斜 key 对应的数据并形成一个单独的 RDD，将每条数据膨胀成 n 条数据，这 n 条数据都按顺序附加一个 0~n 的前缀，不会导致倾斜的大部分 key 也形成另外一个 RDD。
-   再将附加了随机前缀的独立 RDD 与另一个膨胀 n 倍的独立 RDD 进行 join，此时就可以将原先相同的 key 打散成 n 份，分散到多个 task 中去进行 join 了。
-   而另外两个普通的 RDD 就照常 join 即可。
-   最后将两次 join 的结果使用 union 算子合并起来即可，就是最终的 join 结果。

**方案实现原理：** 对于 join 导致的数据倾斜，如果只是某几个 key 导致了倾斜，可以将少数几个 key 分拆成独立 RDD，并附加随机前缀打散成 n 份去进行 join，此时这几个 key 对应的数据就不会集中在少数几个 task 上，而是分散到多个 task 进行 join 了。具体原理见下图。

**方案优点：** 对于 join 导致的数据倾斜，如果只是某几个 key 导致了倾斜，采用该方式可以用最有效的方式打散 key 进行 join。而且只需要针对少数倾斜 key 对应的数据进行扩容 n 倍，不需要对全量数据进行扩容。避免了占用过多内存。

**方案缺点：** 如果导致倾斜的 key 特别多的话，比如成千上万个 key 都导致数据倾斜，那么这种方式也不适合。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9Y8icRMwLQGu99n5l1hP0VTQqKpV6MY7sia1ib4IE53yDCUGWfcZI640Pw/640?wx_fmt=png)

\`// 首先将数据量比较小的 RDD 的数据，collect 到 Driver 中来。  
List&lt;Tuple2&lt;Long, Row>> rdd1Data = rdd1.collect()  
// 然后使用 Spark 的广播功能，将小 RDD 的数据转换成广播变量，这样每个 Executor 就只有一份 RDD 的数据。  
// 可以尽可能节省内存空间，并且减少网络传输性能开销。  
final Broadcast&lt;List&lt;Tuple2&lt;Long, Row>>> rdd1DataBroadcast = sc.broadcast(rdd1Data);

  // 对另外一个 RDD 执行 map 类操作，而不再是 join 类操作。  
JavaPairRDD&lt;String, Tuple2&lt;String, Row>> joinedRdd = rdd2.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, String, Tuple2&lt;String, Row>>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, Tuple2&lt;String, Row>> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                // 在算子函数中，通过广播变量，获取到本地 Executor 中的 rdd1 数据。  
                List&lt;Tuple2&lt;Long, Row>> rdd1Data = rdd1DataBroadcast.value();  
                // 可以将 rdd1 的数据转换为一个 Map，便于后面进行 join 操作。  
                Map&lt;Long, Row> rdd1DataMap = new HashMap&lt;Long, Row>();  
                for(Tuple2&lt;Long, Row> data : rdd1Data) {  
                    rdd1DataMap.put(data.\_1, data.\_2);  
                }  
                // 获取当前 RDD 数据的 key 以及 value。  
                String key = tuple.\_1;  
                String value = tuple.\_2;  
                // 从 rdd1 数据 Map 中，根据 key 获取到可以 join 到的数据。  
                Row rdd1Value = rdd1DataMap.get(key);  
                return new Tuple2&lt;String, String>(key, new Tuple2&lt;String, Row>(value, rdd1Value));  
            }  
        });

  // 这里得提示一下。  
// 上面的做法，仅仅适用于 rdd1 中的 key 没有重复，全部是唯一的场景。  
// 如果 rdd1 中有多个相同的 key，那么就得用 flatMap 类的操作，在进行 join 的时候不能用 map，而是得遍历 rdd1 所有数据进行 join。  
// rdd2 中每条数据都可能会返回多条 join 后的数据。// 首先从包含了少数几个导致数据倾斜 key 的 rdd1 中，采样 10% 的样本数据。  
JavaPairRDD&lt;Long, String> sampledRDD = rdd1.sample(false, 0.1);

  // 对样本数据 RDD 统计出每个 key 的出现次数，并按出现次数降序排序。  
// 对降序排序后的数据，取出 top 1 或者 top 100 的数据，也就是 key 最多的前 n 个数据。  
// 具体取出多少个数据量最多的 key，由大家自己决定，我们这里就取 1 个作为示范。  
JavaPairRDD&lt;Long, Long> mappedSampledRDD = sampledRDD.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;Long, Long> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                return new Tuple2&lt;Long, Long>(tuple.\_1, 1L);  
            }       
        });  
JavaPairRDD&lt;Long, Long> countedSampledRDD = mappedSampledRDD.reduceByKey(  
        new Function2&lt;Long, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Long call(Long v1, Long v2) throws Exception {  
                return v1 + v2;  
            }  
        });  
JavaPairRDD&lt;Long, Long> reversedSampledRDD = countedSampledRDD.mapToPair(   
        new PairFunction&lt;Tuple2&lt;Long,Long>, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;Long, Long> call(Tuple2&lt;Long, Long> tuple)  
                    throws Exception {  
                return new Tuple2&lt;Long, Long>(tuple.\_2, tuple.\_1);  
            }  
        });  
final Long skewedUserid = reversedSampledRDD.sortByKey(false).take(1).get(0).\_2;

  // 从 rdd1 中分拆出导致数据倾斜的 key，形成独立的 RDD。  
JavaPairRDD&lt;Long, String> skewedRDD = rdd1.filter(  
        new Function&lt;Tuple2&lt;Long,String>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, String> tuple) throws Exception {  
                return tuple.\_1.equals(skewedUserid);  
            }  
        });  
// 从 rdd1 中分拆出不导致数据倾斜的普通 key，形成独立的 RDD。  
JavaPairRDD&lt;Long, String> commonRDD = rdd1.filter(  
        new Function&lt;Tuple2&lt;Long,String>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, String> tuple) throws Exception {  
                return !tuple.\_1.equals(skewedUserid);  
            }   
        });

  // rdd2，就是那个所有 key 的分布相对较为均匀的 rdd。  
// 这里将 rdd2 中，前面获取到的 key 对应的数据，过滤出来，分拆成单独的 rdd，并对 rdd 中的数据使用 flatMap 算子都扩容 100 倍。  
// 对扩容的每条数据，都打上 0～100 的前缀。  
JavaPairRDD&lt;String, Row> skewedRdd2 = rdd2.filter(  
         new Function&lt;Tuple2&lt;Long,Row>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, Row> tuple) throws Exception {  
                return tuple._1.equals(skewedUserid);  
            }  
        }).flatMapToPair(new PairFlatMapFunction&lt;Tuple2&lt;Long,Row>, String, Row>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Iterable&lt;Tuple2&lt;String, Row>> call(  
                    Tuple2&lt;Long, Row> tuple) throws Exception {  
                Random random = new Random();  
                List&lt;Tuple2&lt;String, Row>> list = new ArrayList&lt;Tuple2&lt;String, Row>>();  
                for(int i = 0; i &lt; 100; i++) {  
                    list.add(new Tuple2&lt;String, Row>(i + "_" + tuple.\_1, tuple.\_2));  
                }  
                return list;  
            }

                      });

 // 将 rdd1 中分拆出来的导致倾斜的 key 的独立 rdd，每条数据都打上 100 以内的随机前缀。  
// 然后将这个 rdd1 中分拆出来的独立 rdd，与上面 rdd2 中分拆出来的独立 rdd，进行 join。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD1 = skewedRDD.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, String, String>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, String> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                Random random = new Random();  
                int prefix = random.nextInt(100);  
                return new Tuple2&lt;String, String>(prefix + "_" + tuple.\_1, tuple.\_2);  
            }  
        })  
        .join(skewedUserid2infoRDD)  
        .mapToPair(new PairFunction&lt;Tuple2&lt;String,Tuple2&lt;String,Row>>, Long, Tuple2&lt;String, Row>>() {  
                        private static final long serialVersionUID = 1L;  
                        @Override  
                        public Tuple2&lt;Long, Tuple2&lt;String, Row>> call(  
                            Tuple2&lt;String, Tuple2&lt;String, Row>> tuple)  
                            throws Exception {  
                            long key = Long.valueOf(tuple.\_1.split("_")[1]);  
                            return new Tuple2&lt;Long, Tuple2&lt;String, Row>>(key, tuple.\_2);  
                        }  
                    });

 // 将 rdd1 中分拆出来的包含普通 key 的独立 rdd，直接与 rdd2 进行 join。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD2 = commonRDD.join(rdd2);

 // 将倾斜 key join 后的结果与普通 key join 后的结果，uinon 起来。  
// 就是最终的 join 结果。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD = joinedRDD1.union(joinedRDD2);// 首先从包含了少数几个导致数据倾斜 key 的 rdd1 中，采样 10% 的样本数据。  
JavaPairRDD&lt;Long, String> sampledRDD = rdd1.sample(false, 0.1);

  // 对样本数据 RDD 统计出每个 key 的出现次数，并按出现次数降序排序。  
// 对降序排序后的数据，取出 top 1 或者 top 100 的数据，也就是 key 最多的前 n 个数据。  
// 具体取出多少个数据量最多的 key，由大家自己决定，我们这里就取 1 个作为示范。  
JavaPairRDD&lt;Long, Long> mappedSampledRDD = sampledRDD.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;Long, Long> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                return new Tuple2&lt;Long, Long>(tuple.\_1, 1L);  
            }       
        });  
JavaPairRDD&lt;Long, Long> countedSampledRDD = mappedSampledRDD.reduceByKey(  
        new Function2&lt;Long, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Long call(Long v1, Long v2) throws Exception {  
                return v1 + v2;  
            }  
        });  
JavaPairRDD&lt;Long, Long> reversedSampledRDD = countedSampledRDD.mapToPair(   
        new PairFunction&lt;Tuple2&lt;Long,Long>, Long, Long>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;Long, Long> call(Tuple2&lt;Long, Long> tuple)  
                    throws Exception {  
                return new Tuple2&lt;Long, Long>(tuple.\_2, tuple.\_1);  
            }  
        });  
final Long skewedUserid = reversedSampledRDD.sortByKey(false).take(1).get(0).\_2;

  // 从 rdd1 中分拆出导致数据倾斜的 key，形成独立的 RDD。  
JavaPairRDD&lt;Long, String> skewedRDD = rdd1.filter(  
        new Function&lt;Tuple2&lt;Long,String>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, String> tuple) throws Exception {  
                return tuple.\_1.equals(skewedUserid);  
            }  
        });  
// 从 rdd1 中分拆出不导致数据倾斜的普通 key，形成独立的 RDD。  
JavaPairRDD&lt;Long, String> commonRDD = rdd1.filter(  
        new Function&lt;Tuple2&lt;Long,String>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, String> tuple) throws Exception {  
                return !tuple.\_1.equals(skewedUserid);  
            }   
        });

  // rdd2，就是那个所有 key 的分布相对较为均匀的 rdd。  
// 这里将 rdd2 中，前面获取到的 key 对应的数据，过滤出来，分拆成单独的 rdd，并对 rdd 中的数据使用 flatMap 算子都扩容 100 倍。  
// 对扩容的每条数据，都打上 0～100 的前缀。  
JavaPairRDD&lt;String, Row> skewedRdd2 = rdd2.filter(  
         new Function&lt;Tuple2&lt;Long,Row>, Boolean>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Boolean call(Tuple2&lt;Long, Row> tuple) throws Exception {  
                return tuple._1.equals(skewedUserid);  
            }  
        }).flatMapToPair(new PairFlatMapFunction&lt;Tuple2&lt;Long,Row>, String, Row>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Iterable&lt;Tuple2&lt;String, Row>> call(  
                    Tuple2&lt;Long, Row> tuple) throws Exception {  
                Random random = new Random();  
                List&lt;Tuple2&lt;String, Row>> list = new ArrayList&lt;Tuple2&lt;String, Row>>();  
                for(int i = 0; i &lt; 100; i++) {  
                    list.add(new Tuple2&lt;String, Row>(i + "_" + tuple.\_1, tuple.\_2));  
                }  
                return list;  
            }

                      });

 // 将 rdd1 中分拆出来的导致倾斜的 key 的独立 rdd，每条数据都打上 100 以内的随机前缀。  
// 然后将这个 rdd1 中分拆出来的独立 rdd，与上面 rdd2 中分拆出来的独立 rdd，进行 join。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD1 = skewedRDD.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, String, String>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, String> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                Random random = new Random();  
                int prefix = random.nextInt(100);  
                return new Tuple2&lt;String, String>(prefix + "_" + tuple.\_1, tuple.\_2);  
            }  
        })  
        .join(skewedUserid2infoRDD)  
        .mapToPair(new PairFunction&lt;Tuple2&lt;String,Tuple2&lt;String,Row>>, Long, Tuple2&lt;String, Row>>() {  
                        private static final long serialVersionUID = 1L;  
                        @Override  
                        public Tuple2&lt;Long, Tuple2&lt;String, Row>> call(  
                            Tuple2&lt;String, Tuple2&lt;String, Row>> tuple)  
                            throws Exception {  
                            long key = Long.valueOf(tuple.\_1.split("_")[1]);  
                            return new Tuple2&lt;Long, Tuple2&lt;String, Row>>(key, tuple.\_2);  
                        }  
                    });

 // 将 rdd1 中分拆出来的包含普通 key 的独立 rdd，直接与 rdd2 进行 join。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD2 = commonRDD.join(rdd2);

 // 将倾斜 key join 后的结果与普通 key join 后的结果，uinon 起来。  
// 就是最终的 join 结果。  
JavaPairRDD&lt;Long, Tuple2&lt;String, Row>> joinedRDD = joinedRDD1.union(joinedRDD2);

\`

### 解决方案七：使用随机前缀和扩容 RDD 进行 join

**方案适用场景：** 如果在进行 join 操作时，RDD 中有大量的 key 导致数据倾斜，那么进行分拆 key 也没什么意义，此时就只能使用最后一种方案来解决问题了。

**方案实现思路：** 

-   该方案的实现思路基本和 “解决方案六” 类似，首先查看 RDD/Hive 表中的数据分布情况，找到那个造成数据倾斜的 RDD/Hive 表，比如有多个 key 都对应了超过 1 万条数据。
-   然后将该 RDD 的每条数据都打上一个 n 以内的随机前缀。
-   同时对另外一个正常的 RDD 进行扩容，将每条数据都扩容成 n 条数据，扩容出来的每条数据都依次打上一个 0~n 的前缀。
-   最后将两个处理后的 RDD 进行 join 即可。

**方案实现原理：** 将原先一样的 key 通过附加随机前缀变成不一样的 key，然后就可以将这些处理后的 “不同 key” 分散到多个 task 中去处理，而不是让一个 task 处理大量的相同 key。该方案与 “解决方案六” 的不同之处就在于，上一种方案是尽量只对少数倾斜 key 对应的数据进行特殊处理，由于处理过程需要扩容 RDD，因此上一种方案扩容 RDD 后对内存的占用并不大；而这一种方案是针对有大量倾斜 key 的情况，没法将部分 key 拆分出来进行单独处理，因此只能对整个 RDD 进行数据扩容，对内存资源要求很高。

**方案优点：** 对 join 类型的数据倾斜基本都可以处理，而且效果也相对比较显著，性能提升效果非常不错。

**方案缺点：** 该方案更多的是缓解数据倾斜，而不是彻底避免数据倾斜。而且需要对整个 RDD 进行扩容，对内存资源要求很高。

**方案实践经验：** 曾经开发一个数据需求的时候，发现一个 join 导致了数据倾斜。优化之前，作业的执行时间大约是 60 分钟左右；使用该方案优化之后，执行时间缩短到 10 分钟左右，性能提升了 6 倍。

\`// 首先将其中一个 key 分布相对较为均匀的 RDD 膨胀 100 倍。  
JavaPairRDD&lt;String, Row> expandedRDD = rdd1.flatMapToPair(  
        new PairFlatMapFunction&lt;Tuple2&lt;Long,Row>, String, Row>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Iterable&lt;Tuple2&lt;String, Row>> call(Tuple2&lt;Long, Row> tuple)  
                    throws Exception {  
                List&lt;Tuple2&lt;String, Row>> list = new ArrayList&lt;Tuple2&lt;String, Row>>();  
                for(int i = 0; i &lt; 100; i++) {  
                    list.add(new Tuple2&lt;String, Row>(0 + "\_" + tuple.\_1, tuple.\_2));  
                }  
                return list;  
            }  
        });

  // 其次，将另一个有数据倾斜 key 的 RDD，每条数据都打上 100 以内的随机前缀。  
JavaPairRDD&lt;String, String> mappedRDD = rdd2.mapToPair(  
        new PairFunction&lt;Tuple2&lt;Long,String>, String, String>() {  
            private static final long serialVersionUID = 1L;  
            @Override  
            public Tuple2&lt;String, String> call(Tuple2&lt;Long, String> tuple)  
                    throws Exception {  
                Random random = new Random();  
                int prefix = random.nextInt(100);  
                return new Tuple2&lt;String, String>(prefix + "\_" + tuple.\_1, tuple.\_2);  
            }  
        });

  // 将两个处理后的 RDD 进行 join 即可。  
JavaPairRDD&lt;String, Tuple2&lt;String, Row>> joinedRDD = mappedRDD.join(expandedRDD);

\`

### 解决方案八：多种方案组合使用

在实践中发现，很多情况下，如果只是处理较为简单的数据倾斜场景，那么使用上述方案中的某一种基本就可以解决。但是如果要处理一个较为复杂的数据倾斜场景，那么可能需要将多种方案组合起来使用。比如说，我们针对出现了多个数据倾斜环节的 Spark 作业，可以先运用解决方案一和二，预处理一部分数据，并过滤一部分数据来缓解；其次可以对某些 shuffle 操作提升并行度，优化其性能；最后还可以针对不同的聚合或 join 操作，选择一种方案来优化其性能。大家需要对这些方案的思路和原理都透彻理解之后，在实践中根据各种不同的情况，灵活运用多种方案，来解决自己的数据倾斜问题。

## 调优概述

大多数 Spark 作业的性能主要就是消耗在了 shuffle 环节，因为该环节包含了大量的磁盘 IO、序列化、网络数据传输等操作。因此，如果要让作业的性能更上一层楼，就有必要对 shuffle 过程进行调优。但是也必须提醒大家的是，影响一个 Spark 作业性能的因素，主要还是代码开发、资源参数以及数据倾斜，shuffle 调优只能在整个 Spark 的性能调优中占到一小部分而已。因此大家务必把握住调优的基本原则，千万不要舍本逐末。下面我们就给大家详细讲解 shuffle 的原理，以及相关参数的说明，同时给出各个参数的调优建议。

## ShuffleManager 发展概述

在 Spark 的源码中，负责 shuffle 过程的执行、计算和处理的组件主要就是 ShuffleManager，也即 shuffle 管理器。而随着 Spark 的版本的发展，ShuffleManager 也在不断迭代，变得越来越先进。

在 Spark 1.2 以前，默认的 shuffle 计算引擎是 HashShuffleManager。该 ShuffleManager 而 HashShuffleManager 有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘 IO 操作影响了性能。

因此在 Spark 1.2 以后的版本中，默认的 ShuffleManager 改成了 SortShuffleManager。SortShuffleManager 相较于 HashShuffleManager 来说，有了一定的改进。主要就在于，每个 Task 在进行 shuffle 操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并（merge）成一个磁盘文件，因此每个 Task 就只有一个磁盘文件。在下一个 stage 的 shuffle read task 拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

下面我们详细分析一下 HashShuffleManager 和 SortShuffleManager 的原理。

## HashShuffleManager 运行原理

### 未经优化的 HashShuffleManager

下图说明了未经优化的 HashShuffleManager 的原理。这里我们先明确一个假设前提：每个 Executor 只有 1 个 CPU core，也就是说，无论这个 Executor 上分配多少个 task 线程，同一时间都只能执行一个 task 线程。

我们先从 shuffle write 开始说起。shuffle write 阶段，主要就是在一个 stage 结束计算之后，为了下一个 stage 可以执行 shuffle 类的算子（比如 reduceByKey），而将每个 task 处理的数据按 key 进行 “分类”。所谓 “分类”，就是对相同的 key 执行 hash 算法，从而将相同 key 都写入同一个磁盘文件中，而每一个磁盘文件都只属于下游 stage 的一个 task。在将数据写入磁盘之前，会先将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。

那么每个执行 shuffle write 的 task，要为下一个 stage 创建多少个磁盘文件呢？很简单，下一个 stage 的 task 有多少个，当前 stage 的每个 task 就要创建多少份磁盘文件。比如下一个 stage 总共有 100 个 task，那么当前 stage 的每个 task 都要创建 100 份磁盘文件。如果当前 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 Task，那么每个 Executor 上总共就要创建 500 个磁盘文件，所有 Executor 上会创建 5000 个磁盘文件。由此可见，未经优化的 shuffle write 操作所产生的磁盘文件的数量是极其惊人的。

接着我们来说说 shuffle read。shuffle read，通常就是一个 stage 刚开始时要做的事情。此时该 stage 的每一个 task 就需要将上一个 stage 的计算结果中的所有相同 key，从各个节点上通过网络都拉取到自己所在的节点上，然后进行 key 的聚合或连接等操作。由于 shuffle write 的过程中，task 给下游 stage 的每个 task 都创建了一个磁盘文件，因此 shuffle read 的过程中，每个 task 只要从上游 stage 的所有 task 所在节点上，拉取属于自己的那一个磁盘文件即可。

shuffle read 的拉取过程是一边拉取一边进行聚合的。每个 shuffle read task 都会有一个自己的 buffer 缓冲，每次都只能拉取与 buffer 缓冲相同大小的数据，然后通过内存中的一个 Map 进行聚合等操作。聚合完一批数据后，再拉取下一批数据，并放到 buffer 缓冲中进行聚合操作。以此类推，直到最后将所有数据到拉取完，并得到最终的结果。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9APS7G4qVfGBic5CHnib2GxUEpw5xr20g7WuucpsVAicuRNRXrPZNZU3eg/640?wx_fmt=png)

### 优化后的 HashShuffleManager

下图说明了优化后的 HashShuffleManager 的原理。这里说的优化，是指我们可以设置一个参数，spark.shuffle.consolidateFiles。该参数默认值为 false，将其设置为 true 即可开启优化机制。通常来说，如果我们使用 HashShuffleManager，那么都建议开启这个选项。

开启 consolidate 机制之后，在 shuffle write 过程中，task 就不是为下游 stage 的每个 task 创建一个磁盘文件了。此时会出现 shuffleFileGroup 的概念，每个 shuffleFileGroup 会对应一批磁盘文件，磁盘文件的数量与下游 stage 的 task 数量是相同的。一个 Executor 上有多少个 CPU core，就可以并行执行多少个 task。而第一批并行执行的每个 task 都会创建一个 shuffleFileGroup，并将数据写入对应的磁盘文件内。

当 Executor 的 CPU core 执行完一批 task，接着执行下一批 task 时，下一批 task 就会复用之前已有的 shuffleFileGroup，包括其中的磁盘文件。也就是说，此时 task 会将数据写入已有的磁盘文件中，而不会写入新的磁盘文件中。因此，consolidate 机制允许不同的 task 复用同一批磁盘文件，这样就可以有效将多个 task 的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升 shuffle write 的性能。

假设第二个 stage 有 100 个 task，第一个 stage 有 50 个 task，总共还是有 10 个 Executor，每个 Executor 执行 5 个 task。那么原本使用未经优化的 HashShuffleManager 时，每个 Executor 会产生 500 个磁盘文件，所有 Executor 会产生 5000 个磁盘文件的。但是此时经过优化之后，每个 Executor 创建的磁盘文件的数量的计算公式为：CPU core 的数量 \* 下一个 stage 的 task 数量。也就是说，每个 Executor 此时只会创建 100 个磁盘文件，所有 Executor 只会创建 1000 个磁盘文件。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9rWBtP4o0JGjibTP9kUPckzjkx3IRtiarNGmUnaNvExMKGB5333lTUxOw/640?wx_fmt=png)

## SortShuffleManager 运行原理

SortShuffleManager 的运行机制主要分成两种，一种是普通运行机制，另一种是 bypass 运行机制。当 shuffle read task 的数量小于等于 spark.shuffle.sort.bypassMergeThreshold 参数的值时（默认为 200），就会启用 bypass 机制。

### 普通运行机制

下图说明了普通的 SortShuffleManager 的原理。在该模式下，数据会先写入一个内存数据结构中，此时根据不同的 shuffle 算子，可能选用不同的数据结构。如果是 reduceByKey 这种聚合类的 shuffle 算子，那么会选用 Map 数据结构，一边通过 Map 进行聚合，一边写入内存；如果是 join 这种普通的 shuffle 算子，那么会选用 Array 数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。

在溢写到磁盘文件之前，会先根据 key 对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的 batch 数量是 10000 条，也就是说，排序好的数据，会以每批 1 万条数据的形式分批写入磁盘文件。写入磁盘文件是通过 Java 的 BufferedOutputStream 实现的。BufferedOutputStream 是 Java 的缓冲输出流，首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘 IO 次数，提升性能。

一个 task 将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，这就是 merge 过程，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。此外，由于一个 task 就只对应一个磁盘文件，也就意味着该 task 为下游 stage 的 task 准备的数据都在这一个文件中，因此还会单独写一份索引文件，其中标识了下游各个 task 的数据在文件中的 start offset 与 end offset。

SortShuffleManager 由于有一个磁盘文件 merge 的过程，因此大大减少了文件数量。比如第一个 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 task，而第二个 stage 有 100 个 task。由于每个 task 最终只有一个磁盘文件，因此此时每个 Executor 上只有 5 个磁盘文件，所有 Executor 只有 50 个磁盘文件。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9UhCy3GwAU7YK5OCEqGetvGBL8wFZD8TzjJRQs74nPO0CPyfRHh1VlA/640?wx_fmt=png)

### bypass 运行机制

下图说明了 bypass SortShuffleManager 的原理。bypass 运行机制的触发条件如下：

-   shuffle map task 数量小于 spark.shuffle.sort.bypassMergeThreshold 参数的值。
-   不是聚合类的 shuffle 算子（比如 reduceByKey）。

此时 task 会为每个下游 task 都创建一个临时磁盘文件，并将数据按 key 进行 hash 然后根据 key 的 hash 值，将 key 写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

该过程的磁盘写机制其实跟未经优化的 HashShuffleManager 是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的 HashShuffleManager 来说，shuffle read 的性能会更好。

而该机制与普通 SortShuffleManager 运行机制的不同在于：第一，磁盘写机制不同；第二，不会进行排序。也就是说，启用该机制的最大好处在于，shuffle write 过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXdnRTumCBhZjWr6ojF35mM9EZm2q24lEEzKbAUCMKZd23Vh1ibTGpIRMGp13zwoJpOfMkcVk9mc4YQ/640?wx_fmt=png)

shuffle 相关参数调优

以下是 Shffule 过程中的一些主要参数，这里详细讲解了各个参数的功能、默认值以及基于实践经验给出的调优建议。

### spark.shuffle.file.buffer

默认值：32k

参数说明：该参数用于设置 shuffle write task 的 BufferedOutputStream 的 buffer 缓冲大小。将数据写到磁盘文件之前，会先写入 buffer 缓冲中，待缓冲写满之后，才会溢写到磁盘。

调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如 64k），从而减少 shuffle write 过程中溢写磁盘文件的次数，也就可以减少磁盘 IO 次数，进而提升性能。在实践中发现，合理调节该参数，性能会有 1%~5% 的提升。

### spark.reducer.maxSizeInFlight

默认值：48m

参数说明：该参数用于设置 shuffle read task 的 buffer 缓冲大小，而这个 buffer 缓冲决定了每次能够拉取多少数据。

调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如 96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有 1%~5% 的提升。

### spark.shuffle.io.maxRetries

默认值：3

参数说明：shuffle read task 从 shuffle write task 所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。

调优建议：对于那些包含了特别耗时的 shuffle 操作的作业，建议增加重试最大次数（比如 60 次），以避免由于 JVM 的 full gc 或者网络不稳定等因素导致的数据拉取失败。在实践中发现，对于针对超大数据量（数十亿~ 上百亿）的 shuffle 过程，调节该参数可以大幅度提升稳定性。

### spark.shuffle.io.retryWait

默认值：5s

参数说明：具体解释同上，该参数代表了每次重试拉取数据的等待间隔，默认是 5s。

调优建议：建议加大间隔时长（比如 60s），以增加 shuffle 操作的稳定性。

### spark.shuffle.memoryFraction

默认值：0.2

参数说明：该参数代表了 Executor 内存中，分配给 shuffle read task 进行聚合操作的内存比例，默认是 20%。

调优建议：在资源参数调优中讲解过这个参数。如果内存充足，而且很少使用持久化操作，建议调高这个比例，给 shuffle read 的聚合操作更多内存，以避免由于内存不足导致聚合过程中频繁读写磁盘。在实践中发现，合理调节该参数可以将性能提升 10% 左右。

### spark.shuffle.manager

默认值：sort

参数说明：该参数用于设置 ShuffleManager 的类型。Spark 1.5 以后，有三个可选项：hash、sort 和 tungsten-sort。HashShuffleManager 是 Spark 1.2 以前的默认选项，但是 Spark 1.2 以及之后的版本默认都是 SortShuffleManager 了。tungsten-sort 与 sort 类似，但是使用了 tungsten 计划中的堆外内存管理机制，内存使用效率更高。

调优建议：由于 SortShuffleManager 默认会对数据进行排序，因此如果你的业务逻辑中需要该排序机制的话，则使用默认的 SortShuffleManager 就可以；而如果你的业务逻辑不需要对数据进行排序，那么建议参考后面的几个参数调优，通过 bypass 机制或优化的 HashShuffleManager 来避免排序操作，同时提供较好的磁盘读写性能。这里要注意的是，tungsten-sort 要慎用，因为之前发现了一些相应的 bug。

### spark.shuffle.sort.bypassMergeThreshold

默认值：200

参数说明：当 ShuffleManager 为 SortShuffleManager 时，如果 shuffle read task 的数量小于这个阈值（默认是 200），则 shuffle write 过程中不会进行排序操作，而是直接按照未经优化的 HashShuffleManager 的方式去写数据，但是最后会将每个 task 产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。

调优建议：当你使用 SortShuffleManager 时，如果的确不需要排序操作，那么建议将这个参数调大一些，大于 shuffle read task 的数量。那么此时就会自动启用 bypass 机制，map-side 就不会进行排序了，减少了排序的性能开销。但是这种方式下，依然会产生大量的磁盘文件，因此 shuffle write 性能有待提高。

### spark.shuffle.consolidateFiles

默认值：false

参数说明：如果使用 HashShuffleManager，该参数有效。如果设置为 true，那么就会开启 consolidate 机制，会大幅度合并 shuffle write 的输出文件，对于 shuffle read task 数量特别多的情况下，这种方法可以极大地减少磁盘 IO 开销，提升性能。

调优建议：如果的确不需要 SortShuffleManager 的排序机制，那么除了使用 bypass 机制，还可以尝试将 spark.shffle.manager 参数手动指定为 hash，使用 HashShuffleManager，同时开启 consolidate 机制。在实践中尝试过，发现其性能比开启了 bypass 机制的 SortShuffleManager 要高出 10%~30%。

本文分别讲解了开发过程中的优化原则、运行前的资源参数设置调优、运行中的数据倾斜的解决方案、为了精益求精的 shuffle 调优。希望大家能够在阅读本文之后，记住这些性能调优的原则以及方案，在 Spark 作业开发、测试以及运行的过程中多尝试，只有这样，我们才能开发出更优的 Spark 作业，不断提升其性能。

本文链接：[https://tech.meituan.com/2016/05/12/spark-tuning-pro.html](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

[Spark 性能优化指南——基础篇](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247494904&idx=1&sn=ec31482d43ef125277c8b6516c1803c5&chksm=f9ed15d0ce9a9cc61c8390e22bdd80d198ef8072b21476c2dd589cba154dab0f41a08e0e31ea&scene=21#wechat_redirect)  

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXeFzuCNYd7EAPsScZvZ6KhcWvvLhicX9SVRibv8pY7MtyrEYicyrfXfae9E6OQbUtIUr4bicpKBJSFNmA/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/KIoE3ev7XgGGH05kjCX_zQ](https://mp.weixin.qq.com/s/KIoE3ev7XgGGH05kjCX_zQ)
