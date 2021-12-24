# Spark调优宝典(4w字)
**1**

**性能调优**  

**1**

**分配更多资源**

### **分配哪些资源？**

Executor 的数量

每个 Executor 所能分配的 CPU 数量

每个 Executor 所能分配的内存量

Driver 端分配的内存数量

### **在哪里分配这些资源？**

在生产环境中，提交 spark 作业时，用的 spark-submit shell 脚本，里面调整对应的参数：

```sql
/usr/local/spark/bin/spark-submit \





/usr/local/SparkTest-0.0.1-SNAPSHOT-jar-with-dependencies.jar \
```

### **调节到多大，算是最大呢？**

常用的资源调度模式有 Spark Standalone 和 Spark On Yarn。比如说你的每台机器能够给你使用 60G 内存，10 个 cpu core，20 台机器。那么 executor 的数量是 20。平均每个 executor 所能分配 60G 内存和 10 个 cpu core。

### **为什么多分配了这些资源以后，性能会得到提升？**

Ø 增加 executor：

如果 executor 数量比较少，那么，能够并行执行的 task 数量就比较少，就意味着，我们的 Application 的并行执行的能力就很弱。

比如有 3 个 executor，每个 executor 有 2 个 cpu core，那么同时能够并行执行的 task，就是 6 个。6 个执行完以后，再换下一批 6 个 task。

增加了 executor 数量以后，那么，就意味着，能够并行执行的 task 数量，也就变多了。比如原先是 6 个，现在可能可以并行执行 10 个，甚至 20 个，100 个。那么并行能力就比之前提升了数倍，数十倍。相应的，性能（执行的速度），也能提升数倍~ 数十倍。

Ø 增加每个 executor 的 cpu core，也是增加了执行的并行能力。原本 20 个 executor，每个才 2 个 cpu core。能够并行执行的 task 数量，就是 40 个 task。

现在每个 executor 的 cpu core，增加到了 4 个。能够并行执行的 task 数量，就是 80 个 task。就物理性能来看，执行的速度，提升了 2 倍。

Ø 增加每个 executor 的内存量。增加了内存量以后，对性能的提升，有三点：

1、如果需要对 RDD 进行 cache，那么更多的内存，就可以缓存更多的数据，将更少的数据写入磁盘，甚至不写入磁盘。减少了磁盘 IO。

2、对于 shuffle 操作，reduce 端，会需要内存来存放拉取的数据并进行聚合。如果内存不够，也会写入磁盘。如果给 executor 分配更多内存以后，就有更少的数据，需要写入磁盘，甚至不需要写入磁盘。减少了磁盘 IO，提升了性能。

3、对于 task 的执行，可能会创建很多对象。如果内存比较小，可能会频繁导致 JVM 堆内存满了，然后频繁 GC，垃圾回收，minor GC 和 full GC。（速度很慢）。内存加大以后，带来更少的 GC，垃圾回收，避免了速度变慢，速度变快了。

**2**

**调节并行度**

### **并行度的概念**

就是指的是 Spark 作业中，各个 stage 的 task 数量，代表了 Spark 作业的在各个阶段（stage）的并行度。

### **如果不调节并行度，导致并行度过低，会怎么样？**

比如现在 spark-submit 脚本里面，给我们的 spark 作业分配了足够多的资源，比如 50 个 executor，每个 executor 有 10G 内存，每个 executor 有 3 个 cpu core。基本已经达到了集群或者 yarn 队列的资源上限。task 没有设置，或者设置的很少，比如就设置了 100 个 task，你的 Application 任何一个 stage 运行的时候，都有总数在 150 个 cpu core，可以并行运行。但是你现在，只有 100 个 task，平均分配一下，每个 executor 分配到 2 个 task，ok，那么同时在运行的 task，只有 100 个，每个 executor 只会并行运行 2 个 task。每个 executor 剩下的一个 cpu core， 就浪费掉了。

你的资源虽然分配足够了，但是问题是，并行度没有与资源相匹配，导致你分配下去的资源都浪费掉了。

合理的并行度的设置，应该是要设置的足够大，大到可以完全合理的利用你的集群资源。比如上面的例子，总共集群有 150 个 cpu core，可以并行运行 150 个 task。那么就应该将你的 Application 的并行度，至少设置成 150，才能完全有效的利用你的集群资源，让 150 个 task，并行执行。而且 task 增加到 150 个以后，即可以同时并行运行，还可以让每个 task 要处理的数据量变少。比如总共 150G 的数据要处理，如果是 100 个 task，每个 task 计算 1.5G 的数据，现在增加到 150 个 task，可以并行运行，而且每个 task 主要处理 1G 的数据就可以。

很简单的道理，只要合理设置并行度，就可以完全充分利用你的集群计算资源，并且减少每个 task 要处理的数据量，最终，就是提升你的整个 Spark 作业的性能和运行速度。

### **设置并行度**

1）、task 数量，至少设置成与 Spark application 的总 cpu core 数量相同（最理想情况，比如总共 150 个 cpu core，分配了 150 个 task，一起运行，差不多同一时间运行完毕）。

2）、官方是推荐，task 数量，设置成 spark application 总 cpu core 数量的 2~3 倍，比如 150 个 cpu core，基本要设置 task 数量为 300~500。

实际情况，与理想情况不同的，有些 task 会运行的快一点，比如 50s 就完了，有些 task，可能会慢一点，要 1 分半才运行完，所以如果你的 task 数量，刚好设置的跟 cpu core 数量相同，可能还是会导致资源的浪费。比如 150 个 task，10 个先运行完了，剩余 140 个还在运行，但是这个时候，有 10 个 cpu core 就空闲出来了，就导致了浪费。那如果 task 数量设置成 cpu core 总数的 2~3 倍，那么一个 task 运行完了以后，另一个 task 马上可以补上来，就尽量让 cpu core 不要空闲，同时也是尽量提升 spark 作业运行的效率和速度，提升性能。

3）、如何设置一个 Spark Application 的并行度？

```cs
spark.default.parallelism
SparkConf conf = new SparkConf()
  .set("spark.default.parallelism", "500")
```

**3**

**重构 RDD 架构以及 RDD 持久化**

### **RDD\*\***架构重构与优化 \*\*

尽量去复用 RDD，差不多的 RDD，可以抽取成为一个共同的 RDD，供后面的 RDD 计算时，反复使用。

### **公共 \*\***RDD\***\* 一定要实现持久化**

对于要多次计算和使用的公共 RDD，一定要进行持久化。

持久化，就是将 RDD 的数据缓存到内存中 / 磁盘中（BlockManager）以后无论对这个 RDD 做多少次计算，那么都是直接取这个 RDD 的持久化的数据，比如从内存中或者磁盘中，直接提取一份数据。

### **持久化，是可以进行序列化的**

如果正常将数据持久化在内存中，那么可能会导致内存的占用过大，这样的话，也许，会导致 OOM 内存溢出。

当纯内存无法支撑公共 RDD 数据完全存放的时候，就优先考虑使用序列化的方式在纯内存中存储。将 RDD 的每个 partition 的数据，序列化成一个大的字节数组，就一个对象。序列化后，大大减少内存的空间占用。

序列化的方式，唯一的缺点就是，在获取数据的时候，需要反序列化。

如果序列化纯内存方式，还是导致 OOM 内存溢出，就只能考虑磁盘的方式、内存 + 磁盘的普通方式（无序列化）、内存 + 磁盘（序列化）。

### **为了数据的高可靠性，而且内存充足，可以使用双副本机制，进行持久化。**

持久化的双副本机制，持久化后的一个副本，因为机器宕机了，副本丢了，就还是得重新计算一次。持久化的每个数据单元，存储一份副本，放在其他节点上面。从而进行容错。一个副本丢了，不用重新计算，还可以使用另外一份副本。这种方式，仅仅针对你的内存资源极度充足的情况。

**4**

**广播变量**

### **概念及需求**

Spark Application（我们自己写的 Spark 作业）最开始在 Driver 端，在我们提交任务的时候，需要传递到各个 Executor 的 Task 上运行。对于一些只读、固定的数据 (比如从 DB 中读出的数据), 每次都需要 Driver 广播到各个 Task 上，这样效率低下。广播变量允许将变量只广播（提前广播）给各个 Executor。该 Executor 上的各个 Task 再从所在节点的 BlockManager 获取变量，如果本地没有，那么就从 Driver 远程拉取变量副本，并保存在本地的 BlockManager 中。此后这个 executor 上的 task，都会直接使用本地的 BlockManager 中的副本。而不是从 Driver 获取变量，从而提升了效率。

一个 Executor 只需要在第一个 Task 启动时，获得一份 Broadcast 数据，之后的 Task 都从本节点的 BlockManager 中获取相关数据。

### **使用方法**

1）调用 SparkContext.broadcast 方法创建一个 Broadcast\[T]对象。任何序列化的类型都可以这么实现。

2）通过 value 方法访问该对象的值。

3）变量只会被发送到各个节点一次，应作为只读值处理（修改这个值不会影响到别的节点）

**5**

**使用 Kryo 序列化**

### **概念及需求**

默认情况下，Spark 内部是使用 Java 的序列化机制，ObjectOutputStream / ObjectInputStream，对象输入输出流机制，来进行序列化。

这种默认序列化机制的好处在于，处理起来比较方便，也不需要我们手动去做什么事情，只是，你在算子里面使用的变量，必须是实现 Serializable 接口的，可序列化即可。

但是缺点在于，默认的序列化机制的效率不高，序列化的速度比较慢，序列化以后的数据，占用的内存空间相对还是比较大。

Spark 支持使用 Kryo 序列化机制。这种序列化机制，比默认的 Java 序列化机制速度要快，序列化后的数据更小，大概是 Java 序列化机制的 1/10。

所以 Kryo 序列化优化以后，可以让网络传输的数据变少，在集群中耗费的内存资源大大减少。

### **Kryo\*\***序列化机制启用以后生效的几个地方 \*\*

1）、算子函数中使用到的外部变量，使用 Kryo 以后：优化网络传输的性能，可以优化集群中内存的占用和消耗。

2）、持久化 RDD，优化内存的占用和消耗。持久化 RDD 占用的内存越少，task 执行的时候，创建的对象，就不至于频繁的占满内存，频繁发生 GC。

3）、shuffle：可以优化网络传输的性能。

### **使用方法**

第一步，在 SparkConf 中设置一个属性，spark.serializer，org.apache.spark.serializer.KryoSerializer 类。

第二步，注册你使用的需要通过 Kryo 序列化的一些自定义类，SparkConf.registerKryoClasses()。

项目中的使用：

```php
.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
.registerKryoClasses(new Class[]{CategorySortKey.class})
```

**6**

**使用 fastutil 优化数据格式**

### **fastutil\*\***介绍 \*\*

fastutil 是扩展了 Java 标准集合框架（Map、List、Set。HashMap、ArrayList、HashSet）的类库，提供了特殊类型的 map、set、list 和 queue。

fastutil 能够提供更小的内存占用，更快的存取速度。我们使用 fastutil 提供的集合类，来替代自己平时使用的 JDK 的原生的 Map、List、Set，好处在于 fastutil 集合类可以减小内存的占用，并且在进行集合的遍历、根据索引（或者 key）获取元素的值和设置元素的值的时候，提供更快的存取速度。

fastutil 也提供了 64 位的 array、set 和 list，以及高性能快速的，以及实用的 IO 类，来处理二进制和文本类型的文件。

fastutil 最新版本要求 Java 7 以及以上版本。

fastutil 的每一种集合类型，都实现了对应的 Java 中的标准接口（比如 fastutil 的 map，实现了 Java 的 Map 接口），因此可以直接放入已有系统的任何代码中。

fastutil 还提供了一些 JDK 标准类库中没有的额外功能（比如双向迭代器）。

fastutil 除了对象和原始类型为元素的集合，fastutil 也提供引用类型的支持，但是对引用类型是使用等于号（=）进行比较的，而不是 equals() 方法。

fastutil 尽量提供了在任何场景下都是速度最快的集合类库。

### **Spark\*\***中应用 \***\*fastutil\*\***的场景 \*\*

1）、如果算子函数使用了外部变量。第一，你可以使用 Broadcast 广播变量优化。第二，可以使用 Kryo 序列化类库，提升序列化性能和效率。第三，如果外部变量是某种比较大的集合，那么可以考虑使用 fastutil 改写外部变量，首先从源头上就减少内存的占用，通过广播变量进一步减少内存占用，再通过 Kryo 序列化类库进一步减少内存占用。

2）、在你的算子函数里，也就是 task 要执行的计算逻辑里面，如果有逻辑中，出现，要创建比较大的 Map、List 等集合，可能会占用较大的内存空间，而且可能涉及到消耗性能的遍历、存取等集合操作，此时，可以考虑将这些集合类型使用 fastutil 类库重写，使用了 fastutil 集合类以后，就可以在一定程度上，减少 task 创建出来的集合类型的内存占用。避免 executor 内存频繁占满，频繁唤起 GC，导致性能下降。

### **关于 \*\***fastutil\***\* 调优的说明**

fastutil 其实没有你想象中的那么强大，也不会跟官网上说的效果那么一鸣惊人。广播变量、Kryo 序列化类库、fastutil，都是之前所说的，对于性能来说，类似于一种调味品，烤鸡，本来就很好吃了，然后加了一点特质的孜然麻辣粉调料，就更加好吃了一点。分配资源、并行度、RDD 架构与持久化，这三个就是烤鸡。broadcast、kryo、fastutil，类似于调料。

比如说，你的 spark 作业，经过之前一些调优以后，大概 30 分钟运行完，现在加上 broadcast、kryo、fastutil，也许就是优化到 29 分钟运行完、或者更好一点，也许就是 28 分钟、25 分钟。

shuffle 调优，15 分钟。groupByKey 用 reduceByKey 改写，执行本地聚合，也许 10 分钟。跟公司申请更多的资源，比如资源更大的 YARN 队列，1 分钟。

### **fastutil\*\***的使用 \*\*

在 pom.xml 中引用 fastutil 的包

```xml
<dependency>
    <groupId>fastutil</groupId>
    <artifactId>fastutil</artifactId>
    <version>5.0.9</version>
</dependency>
```

速度比较慢，可能是从国外的网去拉取 jar 包，可能要等待 5 分钟，甚至几十分钟，不等

List<Integer> 相当于 IntList

基本都是类似于 IntList 的格式，前缀就是集合的元素类型。特殊的就是 Map，Int2IntMap，代表了 key-value 映射的元素类型。除此之外，还支持 object、reference。

**7**

**调节数据本地化等待时长**

### **task\*\***的 \***\*locality\*\***有五种 \*\*

1）、PROCESS_LOCAL：进程本地化，代码和数据在同一个进程中，也就是在同一个 executor 中。计算数据的 task 由 executor 执行，数据在 executor 的 BlockManager 中，性能最好。

2）、NODE_LOCAL：节点本地化，代码和数据在同一个节点中。比如说，数据作为一个 HDFS block 块，就在节点上，而 task 在节点上某个 executor 中运行，或者是，数据和 task 在一个节点上的不同 executor 中，数据需要在进程间进行传输。

3）、NO_PREF：对于 task 来说，数据从哪里获取都一样，没有好坏之分。

4）、RACK_LOCAL：机架本地化，数据和 task 在一个机架的两个节点上，数据需要通过网络在节点之间进行传输。

5）、ANY：数据和 task 可能在集群中的任何地方，而且不在一个机架中，性能最差。

### **Spark\*\***的任务调度 \*\*

Spark 在 Driver 上，对 Application 的每一个 stage 的 task 进行分配之前都会计算出每个 task 要计算的是哪个分片数据。Spark 的 task 分配算法优先会希望每个 task 正好分配到它要计算的数据所在的节点，这样的话，就不用在网络间传输数据。

但是，有时可能 task 没有机会分配到它的数据所在的节点。为什么呢，可能那个节点的计算资源和计算能力都满了。所以这种时候， Spark 会等待一段时间，默认情况下是 3s（不是绝对的，还有很多种情况，对不同的本地化级别，都会去等待），到最后，实在是等待不了了，就会选择一个比较差的本地化级别。比如说，将 task 分配到靠它要计算的数据所在节点比较近的一个节点，然后进行计算。

但是对于第二种情况，通常来说，肯定是要发生数据传输，task 会通过其所在节点的 BlockManager 来获取数据，BlockManager 发现自己本地没有数据，会通过一个 getRemote() 方法，通过 TransferService（网络数据传输组件）从数据所在节点的 BlockManager 中，获取数据，通过网络传输回 task 所在节点。

对于我们来说，当然不希望是类似于第二种情况的了。最好的，当然是 task 和数据在一个节点上，直接从本地 executor 的 BlockManager 中获取数据，纯内存，或者带一点磁盘 IO。如果要通过网络传输数据的话，性能肯定会下降的。大量网络传输，以及磁盘 IO，都是性能的杀手。

### **我们什么时候要调节这个参数**

观察 spark 作业的运行日志。推荐大家在测试的时候，先用 client 模式在本地就直接可以看到比较全的日志。日志里面会显示：starting task…，PROCESS LOCAL、NODE LOCAL

观察大部分 task 的数据本地化级别，如果大多都是 PROCESS_LOCAL，那就不用调节了。

如果是发现，好多的级别都是 NODE_LOCAL、ANY，那么最好就去调节一下数据本地化的等待时长。要反复调节，每次调节完以后再运行并观察日志，看看大部分的 task 的本地化级别有没有提升，看看整个 spark 作业的运行时间有没有缩短。注意，不要本末倒置，不要本地化级别是提升了，但是因为大量的等待时长，spark 作业的运行时间反而增加了，那还是不要调节了。

### **怎么调节**

spark.locality.wait，默认是 3s。6s，10s

默认情况下，下面 3 个的等待时长，都是跟上面那个是一样的，都是 3s

```css
spark.locality.wait.process
spark.locality.wait.node
spark.locality.wait.rack
new SparkConf().set("spark.locality.wait", "10")
```

**2**

**jvm 调优**

堆内存存放我们创建的一些对象，有老年代和年轻代。理想情况下，老年代都是放一些生命周期很长的对象，数量应该是很少的，比如数据库连接池。我们在 spark task 执行算子函数（我们自己写的），可能会创建很多对象，这些对象都是要放入 JVM 年轻代中的。

每一次放对象的时候，都是放入 eden 区域，和其中一个 survivor 区域。另外一个 survivor 区域是空闲的。

当 eden 区域和一个 survivor 区域放满了以后（spark 运行过程中，产生的对象实在太多了），就会触发 minor gc，小型垃圾回收。把不再使用的对象，从内存中清空，给后面新创建的对象腾出来点儿地方。

清理掉了不再使用的对象之后，那么也会将存活下来的对象（还要继续使用的），放入之前空闲的那一个 survivor 区域中。这里可能会出现一个问题。默认 eden、survior1 和 survivor2 的内存占比是 8:1:1。问题是，如果存活下来的对象是 1.5，一个 survivor 区域放不下。此时就可能通过 JVM 的担保机制（不同 JVM 版本可能对应的行为），将多余的对象，直接放入老年代了。

如果你的 JVM 内存不够大的话，可能导致频繁的年轻代内存满溢，频繁的进行 minor gc。频繁的 minor gc 会导致短时间内，有些存活的对象，多次垃圾回收都没有回收掉。会导致这种短生命周期（其实不一定是要长期使用的）对象，年龄过大，垃圾回收次数太多还没有回收到，跑到老年代。

老年代中，可能会因为内存不足，囤积一大堆，短生命周期的，本来应该在年轻代中的，可能马上就要被回收掉的对象。此时，可能导致老年代频繁满溢。频繁进行 full gc（全局 / 全面垃圾回收）。full gc 就会去回收老年代中的对象。full gc 由于这个算法的设计，是针对的是，老年代中的对象数量很少，满溢进行 full gc 的频率应该很少，因此采取了不太复杂，但是耗费性能和时间的垃圾回收算法。full gc 很慢。

full gc / minor gc，无论是快，还是慢，都会导致 jvm 的工作线程停止工作，stop the world。简而言之，就是说，gc 的时候，spark 停止工作了。等着垃圾回收结束。

内存不充足的时候，出现的问题：

1、频繁 minor gc，也会导致频繁 spark 停止工作

2、老年代囤积大量活跃对象（短生命周期的对象），导致频繁 full gc，full gc 时间很长，短则数十秒，长则数分钟，甚至数小时。可能导致 spark 长时间停止工作。

3、严重影响咱们的 spark 的性能和运行的速度。

## **降低 cache 操作的内存占比**

spark 中，堆内存又被划分成了两块，一块是专门用来给 RDD 的 cache、persist 操作进行 RDD 数据缓存用的。另外一块用来给 spark 算子函数的运行使用的，存放函数中自己创建的对象。

默认情况下，给 RDD cache 操作的内存占比，是 0.6，60% 的内存都给了 cache 操作了。但是问题是，如果某些情况下 cache 不是那么的紧张，问题在于 task 算子函数中创建的对象过多，然后内存又不太大，导致了频繁的 minor gc，甚至频繁 full gc，导致 spark 频繁的停止工作。性能影响会很大。

针对上述这种情况，可以在任务运行界面，去查看你的 spark 作业的运行统计，可以看到每个 stage 的运行情况，包括每个 task 的运行时间、gc 时间等等。如果发现 gc 太频繁，时间太长。此时就可以适当调价这个比例。

降低 cache 操作的内存占比，大不了用 persist 操作，选择将一部分缓存的 RDD 数据写入磁盘，或者序列化方式，配合 Kryo 序列化类，减少 RDD 缓存的内存占用。降低 cache 操作内存占比，对应的，算子函数的内存占比就提升了。这个时候，可能就可以减少 minor gc 的频率，同时减少 full gc 的频率。对性能的提升是有一定的帮助的。

一句话，让 task 执行算子函数时，有更多的内存可以使用。

spark.storage.memoryFraction，0.6 -> 0.5 -> 0.4 -> 0.2

## **调节 executor 堆外内存与连接等待时长**

调节 executor 堆外内存

有时候，如果你的 spark 作业处理的数据量特别大，几亿数据量。然后 spark 作业一运行，时不时的报错，shuffle file cannot find，executor、task lost，out of memory（内存溢出）。

可能是 executor 的堆外内存不太够用，导致 executor 在运行的过程中，可能会内存溢出，可能导致后续的 stage 的 task 在运行的时候，要从一些 executor 中去拉取 shuffle map output 文件，但是 executor 可能已经挂掉了，关联的 block manager 也没有了。所以会报 shuffle output file not found，resubmitting task，executor lost。spark 作业彻底崩溃。

上述情况下，就可以去考虑调节一下 executor 的堆外内存。也许就可以避免报错。此外，有时堆外内存调节的比较大的时候，对于性能来说，也会带来一定的提升。

可以调节堆外内存的上限：

\--conf spark.yarn.executor.memoryOverhead=2048

spark-submit 脚本里面，去用 --conf 的方式，去添加配置。用 new SparkConf().set() 这种方式去设置是没有用的！一定要在 spark-submit 脚本中去设置。

spark.yarn.executor.memoryOverhead（看名字，顾名思义，针对的是基于 yarn 的提交模式）

默认情况下，这个堆外内存上限大概是 300M。通常在项目中，真正处理大数据的时候，这里都会出现问题，导致 spark 作业反复崩溃，无法运行。此时就会去调节这个参数，到至少 1G（1024M），甚至说 2G、4G。

通常这个参数调节上去以后，就会避免掉某些 JVM OOM 的异常问题，同时呢，会让整体 spark 作业的性能，得到较大的提升。

**调节连接等待时长**

我们知道，executor 会优先从自己本地关联的 BlockManager 中获取某份数据。如果本地 block manager 没有的话，那么会通过 TransferService，去远程连接其他节点上 executor 的 block manager 去获取。

而此时上面 executor 去远程连接的那个 executor，因为 task 创建的对象特别大，特别多，

频繁的让 JVM 堆内存满溢，正在进行垃圾回收。而处于垃圾回收过程中，所有的工作线程全部停止，相当于只要一旦进行垃圾回收，spark / executor 停止工作，无法提供响应。

此时呢，就会没有响应，无法建立网络连接，会卡住。spark 默认的网络连接的超时时长，是 60s，如果卡住 60s 都无法建立连接的话，那么就宣告失败了。

报错几次，几次都拉取不到数据的话，可能会导致 spark 作业的崩溃。也可能会导致 DAGScheduler，反复提交几次 stage。TaskScheduler 反复提交几次 task。大大延长我们的 spark 作业的运行时间。

可以考虑调节连接的超时时长：

\--conf spark.core.connection.ack.wait.timeout=300

spark-submit 脚本，切记，不是在 new SparkConf().set() 这种方式来设置的。

spark.core.connection.ack.wait.timeout（spark core，connection，连接，ack，wait timeout，建立不上连接的时候，超时等待时长）

调节这个值比较大以后，通常来说，可以避免部分的偶尔出现的某某文件拉取失败，某某文件 lost 掉了。

**3**

**shuffle 调优**

**原理概述：** 

什么样的情况下，会发生 shuffle？

在 spark 中，主要是以下几个算子：groupByKey、reduceByKey、countByKey、join（分情况，先 groupByKey 后再 join 是不会发生 shuffle 的），等等。

什么是 shuffle？

groupByKey，要把分布在集群各个节点上的数据中的同一个 key，对应的 values，都要集中到一块儿，集中到集群中同一个节点上，更严密一点说，就是集中到一个节点的一个 executor 的一个 task 中。

然后呢，集中一个 key 对应的 values 之后，才能交给我们来进行处理，&lt;key, Iterable<value>>。reduceByKey，算子函数去对 values 集合进行 reduce 操作，最后变成一个 value。countByKey 需要在一个 task 中，获取到一个 key 对应的所有的 value，然后进行计数，统计一共有多少个 value。join，RDD&lt;key, value>，RDD&lt;key, value>，只要是两个 RDD 中，key 相同对应的 2 个 value，都能到一个节点的 executor 的 task 中，给我们进行处理。

shuffle，一定是分为两个 stage 来完成的。因为这其实是个逆向的过程，不是 stage 决定 shuffle，是 shuffle 决定 stage。

reduceByKey(_+_)，在某个 action 触发 job 的时候，DAGScheduler，会负责划分 job 为多个 stage。划分的依据，就是，如果发现有会触发 shuffle 操作的算子，比如 reduceByKey，就将这个操作的前半部分，以及之前所有的 RDD 和 transformation 操作，划分为一个 stage。shuffle 操作的后半部分，以及后面的，直到 action 为止的 RDD 和 transformation 操作，划分为另外一个 stage。

**1**

\***\* 合并 map 端输出文件**
\--------------\*\*

### **如果不合并 \*\***map\***\* 端输出文件的话，会怎么样？**

举例实际生产环境的条件：

100 个节点（每个节点一个 executor）：100 个 executor

每个 executor：2 个 cpu core

总共 1000 个 task：每个 executor 平均 10 个 task

每个节点，10 个 task，每个节点会输出多少份 map 端文件？10 \* 1000=1 万个文件

总共有多少份 map 端输出文件？100 \* 10000 = 100 万。

第一个 stage，每个 task，都会给第二个 stage 的每个 task 创建一份 map 端的输出文件

第二个 stage，每个 task，会到各个节点上面去，拉取第一个 stage 每个 task 输出的，属于自己的那一份文件。

shuffle 中的写磁盘的操作，基本上就是 shuffle 中性能消耗最为严重的部分。

通过上面的分析，一个普通的生产环境的 spark job 的一个 shuffle 环节，会写入磁盘 100 万个文件。

磁盘 IO 对性能和 spark 作业执行速度的影响，是极其惊人和吓人的。

基本上，spark 作业的性能，都消耗在 shuffle 中了，虽然不只是 shuffle 的 map 端输出文件这一个部分，但是这里也是非常大的一个性能消耗点。

### **开启 \*\***shuffle map\***\* 端输出文件合并的机制**

new SparkConf().set("spark.shuffle.consolidateFiles", "true")

默认情况下，是不开启的，就是会发生如上所述的大量 map 端输出文件的操作，严重影响性能。

### **合并 \*\***map\***\* 端输出文件，对咱们的 \*\***spark\***\* 的性能有哪些方面的影响呢？**

1、map task 写入磁盘文件的 IO，减少：100 万文件 -> 20 万文件

2、第二个 stage，原本要拉取第一个 stage 的 task 数量份文件，1000 个 task，第二个 stage 的每个 task，都要拉取 1000 份文件，走网络传输。合并以后，100 个节点，每个节点 2 个 cpu core，第二个 stage 的每个 task，主要拉取 100 \* 2 = 200 个文件即可。此时网络传输的性能消耗也大大减少。

分享一下，实际在生产环境中，使用了 spark.shuffle.consolidateFiles 机制以后，实际的性能调优的效果：对于上述的这种生产环境的配置，性能的提升，还是相当的可观的。spark 作业，5 个小时 -> 2~3 个小时。

大家不要小看这个 map 端输出文件合并机制。实际上，在数据量比较大，你自己本身做了前面的性能调优，executor 上去 ->cpu core 上去 -> 并行度（task 数量）上去，shuffle 没调优，shuffle 就很糟糕了。大量的 map 端输出文件的产生，对性能有比较恶劣的影响。

这个时候，去开启这个机制，可以很有效的提升性能。

**2**

\***\* 调节 map 端内存缓冲与 reduce 端内存占比**
\--------------------------\*\*

### **默认情况下可能出现的问题**

默认情况下，shuffle 的 map task，输出到磁盘文件的时候，统一都会先写入每个 task 自己关联的一个内存缓冲区。

这个缓冲区大小，默认是 32kb。

每一次，当内存缓冲区满溢之后，才会进行 spill 溢写操作，溢写到磁盘文件中去。

reduce 端 task，在拉取到数据之后，会用 hashmap 的数据格式，来对各个 key 对应的 values 进行汇聚。

针对每个 key 对应的 values，执行我们自定义的聚合函数的代码，比如_ + _（把所有 values 累加起来）。

reduce task，在进行汇聚、聚合等操作的时候，实际上，使用的就是自己对应的 executor 的内存，executor（jvm 进程，堆），默认 executor 内存中划分给 reduce task 进行聚合的比例是 0.2。

问题来了，因为比例是 0.2，所以，理论上，很有可能会出现，拉取过来的数据很多，那么在内存中，放不下。这个时候，默认的行为就是将在内存放不下的数据都 spill（溢写）到磁盘文件中去。

在数据量比较大的情况下，可能频繁地发生 reduce 端的磁盘文件的读写。

### **调优方式**

调节 map task 内存缓冲：spark.shuffle.file.buffer，默认 32k（spark 1.3.x 不是这个参数，后面还有一个后缀，kb。spark 1.5.x 以后，变了，就是现在这个参数）

调节 reduce 端聚合内存占比：spark.shuffle.memoryFraction，0.2

### **在实际生产环境中，我们在什么时候来调节两个参数？**

看 Spark UI，如果你的公司是决定采用 standalone 模式，那么狠简单，你的 spark 跑起来，会显示一个 Spark UI 的地址，4040 的端口。进去观察每个 stage 的详情，有哪些 executor，有哪些 task，每个 task 的 shuffle write 和 shuffle read 的量，shuffle 的磁盘和内存读写的数据量。如果是用的 yarn 模式来提交，从 yarn 的界面进去，点击对应的 application，进入 Spark UI，查看详情。

如果发现 shuffle 磁盘的 write 和 read，很大。这个时候，就意味着最好调节一些 shuffle 的参数。首先当然是考虑开启 map 端输出文件合并机制。其次调节上面说的那两个参数。调节的时候的原则：spark.shuffle.file.buffer 每次扩大一倍，然后看看效果，64，128。spark.shuffle.memoryFraction，每次提高 0.1，看看效果。

不能调节的太大，太大了以后过犹不及，因为内存资源是有限的，你这里调节的太大了，其他环节的内存使用就会有问题了。

### **调节以后的效果**

map task 内存缓冲变大了，减少 spill 到磁盘文件的次数。reduce 端聚合内存变大了，减少 spill 到磁盘的次数，而且减少了后面聚合读取磁盘文件的数量。

**3**

\***\*HashShuffleManager 与 SortShuffleManager**
\-----------------------------------------\*\*

### **shuffle 调优概述**

大多数 Spark 作业的性能主要就是消耗在了 shuffle 环 节，因为该环节包含了大量的磁盘 IO、序列化、网络数据传输等操作。因此，如果要让作业的性能更上一层楼，就有必要对 shuffle 过程进行调优。但是也 必须提醒大家的是，影响一个 Spark 作业性能的因素，主要还是代码开发、资源参数以及数据倾斜，shuffle 调优只能在整个 Spark 的性能调优中占 到一小部分而已。因此大家务必把握住调优的基本原则，千万不要舍本逐末。下面我们就给大家详细讲解 shuffle 的原理，以及相关参数的说明，同时给出各个参数的调优建议。

### **ShuffleManager 发展概述**

在 Spark 的源码中，负责 shuffle 过程的执行、计算和处理的组件主要就是 ShuffleManager，也即 shuffle 管理器。

在 Spark 1.2 以前，默认的 shuffle 计算引擎是 HashShuffleManager。该 ShuffleManager 而 HashShuffleManager 有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘 IO 操作影响了性能。

因此在 Spark 1.2 以后的版本中，默认的 ShuffleManager 改成了 SortShuffleManager。SortShuffleManager 相较于 HashShuffleManager 来说，有了一定的改进。主要就在于，每个 Task 在进行 shuffle 操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并（merge）成一个磁盘文件，因此每个 Task 就只有一个磁盘文件。在下一个 stage 的 shuffle read task 拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

在 spark 1.5.x 以后，对于 shuffle manager 又出来了一种新的 manager，tungsten-sort（钨丝），钨丝 sort shuffle manager。官网上一般说，钨丝 sort shuffle manager，效果跟 sort shuffle manager 是差不多的。

但是，唯一的不同之处在于，钨丝 manager，是使用了自己实现的一套内存管理机制，性能上有很大的提升， 而且可以避免 shuffle 过程中产生的大量的 OOM，GC，等等内存相关的异常。

### **hash\*\***、\***\*sort\*\***、\***\*tungsten-sort\*\***。如何来选择？\*\*

1、需不需要数据默认就让 spark 给你进行排序？就好像 mapreduce，默认就是有按照 key 的排序。如果不需要的话，其实还是建议搭建就使用最基本的 HashShuffleManager，因为最开始就是考虑的是不排序，换取高性能。

2、什么时候需要用 sort shuffle manager？如果你需要你的那些数据按 key 排序了，那么就选择这种吧，而且要注意，reduce task 的数量应该是超过 200 的，这样 sort、merge（多个文件合并成一个）的机制，才能生效把。但是这里要注意，你一定要自己考量一下，有没有必要在 shuffle 的过程中，就做这个事情，毕竟对性能是有影响的。

3、如果你不需要排序，而且你希望你的每个 task 输出的文件最终是会合并成一份的，你自己认为可以减少性能开销。可以去调节 bypassMergeThreshold 这个阈值，比如你的 reduce task 数量是 500，默认阈值是 200，所以默认还是会进行 sort 和直接 merge 的。可以将阈值调节成 550，不会进行 sort，按照 hash 的做法，每个 reduce task 创建一份输出文件，最后合并成一份文件。（一定要提醒大家，这个参数，其实我们通常不会在生产环境里去使用，也没有经过验证说，这样的方式，到底有多少性能的提升）

4、如果你想选用 sort based shuffle manager，而且你们公司的 spark 版本比较高，是 1.5.x 版本的，那么可以考虑去尝试使用 tungsten-sort shuffle manager。看看性能的提升与稳定性怎么样。

总结：

1、在生产环境中，不建议大家贸然使用第三点和第四点：

2、如果你不想要你的数据在 shuffle 时排序，那么就自己设置一下，用 hash shuffle manager。

3、如果你的确是需要你的数据在 shuffle 时进行排序的，那么就默认不用动，默认就是 sort shuffle manager。或者是什么？如果你压根儿不 care 是否排序这个事儿，那么就默认让他就是 sort 的。调节一些其他的参数（consolidation 机制）。（80%，都是用这种）

spark.shuffle.manager：hash、sort、tungsten-sort

spark.shuffle.sort.bypassMergeThreshold：200。自己可以设定一个阈值，默认是 200，当 reduce task 数量少于等于 200，map task 创建的输出文件小于等于 200 的，最后会将所有的输出文件合并为一份文件。这样做的好处，就是避免了 sort 排序，节省了性能开销，而且还能将多个 reduce task 的文件合并成一份文件，节省了 reduce task 拉取数据的时候的磁盘 IO 的开销。

**4**

**算子调优**

**1**

\***\*MapPartitions 提升 Map 类操作性能**
\---------------------------\*\*

spark 中，最基本的原则，就是每个 task 处理一个 RDD 的 partition。

### **MapPartitions 的优缺点**

MapPartitions 操作的优点：

如果是普通的 map，比如一个 partition 中有 1 万条数据。ok，那么你的 function 要执行和计算 1 万次。

但是，使用 MapPartitions 操作之后，一个 task 仅仅会执行一次 function，function 一次接收所有的 partition 数据。只要执行一次就可以了，性能比较高。

MapPartitions 的缺点：

如果是普通的 map 操作，一次 function 的执行就处理一条数据。那么如果内存不够用的情况下，比如处理了 1 千条数据了，那么这个时候内存不够了，那么就可以将已经处理完的 1 千条数据从内存里面垃圾回收掉，或者用其他方法，腾出空间来吧。

所以说普通的 map 操作通常不会导致内存的 OOM 异常。

但是 MapPartitions 操作，对于大量数据来说，比如甚至一个 partition，100 万数据，一次传入一个 function 以后，那么可能一下子内存不够，但是又没有办法去腾出内存空间来，可能就 OOM，内存溢出。

### **MapPartitions 使用场景**

当分析的数据量不是特别大的时候，都可以用这种 MapPartitions 系列操作，性能还是非常不错的，是有提升的。比如原来是 15 分钟，（曾经有一次性能调优），12 分钟。10 分钟 ->9 分钟。

但是也有过出问题的经验，MapPartitions 只要一用，直接 OOM，内存溢出，崩溃。

在项目中，自己先去估算一下 RDD 的数据量，以及每个 partition 的量，还有自己分配给每个 executor 的内存资源。看看一下子内存容纳所有的 partition 数据行不行。如果行，可以试一下，能跑通就好。性能肯定是有提升的。但是试了以后，发现 OOM 了，那就放弃吧。

**2**

\***\*filter 过后使用 coalesce 减少分区数量**
\----------------------------\*\*

### **出现问题**

默认情况下，经过了 filter 之后，RDD 中的每个 partition 的数据量，可能都不太一样了。（原本每个 partition 的数据量可能是差不多的）

可能出现的问题：

1、每个 partition 数据量变少了，但是在后面进行处理的时候，还是要跟 partition 数量一样数量的 task，来进行处理，有点浪费 task 计算资源。

2、每个 partition 的数据量不一样，会导致后面的每个 task 处理每个 partition 的时候，每个 task 要处理的数据量就不同，这样就会导致有些 task 运行的速度很快，有些 task 运行的速度很慢。这就是数据倾斜。

针对上述的两个问题，我们希望应该能够怎么样？

1、针对第一个问题，我们希望可以进行 partition 的压缩吧，因为数据量变少了，那么 partition 其实也完全可以对应的变少。比如原来是 4 个 partition，现在完全可以变成 2 个 partition。那么就只要用后面的 2 个 task 来处理即可。就不会造成 task 计算资源的浪费。（不必要，针对只有一点点数据的 partition，还去启动一个 task 来计算）

2、针对第二个问题，其实解决方案跟第一个问题是一样的，也是去压缩 partition，尽量让每个 partition 的数据量差不多。那么后面的 task 分配到的 partition 的数据量也就差不多。不会造成有的 task 运行速度特别慢，有的 task 运行速度特别快。避免了数据倾斜的问题。

### **解决问题方法**

调用 coalesce 算子

主要就是用于在 filter 操作之后，针对每个 partition 的数据量各不相同的情况，来压缩 partition 的数量，而且让每个 partition 的数据量都尽量均匀紧凑。从而便于后面的 task 进行计算操作，在某种程度上，能够一定程度的提升性能。

**3**

\***\* 使用 foreachPartition 优化写数据库性能**
\------------------------------\*\*

### **默认的 \*\***foreach\***\* 的性能缺陷在哪里？**

首先，对于每条数据，都要单独去调用一次 function，task 为每个数据，都要去执行一次 function 函数。

如果 100 万条数据，（一个 partition），调用 100 万次。性能比较差。

另外一个非常非常重要的一点

如果每个数据，你都去创建一个数据库连接的话，那么你就得创建 100 万次数据库连接。

但是要注意的是，数据库连接的创建和销毁，都是非常非常消耗性能的。虽然我们之前已经用了数据库连接池，只是创建了固定数量的数据库连接。

你还是得多次通过数据库连接，往数据库（MySQL）发送一条 SQL 语句，然后 MySQL 需要去执行这条 SQL 语句。如果有 100 万条数据，那么就是 100 万次发送 SQL 语句。

以上两点（数据库连接，多次发送 SQL 语句），都是非常消耗性能的。

### **用了 \*\***foreachPartition\***\* 算子之后，好处在哪里？**

1、对于我们写的 function 函数，就调用一次，一次传入一个 partition 所有的数据。

2、主要创建或者获取一个数据库连接就可以。

3、只要向数据库发送一次 SQL 语句和多组参数即可。

注意，与 mapPartitions 操作一样，如果一个 partition 的数量真的特别特别大，比如是 100 万，那基本上就不太靠谱了。很有可能会发生 OOM，内存溢出的问题。

**4**

\***\* 使用 repartition 解决 Spark SQL 低并行度的性能问题**
\-------------------------------------\*\*

### **设置并行度**

并行度：之前说过，并行度是设置的：

1、spark.default.parallelism

2、textFile()，传入第二个参数，指定 partition 数量（比较少用）

在生产环境中，是最好设置一下并行度。官网有推荐的设置方式，根据你的 application 的总 cpu core 数量（在 spark-submit 中可以指定），自己手动设置 spark.default.parallelism 参数，指定为 cpu core 总数的 2~3 倍。

### **你设置的这个并行度，在哪些情况下会生效？什么情况下不会生效？**

如果你压根儿没有使用 Spark SQL（DataFrame），那么你整个 spark application 默认所有 stage 的并行度都是你设置的那个参数。（除非你使用 coalesce 算子缩减过 partition 数量）。

问题来了，用 Spark SQL 的情况下，stage 的并行度没法自己指定。Spark SQL 自己会默认根据 hive 表对应的 hdfs 文件的 block，自动设置 Spark SQL 查询所在的那个 stage 的并行度。你自己通过 spark.default.parallelism 参数指定的并行度，只会在没有 Spark SQL 的 stage 中生效。

比如你第一个 stage，用了 Spark SQL 从 hive 表中查询出了一些数据，然后做了一些 transformation 操作，接着做了一个 shuffle 操作（groupByKey）。下一个 stage，在 shuffle 操作之后，做了一些 transformation 操作。hive 表，对应了一个 hdfs 文件，有 20 个 block。你自己设置了 spark.default.parallelism 参数为 100。

你的第一个 stage 的并行度，是不受你的控制的，就只有 20 个 task。第二个 stage，才会变成你自己设置的那个并行度，100。

### **可能出现的问题？**

Spark SQL 默认情况下，它的那个并行度，咱们没法设置。可能导致的问题，也许没什么问题，也许很有问题。Spark SQL 所在的那个 stage 中，后面的那些 transformation 操作，可能会有非常复杂的业务逻辑，甚至说复杂的算法。如果你的 Spark SQL 默认把 task 数量设置的很少，20 个，然后每个 task 要处理为数不少的数据量，然后还要执行特别复杂的算法。

这个时候，就会导致第一个 stage 的速度，特别慢。第二个 stage1000 个 task 非常快。

### **解决 \*\***Spark SQL\***\* 无法设置并行度和 \*\***task\***\* 数量的办法**

repartition 算子，你用 Spark SQL 这一步的并行度和 task 数量，肯定是没有办法去改变了。但是呢，可以将你用 Spark SQL 查询出来的 RDD，使用 repartition 算子去重新进行分区，此时可以分成多个 partition。然后呢，从 repartition 以后的 RDD，再往后，并行度和 task 数量，就会按照你预期的来了。就可以避免跟 Spark SQL 绑定在一个 stage 中的算子，只能使用少量的 task 去处理大量数据以及复杂的算法逻辑。

**5**

\***\*reduceByKey 本地聚合介绍**
\---------------------\*\*

reduceByKey，相较于普通的 shuffle 操作（比如 groupByKey），它的一个特点，就是说，会进行 map 端的本地聚合。对 map 端给下个 stage 每个 task 创建的输出文件中，写数据之前，就会进行本地的 combiner 操作，也就是说对每一个 key，对应的 values，都会执行你的算子函数（_ + _）

### **用 \*\***reduceByKey\***\* 对性能的提升**

1、在本地进行聚合以后，在 map 端的数据量就变少了，减少磁盘 IO。而且可以减少磁盘空间的占用。

2、下一个 stage，拉取数据的量，也就变少了。减少网络的数据传输的性能消耗。

3、在 reduce 端进行数据缓存的内存占用变少了。

4、reduce 端，要进行聚合的数据量也变少了。

### **reduceByKey\*\***在什么情况下使用呢？\*\*

1、非常普通的，比如说，就是要实现类似于 wordcount 程序一样的，对每个 key 对应的值，进行某种数据公式或者算法的计算（累加、类乘）。

2、对于一些类似于要对每个 key 进行一些字符串拼接的这种较为复杂的操作，可以自己衡量一下，其实有时，也是可以使用 reduceByKey 来实现的。但是不太好实现。如果真能够实现出来，对性能绝对是有帮助的。（shuffle 基本上就占了整个 spark 作业的 90% 以上的性能消耗，主要能对 shuffle 进行一定的调优，都是有价值的）

**5**

**troubleshooting**

**1**

\***\* 控制 shuffle reduce 端缓冲大小以避免 OOM**
\-------------------------------\*\*

map 端的 task 是不断的输出数据的，数据量可能是很大的。

但是，其实 reduce 端的 task，并不是等到 map 端 task 将属于自己的那份数据全部写入磁盘文件之后，再去拉取的。map 端写一点数据，reduce 端 task 就会拉取一小部分数据，立即进行后面的聚合、算子函数的应用。

每次 reduece 能够拉取多少数据，就由 buffer 来决定。因为拉取过来的数据，都是先放在 buffer 中的。然后才用后面的 executor 分配的堆内存占比（0.2），hashmap，去进行后续的聚合、函数的执行。

### **reduce\*\***端缓冲大小的另外一面，关于性能调优的一面 \*\*

假如 Map 端输出的数据量也不是特别大，然后你的整个 application 的资源也特别充足。200 个 executor、5 个 cpu core、10G 内存。

其实可以尝试去增加这个 reduce 端缓冲大小的，比如从 48M，变成 96M。那么这样的话，每次 reduce task 能够拉取的数据量就很大。需要拉取的次数也就变少了。比如原先需要拉取 100 次，现在只要拉取 50 次就可以执行完了。

对网络传输性能开销的减少，以及 reduce 端聚合操作执行的次数的减少，都是有帮助的。

最终达到的效果，就应该是性能上的一定程度上的提升。

注意，一定要在资源充足的前提下做此操作。

### **reduce\*\***端缓冲（\***\*buffer\*\***），可能会出现的问题及解决方式 \*\*

可能会出现，默认是 48MB，也许大多数时候，reduce 端 task 一边拉取一边计算，不一定一直都会拉满 48M 的数据。大多数时候，拉取个 10M 数据，就计算掉了。

大多数时候，也许不会出现什么问题。但是有的时候，map 端的数据量特别大，然后写出的速度特别快。reduce 端所有 task，拉取的时候，全部达到自己的缓冲的最大极限值，缓冲区 48M，全部填满。

这个时候，再加上你的 reduce 端执行的聚合函数的代码，可能会创建大量的对象。也许，一下子内存就撑不住了，就会 OOM。reduce 端的内存中，就会发生内存溢出的问题。

针对上述的可能出现的问题，我们该怎么来解决呢？

这个时候，就应该减少 reduce 端 task 缓冲的大小。我宁愿多拉取几次，但是每次同时能够拉取到 reduce 端每个 task 的数量比较少，就不容易发生 OOM 内存溢出的问题。（比如，可以调节成 12M）

在实际生产环境中，我们都是碰到过这种问题的。这是典型的以性能换执行的原理。reduce 端缓冲小了，不容易 OOM 了，但是，性能一定是有所下降的，你要拉取的次数就多了。就走更多的网络传输开销。

这种时候，只能采取牺牲性能的方式了，spark 作业，首先，第一要义，就是一定要让它可以跑起来。

### **操作方法**

```css
new SparkConf().set(spark.reducer.maxSizeInFlight，”48”)
```

**2**

\***\* 解决 JVM GC 导致的 shuffle 文件拉取失败**
\----------------------------\*\*

### **问题 \*\***描述 \*\*

有时会出现一种情况，在 spark 的作业中，log 日志会提示 shuffle file not found。（spark 作业中，非常常见的）而且有的时候，它是偶尔才会出现的一种情况。有的时候，出现这种情况以后，重新去提交 task。重新执行一遍，发现就好了。没有这种错误了。

log 怎么看？用 client 模式去提交你的 spark 作业。比如 standalone client 或 yarn client。一提交作业，直接可以在本地看到更新的 log。

问题原因：比如，executor 的 JVM 进程可能内存不够用了。那么此时就会执行 GC。minor GC or full GC。此时就会导致 executor 内，所有的工作线程全部停止。比如 BlockManager，基于 netty 的网络通信。

下一个 stage 的 executor，可能还没有停止掉的 task 想要去上一个 stage 的 task 所在的 exeuctor 去拉取属于自己的数据，结果由于对方正在 gc，就导致拉取了半天没有拉取到。

就很可能会报出 shuffle file not found。但是，可能下一个 stage 又重新提交了 task 以后，再执行就没有问题了，因为可能第二次就没有碰到 JVM 在 gc 了。

### **解决方案**

spark.shuffle.io.maxRetries 3

第一个参数，意思就是说，shuffle 文件拉取的时候，如果没有拉取到（拉取失败），最多或重试几次（会重新拉取几次文件），默认是 3 次。

spark.shuffle.io.retryWait 5s

第二个参数，意思就是说，每一次重试拉取文件的时间间隔，默认是 5s 钟。

默认情况下，假如说第一个 stage 的 executor 正在进行漫长的 full gc。第二个 stage 的 executor 尝试去拉取文件，结果没有拉取到，默认情况下，会反复重试拉取 3 次，每次间隔是五秒钟。最多只会等待 3 \* 5s = 15s。如果 15s 内，没有拉取到 shuffle file。就会报出 shuffle file not found。

针对这种情况，我们完全可以进行预备性的参数调节。增大上述两个参数的值，达到比较大的一个值，尽量保证第二个 stage 的 task，一定能够拉取到上一个 stage 的输出文件。避免报 shuffle file not found。然后可能会重新提交 stage 和 task 去执行。那样反而对性能也不好。

spark.shuffle.io.maxRetries 60

spark.shuffle.io.retryWait 60s

最多可以忍受 1 个小时没有拉取到 shuffle file。只是去设置一个最大的可能的值。full gc 不可能 1 个小时都没结束吧。

这样呢，就可以尽量避免因为 gc 导致的 shuffle file not found，无法拉取到的问题。

**3**

\***\*YARN 队列资源不足导致的 application 直接失败**
\--------------------------------\*\*

### **问题描述**

如果说，你是基于 yarn 来提交 spark。比如 yarn-cluster 或者 yarn-client。你可以指定提交到某个 hadoop 队列上的。每个队列都是可以有自己的资源的。

跟大家说一个生产环境中的，给 spark 用的 yarn 资源队列的情况：500G 内存，200 个 cpu core。

比如说，某个 spark application，在 spark-submit 里面你自己配了，executor，80 个。每个 executor，4G 内存。每个 executor，2 个 cpu core。你的 spark 作业每次运行，大概要消耗掉 320G 内存，以及 160 个 cpu core。

乍看起来，咱们的队列资源，是足够的，500G 内存，280 个 cpu core。

首先，第一点，你的 spark 作业实际运行起来以后，耗费掉的资源量，可能是比你在 spark-submit 里面配置的，以及你预期的，是要大一些的。400G 内存，190 个 cpu core。

那么这个时候，的确，咱们的队列资源还是有一些剩余的。但问题是如果你同时又提交了一个 spark 作业上去，一模一样的。那就可能会出问题。

第二个 spark 作业，又要申请 320G 内存 + 160 个 cpu core。结果，发现队列资源不足。

此时，可能会出现两种情况：（备注，具体出现哪种情况，跟你的 YARN、Hadoop 的版本，你们公司的一些运维参数，以及配置、硬件、资源肯能都有关系）

1、YARN，发现资源不足时，你的 spark 作业，并没有 hang 在那里，等待资源的分配，而是直接打印一行 fail 的 log，直接就 fail 掉了。

2、YARN，发现资源不足，你的 spark 作业，就 hang 在那里。一直等待之前的 spark 作业执行完，等待有资源分配给自己来执行。

### **解决方案**

1、在你的 J2EE（我们这个项目里面，spark 作业的运行， J2EE 平台触发的，执行 spark-submit 脚本的平台）进行限制，同时只能提交一个 spark 作业到 yarn 上去执行，确保一个 spark 作业的资源肯定是有的。

2、你应该采用一些简单的调度区分的方式，比如说，有的 spark 作业可能是要长时间运行的，比如运行 30 分钟。有的 spark 作业，可能是短时间运行的，可能就运行 2 分钟。此时，都提交到一个队列上去，肯定不合适。很可能出现 30 分钟的作业卡住后面一大堆 2 分钟的作业。分队列，可以申请（跟你们的 YARN、Hadoop 运维的同事申请）。你自己给自己搞两个调度队列。每个队列的根据你要执行的作业的情况来设置。在你的 J2EE 程序里面，要判断，如果是长时间运行的作业，就干脆都提交到某一个固定的队列里面去把。如果是短时间运行的作业，就统一提交到另外一个队列里面去。这样，避免了长时间运行的作业，阻塞了短时间运行的作业。

3、你的队列里面，无论何时，只会有一个作业在里面运行。那么此时，就应该用我们之前讲过的性能调优的手段，去将每个队列能承载的最大的资源，分配给你的每一个 spark 作业，比如 80 个 executor，6G 的内存，3 个 cpu core。尽量让你的 spark 作业每一次运行，都达到最满的资源使用率，最快的速度，最好的性能。并行度，240 个 cpu core，720 个 task。

4、在 J2EE 中，通过线程池的方式（一个线程池对应一个资源队列），来实现上述我们说的方案。

**4**

\***\* 解决各种序列化导致的报错**
\----------------\*\*

### **问题描述**

用 client 模式去提交 spark 作业，观察本地打印出来的 log。如果出现了类似于 Serializable、Serialize 等等字眼报错的 log，那么恭喜大家，就碰到了序列化问题导致的报错。

### **序列化报错及解决方法**

1、你的算子函数里面，如果使用到了外部的自定义类型的变量，那么此时，就要求你的自定义类型，必须是可序列化的。

```java
final Teacher teacher = new Teacher("leo");
studentsRDD.foreach(new VoidFunction() {
public void call(Row row) throws Exception {
   String teacherName = teacher.getName();
   ....  
}
});

public class Teacher implements Serializable {
}
```

2、如果要将自定义的类型，作为 RDD 的元素类型，那么自定义的类型也必须是可以序列化的。

```java
JavaPairRDD<Integer, Teacher> teacherRDD
JavaPairRDD<Integer, Student> studentRDD
studentRDD.join(teacherRDD)
public class Teacher implements Serializable {
}

public class Student implements Serializable {
}
```

3、不能在上述两种情况下，去使用一些第三方的，不支持序列化的类型。

```cs
Connection conn =
studentsRDD.foreach(new VoidFunction() {
public void call(Row row) throws Exception {
  conn.....
}
});
```

Connection 是不支持序列化的

**5**

\***\* 解决算子函数返回 NULL 导致的问题**
\---------------------\*\*

### **问题描述**

在有些算子函数里面，是需要我们有一个返回值的。但是，有时候不需要返回值。我们如果直接返回 NULL 的话，是会报错的。

Scala.Math(NULL)，异常

### **解决方案**

如果碰到你的确是对于某些值不想要有返回值的话，有一个解决的办法：

1、在返回的时候，返回一些特殊的值，不要返回 null，比如 “-999”

2、在通过算子获取到了一个 RDD 之后，可以对这个 RDD 执行 filter 操作，进行数据过滤。filter 内，可以对数据进行判定，如果是 - 999，那么就返回 false，给过滤掉就可以了。

3、大家不要忘了，之前咱们讲过的那个算子调优里面的 coalesce 算子，在 filter 之后，可以使用 coalesce 算子压缩一下 RDD 的 partition 的数量，让各个 partition 的数据比较紧凑一些。也能提升一些性能。

**6**

\***\* 解决 yarn-client 模式导致的网卡流量激增问题**
\------------------------------\*\*

### **Spark\*\***-On-Yarn 任务执行流程 \*\*

Driver 到底是什么？

我们写的 spark 程序，打成 jar 包，用 spark-submit 来提交。jar 包中的一个 main 类，通过 jvm 的命令启动起来。

JVM 进程，其实就是 Driver 进程。

Driver 进程启动起来以后，执行我们自己写的 main 函数，从 new SparkContext() 开始。

driver 接收到属于自己的 executor 进程的注册之后，就可以去进行我们写的 spark 作业代码的执行了。此时会一行一行的去执行咱们写的那些 spark 代码。执行到某个 action 操作的时候，就会触发一个 job，然后 DAGScheduler 会将 job 划分为一个一个的 stage，为每个 stage 都创建指定数量的 task。TaskScheduler 将每个 stage 的 task 分配到各个 executor 上面去执行。

task 就会执行咱们写的算子函数。

spark 在 yarn-client 模式下，application 的注册（executor 的申请）和计算 task 的调度，是分离开来的。

standalone 模式下，这两个操作都是 driver 负责的。

ApplicationMaster(ExecutorLauncher) 负责 executor 的申请，driver 负责 job 和 stage 的划分，以及 task 的创建、分配和调度。

每种计算框架（mr、spark），如果想要在 yarn 上执行自己的计算应用，那么就必须自己实现和提供一个 ApplicationMaster。相当于是实现了 yarn 提供的接口，spark 自己开发的一个类。

### **yarn-client\*\***模式下，会产生什么样的问题呢？\*\*

由于 driver 是启动在本地机器的，而且 driver 是全权负责所有的任务的调度的，也就是说要跟 yarn 集群上运行的多个 executor 进行频繁的通信（中间有 task 的启动消息、task 的执行统计消息、task 的运行状态、shuffle 的输出结果）。

想象一下，比如你的 executor 有 100 个，stage 有 10 个，task 有 1000 个。每个 stage 运行的时候，都有 1000 个 task 提交到 executor 上面去运行，平均每个 executor 有 10 个 task。接下来问题来了，driver 要频繁地跟 executor 上运行的 1000 个 task 进行通信。通信消息特别多，通信的频率特别高。运行完一个 stage，接着运行下一个 stage，又是频繁的通信。

在整个 spark 运行的生命周期内，都会频繁的去进行通信和调度。所有这一切通信和调度都是从你的本地机器上发出去的，和接收到的。这是最要命的地方。你的本地机器，很可能在 30 分钟内（spark 作业运行的周期内），进行频繁大量的网络通信。那么此时，你的本地机器的网络通信负载是非常非常高的。会导致你的本地机器的网卡流量会激增！

你的本地机器的网卡流量激增，当然不是一件好事了。因为在一些大的公司里面，对每台机器的使用情况，都是有监控的。不会允许单个机器出现耗费大量网络带宽等等这种资源的情况。

### **解决方案**

实际上解决的方法很简单，就是心里要清楚，yarn-client 模式是什么情况下，可以使用的？yarn-client 模式，通常咱们就只会使用在测试环境中，你写好了某个 spark 作业，打了一个 jar 包，在某台测试机器上，用 yarn-client 模式去提交一下。因为测试的行为是偶尔为之的，不会长时间连续提交大量的 spark 作业去测试。还有一点好处，yarn-client 模式提交，可以在本地机器观察到详细全面的 log。通过查看 log，可以去解决线上报错的故障（troubleshooting）、对性能进行观察并进行性能调优。

实际上线了以后，在生产环境中，都得用 yarn-cluster 模式，去提交你的 spark 作业。

yarn-cluster 模式，就跟你的本地机器引起的网卡流量激增的问题，就没有关系了。也就是说，就算有问题，也应该是 yarn 运维团队和基础运维团队之间的事情了。使用了 yarn-cluster 模式以后，就不是你的本地机器运行 Driver，进行 task 调度了。是 yarn 集群中，某个节点会运行 driver 进程，负责 task 调度。

**7**

\***\* 解决 yarn-cluster 模式的 JVM 栈内存溢出问题**
\-------------------------------\*\*

### **问题描述**

有的时候，运行一些包含了 spark sql 的 spark 作业，可能会碰到 yarn-client 模式下，可以正常提交运行。yarn-cluster 模式下，可能无法提交运行的，会报出 JVM 的 PermGen（永久代）的内存溢出，OOM。

yarn-client 模式下，driver 是运行在本地机器上的，spark 使用的 JVM 的 PermGen 的配置，是本地的 spark-class 文件（spark 客户端是默认有配置的），JVM 的永久代的大小是 128M，这个是没有问题的。但是在 yarn-cluster 模式下，driver 是运行在 yarn 集群的某个节点上的，使用的是没有经过配置的默认设置（PermGen 永久代大小），82M。

spark-sql，它的内部是要进行很复杂的 SQL 的语义解析、语法树的转换等等，特别复杂。在这种复杂的情况下，如果说你的 sql 本身特别复杂的话，很可能会比较导致性能的消耗，内存的消耗。可能对 PermGen 永久代的占用会比较大。

所以，此时，如果对永久代的占用需求，超过了 82M 的话，但是呢又在 128M 以内，就会出现如上所述的问题，yarn-client 模式下，默认是 128M，这个还能运行，如果在 yarn-cluster 模式下，默认是 82M，就有问题了。会报出 PermGen Out of Memory error log。

### **解决方案**

既然是 JVM 的 PermGen 永久代内存溢出，那么就是内存不够用。我们就给 yarn-cluster 模式下的 driver 的 PermGen 多设置一些。

spark-submit 脚本中，加入以下配置即可：

```javascript
--conf spark.driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"
```

这个就设置了 driver 永久代的大小，默认是 128M，最大是 256M。这样的话，就可以基本保证你的 spark 作业不会出现上述的 yarn-cluster 模式导致的永久代内存溢出的问题。

spark sql 中，写 sql，要注意一个问题：

如果 sql 有大量的 or 语句。比如 where keywords=''or keywords='' or keywords=''

当达到 or 语句，有成百上千的时候，此时可能就会出现一个 driver 端的 jvm stack overflow，JVM 栈内存溢出的问题。

JVM 栈内存溢出，基本上就是由于调用的方法层级过多，因为产生了大量的，非常深的，超出了 JVM 栈深度限制的递归方法。我们的猜测，spark sql 有大量 or 语句的时候，spark sql 内部源码中，在解析 sql，比如转换成语法树，或者进行执行计划的生成的时候，对 or 的处理是递归。or 特别多的话，就会发生大量的递归。

JVM Stack Memory Overflow，栈内存溢出。

这种时候，建议不要搞那么复杂的 spark sql 语句。采用替代方案：**将一条 \*\***sql\***\* 语句，拆解成多条 \*\***sql\***\* 语句来执行**。每条 sql 语句，就只有 100 个 or 子句以内。一条一条 SQL 语句来执行。根据生产环境经验的测试，一条 sql 语句，100 个 or 子句以内，是还可以的。通常情况下，不会报那个栈内存溢出。

**8**

\***\* 错误的持久化方式以及 checkpoint 的使用**
\---------------------------\*\*

### **使用持久化方式**

错误的持久化使用方式：

usersRDD，想要对这个 RDD 做一个 cache，希望能够在后面多次使用这个 RDD 的时候，不用反复重新计算 RDD。可以直接使用通过各个节点上的 executor 的 BlockManager 管理的内存 / 磁盘上的数据，避免重新反复计算 RDD。

usersRDD.cache()

usersRDD.count()

usersRDD.take()

上面这种方式，不要说会不会生效了，实际上是会报错的。会报什么错误呢？会报一大堆 file not found 的错误。

正确的持久化使用方式：

usersRDD

usersRDD = usersRDD.cache() // Java

val cachedUsersRDD = usersRDD.cache() // Scala

之后再去使用 usersRDD，或者 cachedUsersRDD 就可以了。

### **checkpoint 的使用**

对于持久化，大多数时候都是会正常工作的。但有些时候会出现意外。

比如说，缓存在内存中的数据，可能莫名其妙就丢失掉了。

或者说，存储在磁盘文件中的数据，莫名其妙就没了，文件被误删了。

出现上述情况的时候，如果要对这个 RDD 执行某些操作，可能会发现 RDD 的某个 partition 找不到了。

下来 task 就会对消失的 partition 重新计算，计算完以后再缓存和使用。

有些时候，计算某个 RDD，可能是极其耗时的。可能 RDD 之前有大量的父 RDD。那么如果你要重新计算一个 partition，可能要重新计算之前所有的父 RDD 对应的 partition。

这种情况下，就可以选择对这个 RDD 进行 checkpoint，以防万一。进行 checkpoint，就是说，会将 RDD 的数据，持久化一份到容错的文件系统上（比如 hdfs）。

在对这个 RDD 进行计算的时候，如果发现它的缓存数据不见了。优先就是先找一下有没有 checkpoint 数据（到 hdfs 上面去找）。如果有的话，就使用 checkpoint 数据了。不至于去重新计算。

checkpoint，其实就是可以作为是 cache 的一个备胎。如果 cache 失效了，checkpoint 就可以上来使用了。

checkpoint 有利有弊，利在于，提高了 spark 作业的可靠性，一旦发生问题，还是很可靠的，不用重新计算大量的 rdd。但是弊在于，进行 checkpoint 操作的时候，也就是将 rdd 数据写入 hdfs 中的时候，还是会消耗性能的。

checkpoint，用性能换可靠性。

checkpoint 原理：

1、在代码中，用 SparkContext，设置一个 checkpoint 目录，可以是一个容错文件系统的目录，比如 hdfs。

2、在代码中，对需要进行 checkpoint 的 rdd，执行 RDD.checkpoint()。

3、RDDCheckpointData（spark 内部的 API），接管你的 RDD，会标记为 marked for checkpoint，准备进行 checkpoint。

4、你的 job 运行完之后，会调用一个 finalRDD.doCheckpoint() 方法，会顺着 rdd lineage，回溯扫描，发现有标记为待 checkpoint 的 rdd，就会进行二次标记，inProgressCheckpoint，正在接受 checkpoint 操作。

5、job 执行完之后，就会启动一个内部的新 job，去将标记为 inProgressCheckpoint 的 rdd 的数据，都写入 hdfs 文件中。（备注，如果 rdd 之前 cache 过，会直接从缓存中获取数据，写入 hdfs 中。如果没有 cache 过，那么就会重新计算一遍这个 rdd，再 checkpoint）。

6、将 checkpoint 过的 rdd 之前的依赖 rdd，改成一个 CheckpointRDD\*，强制改变你的 rdd 的 lineage。后面如果 rdd 的 cache 数据获取失败，直接会通过它的上游 CheckpointRDD，去容错的文件系统，比如 hdfs，中，获取 checkpoint 的数据。

checkpoint 的使用：

1、sc.checkpointFile("hdfs://")，设置 checkpoint 目录

2、对 RDD 执行 checkpoint 操作

**6**

**数据倾斜解决方案**

数据倾斜的解决，跟之前讲解的性能调优，有一点异曲同工之妙。

性能调优中最有效最直接最简单的方式就是加资源加并行度，并注意 RDD 架构（复用同一个 RDD，加上 cache 缓存）。相对于前面，shuffle、jvm 等是次要的。

**1**

\***\* 原理以及现象分析**
\------------\*\*

### **数据倾斜怎么出现的**

在执行 shuffle 操作的时候，是按照 key，来进行 values 的数据的输出、拉取和聚合的。

同一个 key 的 values，一定是分配到一个 reduce task 进行处理的。

多个 key 对应的 values，比如一共是 90 万。可能某个 key 对应了 88 万数据，被分配到一个 task 上去面去执行。

另外两个 task，可能各分配到了 1 万数据，可能是数百个 key，对应的 1 万条数据。

这样就会出现数据倾斜问题。

想象一下，出现数据倾斜以后的运行的情况。很糟糕！

其中两个 task，各分配到了 1 万数据，可能同时在 10 分钟内都运行完了。另外一个 task 有 88 万条，88 \* 10 =  880 分钟 = 14.5 个小时。

大家看，本来另外两个 task 很快就运行完毕了（10 分钟），但是由于一个拖后腿的家伙，第三个 task，要 14.5 个小时才能运行完，就导致整个 spark 作业，也得 14.5 个小时才能运行完。

数据倾斜，一旦出现，是不是性能杀手？！

### **发生数据倾斜以后的现象**

Spark 数据倾斜，有两种表现：

1、你的大部分的 task，都执行的特别特别快，（你要用 client 模式，standalone client，yarn client，本地机器一执行 spark-submit 脚本，就会开始打印 log），task175 finished，剩下几个 task，执行的特别特别慢，前面的 task，一般 1s 可以执行完 5 个，最后发现 1000 个 task，998，999 task，要执行 1 个小时，2 个小时才能执行完一个 task。

出现以上 loginfo，就表明出现数据倾斜了。

这样还算好的，因为虽然老牛拉破车一样非常慢，但是至少还能跑。

2、另一种情况是，运行的时候，其他 task 都执行完了，也没什么特别的问题，但是有的 task，就是会突然间报了一个 OOM，JVM Out Of Memory，内存溢出了，task failed，task lost，resubmitting task。反复执行几次都到了某个 task 就是跑不通，最后就挂掉。

某个 task 就直接 OOM，那么基本上也是因为数据倾斜了，task 分配的数量实在是太大了！所以内存放不下，然后你的 task 每处理一条数据，还要创建大量的对象，内存爆掉了。

这样也表明出现数据倾斜了。

这种就不太好了，因为你的程序如果不去解决数据倾斜的问题，压根儿就跑不出来。

作业都跑不完，还谈什么性能调优这些东西？！

### **定位数据倾斜出现的原因与出现问题的位置**

根据 log 去定位

出现数据倾斜的原因，基本只可能是因为发生了 shuffle 操作，在 shuffle 的过程中，出现了数据倾斜的问题。因为某个或者某些 key 对应的数据，远远的高于其他的 key。

1、你在自己的程序里面找找，哪些地方用了会产生 shuffle 的算子，groupByKey、countByKey、reduceByKey、join

2、看 log

log 一般会报是在你的哪一行代码，导致了 OOM 异常。或者看 log，看看是执行到了第几个 stage。spark 代码，是怎么划分成一个一个的 stage 的。哪一个 stage 生成的 task 特别慢，就能够自己用肉眼去对你的 spark 代码进行 stage 的划分，就能够通过 stage 定位到你的代码，到底哪里发生了数据倾斜。

**2**

\***\* 聚合源数据以及过滤导致倾斜的 key**
\---------------------\*\*

数据倾斜解决方案，第一个方案和第二个方案，一起来讲。这两个方案是最直接、最有效、最简单的解决数据倾斜问题的方案。

第一个方案：聚合源数据。

第二个方案：过滤导致倾斜的 key。

后面的五个方案，尤其是最后 4 个方案，都是那种特别狂拽炫酷吊炸天的方案。但没有第一二个方案简单直接。如果碰到了数据倾斜的问题。上来就先考虑第一个和第二个方案看能不能做，如果能做的话，后面的 5 个方案，都不用去搞了。

有效、简单、直接才是最好的，彻底根除了数据倾斜的问题。

### **方案一：聚合源数据**

一些聚合的操作，比如 groupByKey、reduceByKey，groupByKey 说白了就是拿到每个 key 对应的 values。reduceByKey 说白了就是对每个 key 对应的 values 执行一定的计算。

这些操作，比如 groupByKey 和 reduceByKey，包括之前说的 join。都是在 spark 作业中执行的。

spark 作业的数据来源，通常是哪里呢？90% 的情况下，数据来源都是 hive 表（hdfs，大数据分布式存储系统）。hdfs 上存储的大数据。hive 表中的数据通常是怎么出来的呢？有了 spark 以后，hive 比较适合做什么事情？hive 就是适合做离线的，晚上凌晨跑的，ETL（extract transform load，数据的采集、清洗、导入），hive sql，去做这些事情，从而去形成一个完整的 hive 中的数据仓库。说白了，数据仓库，就是一堆表。

spark 作业的源表，hive 表，通常情况下来说，也是通过某些 hive etl 生成的。hive etl 可能是晚上凌晨在那儿跑。今天跑昨天的数据。

数据倾斜，某个 key 对应的 80 万数据，某些 key 对应几百条，某些 key 对应几十条。现在咱们直接在生成 hive 表的 hive etl 中对数据进行聚合。比如按 key 来分组，将 key 对应的所有的 values 全部用一种特殊的格式拼接到一个字符串里面去，比如 “key=sessionid, value: action_seq=1|user_id=1|search_keyword = 火锅 | category_id=001;action_seq=2|user_id=1|search_keyword = 涮肉 | category_id=001”。

对 key 进行 group，在 spark 中，拿到 key=sessionid，values<Iterable>。hive etl 中，直接对 key 进行了聚合。那么也就意味着，每个 key 就只对应一条数据。在 spark 中，就不需要再去执行 groupByKey+map 这种操作了。直接对每个 key 对应的 values 字符串进行 map 操作，进行你需要的操作即可。

spark 中，可能对这个操作，就不需要执行 shffule 操作了，也就根本不可能导致数据倾斜。

或者是对每个 key 在 hive etl 中进行聚合，对所有 values 聚合一下，不一定是拼接起来，可能是直接进行计算。reduceByKey 计算函数应用在 hive etl 中，从而得到每个 key 的 values。

聚合源数据方案第二种做法是，你可能没有办法对每个 key 聚合出来一条数据。那么也可以做一个妥协，对每个 key 对应的数据，10 万条。有好几个粒度，比如 10 万条里面包含了几个城市、几天、几个地区的数据，现在放粗粒度。直接就按照城市粒度，做一下聚合，几个城市，几天、几个地区粒度的数据，都给聚合起来。比如说

city_id date area_id

select ... from ... group by city_id

尽量去聚合，减少每个 key 对应的数量，也许聚合到比较粗的粒度之后，原先有 10 万数据量的 key，现在只有 1 万数据量。减轻数据倾斜的现象和问题。

### **方案二：过滤导致倾斜的 key**

如果你能够接受某些数据在 spark 作业中直接就摒弃掉不使用。比如说，总共有 100 万个 key。只有 2 个 key 是数据量达到 10 万的。其他所有的 key，对应的数量都是几十万。

这个时候，你自己可以去取舍，如果业务和需求可以理解和接受的话，在你从 hive 表查询源数据的时候，直接在 sql 中用 where 条件，过滤掉某几个 key。

那么这几个原先有大量数据，会导致数据倾斜的 key，被过滤掉之后，那么在你的 spark 作业中，自然就不会发生数据倾斜了。

**3**

\***\* 提高 shuffle 操作 reduce 并行度**
\------------------------\*\*

### **问题描述**

第一个和第二个方案，都不适合做，然后再考虑这个方案。

将 reduce task 的数量变多，就可以让每个 reduce task 分配到更少的数据量。这样的话也许就可以缓解甚至是基本解决掉数据倾斜的问题。

### **提升 \*\***shuffle reduce\***\* 端并行度的操作方法**

很简单，主要给我们所有的 shuffle 算子，比如 groupByKey、countByKey、reduceByKey。在调用的时候，传入进去一个参数。那个数字，就代表了那个 shuffle 操作的 reduce 端的并行度。那么在进行 shuffle 操作的时候，就会对应着创建指定数量的 reduce task。

这样的话，就可以让每个 reduce task 分配到更少的数据。基本可以缓解数据倾斜的问题。

比如说，原本某个 task 分配数据特别多，直接 OOM，内存溢出了，程序没法运行，直接挂掉。按照 log，找到发生数据倾斜的 shuffle 操作，给它传入一个并行度数字，这样的话，原先那个 task 分配到的数据，肯定会变少。就至少可以避免 OOM 的情况，程序至少是可以跑的。

### **提升 \*\***shuffle reduce\***\* 并行度的缺陷**

治标不治本的意思，因为它没有从根本上改变数据倾斜的本质和问题。不像第一个和第二个方案（直接避免了数据倾斜的发生）。原理没有改变，只是说，尽可能地去缓解和减轻 shuffle reduce task 的数据压力，以及数据倾斜的问题。

实际生产环境中的经验：

1、如果最理想的情况下，提升并行度以后，减轻了数据倾斜的问题，或者甚至可以让数据倾斜的现象忽略不计，那么就最好。就不用做其他的数据倾斜解决方案了。

2、不太理想的情况下，比如之前某个 task 运行特别慢，要 5 个小时，现在稍微快了一点，变成了 4 个小时。或者是原先运行到某个 task，直接 OOM，现在至少不会 OOM 了，但是那个 task 运行特别慢，要 5 个小时才能跑完。

那么，如果出现第二种情况的话，各位，就立即放弃第三种方案，开始去尝试和选择后面的四种方案。

**4**

\***\* 使用随机 key 实现双重聚合**
\-----------------\*\*

### **使用场景**

groupByKey、reduceByKey 比较适合使用这种方式。join 咱们通常不会这样来做，后面会讲三种针对不同的 join 造成的数据倾斜的问题的解决方案。

### **解决方案**

第一轮聚合的时候，对 key 进行打散，将原先一样的 key，变成不一样的 key，相当于是将每个 key 分为多组。

先针对多个组，进行 key 的局部聚合。接着，再去除掉每个 key 的前缀，然后对所有的 key 进行全局的聚合。

对 groupByKey、reduceByKey 造成的数据倾斜，有比较好的效果。

如果说，之前的第一、第二、第三种方案，都没法解决数据倾斜的问题，那么就只能依靠这一种方式了。

**5**

\***\* 将 reduce join 转换为 map join**
\---------------------------\*\*

### **使用方式**

普通的 join，那么肯定是要走 shuffle。既然是走 shuffle，那么普通的 join 就肯定是走的是 reduce join。那怎么将 reduce join 转换为 mapjoin 呢？先将所有相同的 key，对应的 value 汇聚到一个 task 中，然后再进行 join。

### **使用场景**

这种方式适合在什么样的情况下来使用？

如果两个 RDD 要进行 join，其中一个 RDD 是比较小的。比如一个 RDD 是 100 万数据，一个 RDD 是 1 万数据。（一个 RDD 是 1 亿数据，一个 RDD 是 100 万数据）。

其中一个 RDD 必须是比较小的，broadcast 出去那个小 RDD 的数据以后，就会在每个 executor 的 block manager 中都保存一份。要确保你的内存足够存放那个小 RDD 中的数据。

这种方式下，根本不会发生 shuffle 操作，肯定也不会发生数据倾斜。从根本上杜绝了 join 操作可能导致的数据倾斜的问题。

对于 join 中有数据倾斜的情况，大家尽量第一时间先考虑这种方式，效果非常好。

不适合的情况

两个 RDD 都比较大，那么这个时候，你去将其中一个 RDD 做成 broadcast，就很笨拙了。很可能导致内存不足。最终导致内存溢出，程序挂掉。

而且其中某些 key（或者是某个 key），还发生了数据倾斜。此时可以采用最后两种方式。

对于 join 这种操作，不光是考虑数据倾斜的问题。即使是没有数据倾斜问题，也完全可以优先考虑，用我们讲的这种高级的 reduce join 转 map join 的技术，不要用普通的 join，去通过 shuffle，进行数据的 join。完全可以通过简单的 map，使用 map join 的方式，牺牲一点内存资源。在可行的情况下，优先这么使用。

不走 shuffle，直接走 map，是不是性能也会高很多？这是肯定的。

**6**

\***\*sample 采样倾斜 key 单独进行 join**
\-------------------------\*\*

### **方案实现思路**

将发生数据倾斜的 key，单独拉出来，放到一个 RDD 中去。就用这个原本会倾斜的 key RDD 跟其他 RDD 单独去 join 一下，这个时候 key 对应的数据可能就会分散到多个 task 中去进行 join 操作。

就不至于说是，这个 key 跟之前其他的 key 混合在一个 RDD 中时，肯定是会导致一个 key 对应的所有数据都到一个 task 中去，就会导致数据倾斜。

### **使用场景**

这种方案什么时候适合使用？

优先对于 join，肯定是希望能够采用上一个方案，即 reduce join 转换 map join。两个 RDD 数据都比较大，那么就不要那么搞了。

针对你的 RDD 的数据，你可以自己把它转换成一个中间表，或者是直接用 countByKey() 的方式，你可以看一下这个 RDD 各个 key 对应的数据量。此时如果你发现整个 RDD 就一个，或者少数几个 key 对应的数据量特别多。尽量建议，比如就是一个 key 对应的数据量特别多。

此时可以采用这种方案，单拉出来那个最多的 key，单独进行 join，尽可能地将 key 分散到各个 task 上去进行 join 操作。

什么时候不适用呢？

如果一个 RDD 中，导致数据倾斜的 key 特别多。那么此时，最好还是不要这样了。还是使用我们最后一个方案，终极的 join 数据倾斜的解决方案。

就是说，咱们单拉出来了一个或者少数几个可能会产生数据倾斜的 key，然后还可以进行更加优化的一个操作。

对于那个 key，从另外一个要 join 的表中，也过滤出来一份数据，比如可能就只有一条数据。userid2infoRDD，一个 userid key，就对应一条数据。

然后呢，采取对那个只有一条数据的 RDD，进行 flatMap 操作，打上 100 个随机数，作为前缀，返回 100 条数据。

单独拉出来的可能产生数据倾斜的 RDD，给每一条数据，都打上一个 100 以内的随机数，作为前缀。

再去进行 join，是不是性能就更好了。肯定可以将数据进行打散，去进行 join。join 完以后，可以执行 map 操作，去将之前打上的随机数给去掉，然后再和另外一个普通 RDD join 以后的结果进行 union 操作。

**7**

\***\* 使用随机数以及扩容表进行 join**
\--------------------\*\*

### **使用场景及步骤**

当采用随机数和扩容表进行 join 解决数据倾斜的时候，就代表着，你的之前的数据倾斜的解决方案，都没法使用。

这个方案是没办法彻底解决数据倾斜的，更多的，是一种对数据倾斜的缓解。

步骤：

1、选择一个 RDD，要用 flatMap，进行扩容，将每条数据，映射为多条数据，每个映射出来的数据，都带了一个 n 以内的随机数，通常来说会选择 10。

2、将另外一个 RDD，做普通的 map 映射操作，每条数据都打上一个 10 以内的随机数。

3、最后将两个处理后的 RDD 进行 join 操作。

### **局限性**

1、因为你的两个 RDD 都很大，所以你没有办法去将某一个 RDD 扩的特别大，一般咱们就是 10 倍。

2、如果就是 10 倍的话，那么数据倾斜问题的确是只能说是缓解和减轻，不能说彻底解决。

sample 采样倾斜 key 并单独进行 join

将 key，从另外一个 RDD 中过滤出的数据，可能只有一条或者几条，此时，咱们可以任意进行扩容，扩成 1000 倍。

将从第一个 RDD 中拆分出来的那个倾斜 key RDD，打上 1000 以内的一个随机数。

这种情况下，还可以配合上，提升 shuffle reduce 并行度，join(rdd, 1000)。通常情况下，效果还是非常不错的。

打散成 100 份，甚至 1000 份，2000 份，去进行 join，那么就肯定没有数据倾斜的问题了吧。

**7**

**实时计算程序性能调优**

1、并行化数据接收：处理多个 topic 的数据时比较有效

```javascript
int numStreams = 5;
List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<JavaPairDStream<String, String>>(numStreams);
for (int i = 0; i < numStreams; i++) {
  kafkaStreams.add(KafkaUtils.createStream(...));
}
JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
unifiedStream.print();
```

2、spark.streaming.blockInterval：增加 block 数量，增加每个 batch rdd 的 partition 数量，增加处理并行度

receiver 从数据源源源不断地获取到数据；首先是会按照 block interval，将指定时间间隔的数据，收集为一个 block；默认时间是 200ms，官方推荐不要小于 50ms；接着呢，会将指定 batch interval 时间间隔内的 block，合并为一个 batch；创建为一个 rdd，然后启动一个 job，去处理这个 batch rdd 中的数据

batch rdd，它的 partition 数量是多少呢？一个 batch 有多少个 block，就有多少个 partition；就意味着并行度是多少；就意味着每个 batch rdd 有多少个 task 会并行计算和处理。

当然是希望可以比默认的 task 数量和并行度再多一些了；可以手动调节 block interval；减少 block interval；每个 batch 可以包含更多的 block；有更多的 partition；也就有更多的 task 并行处理每个 batch rdd。

定死了，初始的 rdd 过来，直接就是固定的 partition 数量了

3、inputStream.repartition(<number of partitions>)：重分区，增加每个 batch rdd 的 partition 数量

有些时候，希望对某些 dstream 中的 rdd 进行定制化的分区

对 dstream 中的 rdd 进行重分区，去重分区成指定数量的分区，这样也可以提高指定 dstream 的 rdd 的计算并行度

4、调节并行度

```css
spark.default.parallelism
reduceByKey(numPartitions)
```

5、使用 Kryo 序列化机制：

spark streaming，也是有不少序列化的场景的

提高序列化 task 发送到 executor 上执行的性能，如果 task 很多的时候，task 序列化和反序列化的性能开销也比较可观

默认输入数据的存储级别是 StorageLevel.MEMORY_AND_DISK_SER_2，receiver 接收到数据，默认就会进行持久化操作；首先序列化数据，存储到内存中；如果内存资源不够大，那么就写入磁盘；而且，还会写一份冗余副本到其他 executor 的 block manager 中，进行数据冗余。

6、batch interval：每个的处理时间必须小于 batch interval

实际上你的 spark streaming 跑起来以后，其实都是可以在 spark ui 上观察它的运行情况的；可以看到 batch 的处理时间；

如果发现 batch 的处理时间大于 batch interval，就必须调节 batch interval

尽量不要让 batch 处理时间大于 batch interval

比如你的 batch 每隔 5 秒生成一次；你的 batch 处理时间要达到 6 秒；就会出现，batch 在你的内存中日积月累，一直囤积着，没法及时计算掉，释放内存空间；而且对内存空间的占用越来越大，那么此时会导致内存空间快速消耗

如果发现 batch 处理时间比 batch interval 要大，就尽量将 batch interval 调节大一些

====================================

此文投稿来自：多易教育 (地表最强大数据教学机构)

说到多易教育，除了之前大力推荐的离线数仓项目外，目前涛哥最新研发的项目 ： flink 动态规则实时智能营销系统

本动态规则实时运营系统，核心技术为 flink+kafka+clickhouse+redis+hbase，可以是一个相对独立的大数据系统

也可以整合用户画像系统，用户行为分析系统，实时数仓等；

详情可扫码查看：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcGyNcDm7GntqV7fh5IIF4rp0XhVNUFAOKeLTRjdnNIGnwjMLI7icEwQ7IOWreoKazAia3vmrohrn6w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_gif/dXCnejTRMLcGyNcDm7GntqV7fh5IIF4rOSmRy8ZXM9zqiaYy3hoAvFAdPahNibnZ5Bvwg5IrnQiaJmR1y6m1olxRw/640?wx_fmt=gif)

● [2021 大数据面试真题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247487775&idx=1&sn=32e90ad3274db5ff404060ac35379596&chksm=ea68fbd3dd1f72c583c92d24ba424b680eb6e8290c2d4af60356c12a3dc62a13f88e07ea3971&scene=21#wechat_redirect)

● [数据仓库体系](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485691&idx=1&sn=d6cb1353031e07e4b02cd903d8b57911&chksm=ea68e237dd1f6b210f65f25ef42dabf4453d3bfa36fe8f33b149c0ff5329f77b9b792eef7882&scene=21#wechat_redirect)

● [数据质量管理](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485039&idx=1&sn=140c3bc720da51765292fe3f5082fe38&chksm=ea68eca3dd1f65b5aef4d6f7ab0c33d3d3033bcc0eead1650be079687e0b4e898562bfe4d25b&scene=21#wechat_redirect)

● [元数据管理](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485186&idx=1&sn=85fbe5703c56aa2dcfd2980fccbab4f6&chksm=ea68edcedd1f64d8e2d8c3da6b456fcaa4b105f2216a2bddb2393a7380498166225de5e855b4&scene=21#wechat_redirect)

● [模型优化](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247487977&idx=1&sn=ba5f0b10595e485aef9a8722538c507f&chksm=ea68fb25dd1f72336f6e4588071544acf94aae073df3b62f02170281e267a14e3be7a47e6f3c&scene=21#wechat_redirect)

● [Hadoop](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247486497&idx=1&sn=4498d46a2d7e3a1b3eecd5bda171d3f5&chksm=ea68e6eddd1f6ffb7cb6b19263178fc0e5d1f564dd11eaf6f932c119904d65cb99b2dc613a90&scene=21#wechat_redirect)

● [Hive](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485965&idx=1&sn=2fc0899c82dffe7721c55ad717cd2678&chksm=ea68e0c1dd1f69d70689408d9e9f66f00a638d851b9ffb50484d697ecf05f9b0f2989a10b80d&scene=21#wechat_redirect)

● [Hive 调优](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485048&idx=1&sn=5fc1219f4947bea9743cd938cec510c7&chksm=ea68ecb4dd1f65a2df364d79272e0e472a394c5b13b5d55d848c89d9498ccb7ea78a933fbdea&scene=21#wechat_redirect)

● [Flink](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247486303&idx=1&sn=c1f58c14372ae9ba22dc9c464ae04143&chksm=ea68e193dd1f6885c3fa1903114ea62aa9f6ee4ee78384c667306fd815c37de9cf1fb300212a&scene=21#wechat_redirect)

● [Clickhouse](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247487158&idx=1&sn=01bec273b27ea954bc53a1b3188b2289&chksm=ea68e47add1f6d6cb29ca44b2edc5cb5a2e26ff8064a3360f7b78dfd70a791cbfdb50acf4dcd&scene=21#wechat_redirect)

● [Hbase](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485858&idx=1&sn=75a3a3dd71a612b5ac4412c423554787&chksm=ea68e36edd1f6a785b739bd2ec2f0cc94f5d3679ac40f631fbd61abad96e9652bea14cc705f5&scene=21#wechat_redirect)

● [Kafka](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485999&idx=1&sn=b7e6e0aabb77504e920ff4342f44d26f&chksm=ea68e0e3dd1f69f56ac2c97839cf8d043d9b503b5461fe006fa14a9974375a386cd66000aadd&scene=21#wechat_redirect)

● [DolphinScheduler 海豚调度](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247486107&idx=1&sn=b1fb84db241e15defd91eb3a6a3699c9&chksm=ea68e057dd1f69411f72e646bdffd28d0ac1682ede5d613c698d5fff18f1b861de39627d86ea&scene=21#wechat_redirect)  

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLeibmoxSpRXUevnVIz4RyV0AUrglXsIgLOkFMNuffPAh5lZbwOnSoRHbZqT12VyKibm0EmnTicXBSRTg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/viZJ6yoikDOuaFUKaIyERA](https://mp.weixin.qq.com/s/viZJ6yoikDOuaFUKaIyERA)
