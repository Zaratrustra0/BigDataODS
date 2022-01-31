# Spark单value，key-value类型21个算子(图解与源码)
点击上方关注**大数据左右手**回复关键字【资料】

即可获取此文档，方便查阅

## Spark 单 value 算子

### 1. map 算子（改变结构就用 map）

#### 先看 map 函数

     /**   * Return a new RDD by applying a function to all elements of this RDD.   */  def map[U: ClassTag](f: T => U): RDD[U] = withScope {    val cleanF = sc.clean(f)    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))  }

#### 功能说明

参数 f 是一个函数，它可以接收一个参数。当某个 RDD 执行 map 方法时，会遍历该 RDD 中的每一个数据项，并依次应用 f 函数，从而产生一个新的 RDD。即，这个新 RDD 中的每一个元素都是原来 RDD 中每一个元素依次应用 f 函数而得到的。

#### 代码演示

     // 创建SparkConf val conf: SparkConf = new SparkConf() // 创建SparkContext，该对象是提交Spark App的入口  val sc: SparkContext = new SparkContext(conf)  // 参数（数据源，分区数(可选) ）    val rdd: RDD[Int] = sc.makeRDD(1 to 4,2)    // map操作 元素乘以2    val mapRdd: RDD[Int] = rdd.map(_*2)     mapRdd.collect().foreach(println)结果：2 4 6 8

#### 关于分区

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1WMfmNHJcicE0llAibIB4VYwe6GY4FEO0K7Xq7zeSSqLGuRtBt7PoSicvA/640?wx_fmt=png)

图片中的说明：先把一个数据拿过来以后进行 \*2 操作 例如拿 1 过来后 \*2 = 2 后，1 这个数据就离开这块区域 然后进行第二个数据的处理…

注意：map 的分区数和 RDD 的分区数一致（看下面源码）。

    def map[U: ClassTag](f: T => U): RDD[U] = withScope {    val cleanF = sc.clean(f)    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))  }往下走 override def getPartitions: Array[Partition] = firstParent[T].partitions再往下走firstParent/** Returns the first parent RDD */  protected[spark] def firstParent[U: ClassTag]: RDD[U] = {    dependencies.head.rdd.asInstanceOf[RDD[U]]  }主要的是：firstParent[T].partitions 这里

### 2. mapPartitions() 以分区为单位执行 Map

#### 先看 mapPartitions 函数

    def mapPartitions[U: ClassTag](      f: Iterator[T] => Iterator[U],      preservesPartitioning: Boolean = false): RDD[U] = withScope {    val cleanedF = sc.clean(f)    new MapPartitionsRDD(      this,      (context: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(iter),      preservesPartitioning)  }

#### 功能说明

f: Iterator\[T] => Iterator\[U]：f 函数把每个分区的数据分别放入到迭代器中（批处理）。

preservesPartitioning: Boolean = false ：是否保留 RDD 的分区信息。

功能：一次处理一个分区数据。

#### 代码演示

     // 前面代码省略，直接从数据源开始 val rdd: RDD[Int] = sc.makeRDD(1 to 4，2)    val mapRdd = rdd.mapPartitions(_.map(_*2))    mapRdd.collect().foreach(println)结果：2468

#### 关于分区

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1VZh6ibWloJqBfFdWia7fVPRV9CTOVS85zC6X9CyWbBqhFyupJdjqmiarw/640?wx_fmt=png)

分区说明 每一个分区的数据会先到内存空间，然后才进行逻辑操作，整个分区操作完之后，拿到分区的数据才会释放掉。从性能方面讲：批处理效率高 从内存方面：需要内存空间较大

### 3. mapPartitionsWithIndex() 带分区号

#### 先看 mapPartitionsWithIndex 函数

    def mapPartitionsWithIndex[U: ClassTag](  // Int表示分区编号      f: (Int, Iterator[T]) => Iterator[U],       preservesPartitioning: Boolean = false): RDD[U]

#### 功能说明

f: (Int, Iterator\[T]) => Iterator\[U]：f 函数把每个分区的数据分别放入到迭代器中（批处理）并且加上分区号

preservesPartitioning: Boolean = false ：是否保留 RDD 的分区信息

功能：比 mapPartitions 多一个整数参数表示分区号

#### 代码演示

     val rdd: RDD[Int] = sc.makeRDD(1 to 4, 2)    val mapRdd = rdd.mapPartitionsWithIndex((index, items) => {      items.map((index, _))    })    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果：(0,1)(0,2)(1,3)(1,4)

### 4. flatMap() 扁平化

#### 先看 flatMap 函数

    def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {    val cleanF = sc.clean(f)    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))  }

#### 功能说明

与 map 操作类似，将 RDD 中的每一个元素通过应用 f 函数依次转换为新的元素，并封装到 RDD 中。

区别：在 flatMap 操作中，f 函数的返回值是一个集合，并且会将每一个该集合中的元素拆分出来放到新的 RDD 中。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1Jw5VLZzDC7WNNGUR82Hic83b7cpxuwxCuMQlbufJemC2PmlW9SPxKbg/640?wx_fmt=png)

#### 代码演示

     val listRDD=sc.makeRDD(List(List(1,2),List(3,4),List(5,6),List(7)), 2)    val mapRdd: RDD[Int]= listRDD.flatMap(item=>item)    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果：1234567

### 5. glom() 分区转换数组

#### 先看 glom 函数

     def glom(): RDD[Array[T]] = withScope {    new MapPartitionsRDD[Array[T], T](this, (context, pid, iter) => Iterator(iter.toArray))  }

#### 功能说明

该操作将 RDD 中每一个分区变成一个数组，并放置在新的 RDD 中，数组中元素的类型与原分区中元素类型一致。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r15WZMLDEkoF9Nic1eYGMDy5ia6Gw0aaqdQouv3CrWNyNHSEXDtnHFMPuA/640?wx_fmt=png)

#### 代码演示（求两个分区中的最大值）

     val sc: SparkContext = new SparkContext(conf)    val rdd = sc.makeRDD(1 to 4, 2)    val mapRdd = rdd.glom().map(_.max)    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)     结果：24

### 6. groupBy() 分组

#### 先看 groupBy 函数

    def groupBy[K](f: T => K)(implicit kt: ClassTag[K]): RDD[(K, Iterable[T])] = withScope {    groupBy[K](f, defaultPartitioner(this))  }

#### 功能说明

分组，按照传入函数的返回值进行分组。将相同的 key 对应的值放入一个迭代器。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1SEjiaoicE2pz2CXzeGaaY79WUib2YPwWgVokyf4jCqMkaHhj3Tf7hesug/640?wx_fmt=png)

#### 代码演示 （按照元素模以 2 的值进行分组）

     val rdd = sc.makeRDD(1 to 4,2)    val mapRdd: RDD[(Int, Iterable[Int])] = rdd.groupBy(_ % 2)    // 打印修改后的RDD中数据    mapRdd.collect().foreach(println)结果：(0,CompactBuffer(2, 4))(1,CompactBuffer(1, 3))

### 7. sample() 采样

#### 先看 sample 函数

    def sample(      withReplacement: Boolean,      fraction: Double,      seed: Long = Utils.random.nextLong): RDD[T]

#### 功能说明

withReplacement：true 为有放回的抽样，false 为无放回的抽样。

fraction 表示：以指定的随机种子随机抽样出数量为 fraction 的数据。

seed 表示：指定随机数生成器种子。

两个算法介绍：

     抽取数据不放回（伯努利算法）   val sampleRDD: RDD[Int] = dataRDD.sample(false, 0.5)        伯努利算法：又叫0、1分布。例如扔硬币，要么正面，要么反面。        具体实现：根据种子和随机算法算出一个数和第二个参数设置几率比较，小于第二个参数要，大于不要        第一个参数：抽取的数据是否放回，false：不放回        第二个参数：抽取的机率，范围在[0,1]之间,0：全不取；1：全取；        第三个参数：随机数种子 抽取数据放回（泊松算法）   val sampleRDD1: RDD[Int] = dataRDD.sample(true, 2)   第一个参数：抽取的数据是否放回，true：放回；false：不放回        第二个参数：重复数据的几率，范围大于等于0.表示每一个元素被期望抽取到的次数        第三个参数：随机数种子

#### 代码演示

     val dataRDD: RDD[Int] = sc.makeRDD(List(1,2,3,4,5,6))    val mapRdd: RDD[Int] = dataRDD.sample(false, 0.5)    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果：356 val sc: SparkContext = new SparkContext(conf)    val dataRDD: RDD[Int] = sc.makeRDD(List(1,2,3,4,5,6))    val mapRdd: RDD[Int] = dataRDD.sample(true, 2)结果：1111133345555566

#### 8. distinct() 去重

#### 先看 distinct 函数

    def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {    map(x => (x, null)).reduceByKey((x, y) => x, numPartitions).map(_._1)  }  /**   * Return a new RDD containing the distinct elements in this RDD.   */  def distinct(): RDD[T] = withScope {    distinct(partitions.length)  }

#### 功能说明

对内部的元素去重，distinct 后会生成与原 RDD 分区个数不一致的分区数。上面的函数还可以对去重后的修改分区个数。

#### 代码演示

     val sc: SparkContext = new SparkContext(conf)    val distinctRdd: RDD[Int] = sc.makeRDD(List(1,2,1,5,2,9,6,1))    distinctRdd.distinct(2).collect().foreach(println)结果62195

#### distinct() 实现的源码

    def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {    map(x => (x, null)).reduceByKey((x, y) => x, numPartitions).map(_._1)  }也就是这个玩意： map(x => (x, null)).reduceByKey((x, y) => x, numPartitions).map(_._1)

### 9. coalesce() 合并分区

#### 先看 coalesce 函数

    def coalesce(numPartitions: Int, shuffle: Boolean = false,               partitionCoalescer: Option[PartitionCoalescer] = Option.empty)              (implicit ord: Ordering[T] = null)      : RDD[T]

#### 功能说明

功能说明：缩减分区数，用于大数据集过滤后，提高小数据集的执行效率。

默认 false 不执行 shuffle。

#### 代码演示

     val rdd: RDD[Int] = sc.makeRDD(Array(1,2,3,4),4)    val mapRdd: RDD[Int] = rdd.coalesce(2)    mapRdd.mapPartitionsWithIndex{      (index,values)=>values.map((index,_))    }.collect().foreach(println)结果(0,1)(0,2)(1,3)(1,4)

#### 无 shuffle

    设置2个分区后的结果：(0,1) (0,2) (1,3) (1,4) 设置3个分区后的结果：(0,1) (1,2) (2,3) (2,4) 设置4个或者以上(0,1) (1,2) (2,3) (3,4) 

#### 设置 shuffle

     val rdd: RDD[Int] = sc.makeRDD(Array(1,2,3,4),4)    val mapRdd: RDD[Int] = rdd.coalesce(2，true)    mapRdd.mapPartitionsWithIndex{      (index,values)=>values.map((index,_))    }.collect().foreach(println)结果(0,1)(0,2)(0,3)(0,4)设置true后开启shuffle 设置1 ,2后的结果(0,1) (0,2) (0,3) (0,4) 设置3后的结果(0,1) (1,2) (1,3) (2,4) 设置4后的结果(3,1) (3,2) (3,3) (3,4) ....

#### 源码

    for (i <- 0 until maxPartitions) {  val rangeStart = ((i.toLong * prev.partitions.length) / maxPartitions).toInt  val rangeEnd = (((i.toLong + 1) * prev.partitions.length) / maxPartitions).toInt  (rangeStart until rangeEnd).foreach{ j => groupArr(i).partitions += prev.partitions(j) }}解释说明：    maxPartitions：传进来的新分区数    prev.partitions：之前RDD的分区数 分区i    开始 = 分区号*前一个分区数 / 新的分区数    结束 =( 分区号+1)*前一个分区数 / 新的分区数

### 10. repartition() 重新分区（执行 Shuffle）

#### 先看 repartition 函数

    def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {    coalesce(numPartitions, shuffle = true)  }

#### 功能说明

该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。无论是将分区数多的 RDD 转换为分区数少的 RDD，还是将分区数少的 RDD 转换为分区数多的 RDD，repartition 操作都可以完成，因为无论如何都会经 shuffle 过程。

#### 代码演示

     val rdd: RDD[Int] = sc.makeRDD(Array(1,2,3,4,5,6,7,8),4)    val mapRdd: RDD[Int] = rdd.repartition(8)    mapRdd.mapPartitionsWithIndex{      (index,values) =>values.map((index,_))    }.collect().foreach(println)结果(6,1)(6,3)(6,5)(6,7)(7,2)(7,4)(7,6)(7,8)

#### coalesce 和 repartition 对比与区别

（1）coalesce 重新分区，可以选择是否进行 shuffle 过程。由参数 shuffle: Boolean = false/true 决定。

（2）repartition 实际上是调用的 coalesce，进行 shuffle。源码如下：def repartition(numPartitions: Int)(implicit ord: Ordering\[T] = null): RDD\[T] = withScope {coalesce(numPartitions, shuffle = true) }

（3）coalesce 一般为缩减分区，如果扩大分区，也不会增加分区总数，意义不大。

（4）repartition 扩大分区执行 shuffle，可以达到扩大分区的效果。

#### 11. sortBy() 排序

#### 先看 sortBy 函数

    def sortBy[K](      f: (T) => K,      ascending: Boolean = true,      numPartitions: Int = this.partitions.length)

#### 功能说明

该操作用于排序数据。在排序之前，可以将数据通过 f 函数进行处理，之后按照 f 函数处理的结果进行排序，默认为正序排列。排序后新产生的 RDD 的分区数与原 RDD 的分区数一致。

#### 代码演示

     val rdd: RDD[Int] = sc.makeRDD(Array(1,2,5,3,6,2))    val mapRdd = rdd.sortBy(item=>item) // 默认为true为正序，false为倒序    // 打印修改后的RDD中数据    mapRdd.collect().foreach(println)结果122356 val rdd: RDD[(Int,Int)] = sc.makeRDD(Array((5,2),(5,3),(6,2)))    val mapRdd = rdd.sortBy(item=>item) // 默认为true为正序，false为倒序    // 打印修改后的RDD中数据    mapRdd.collect().foreach(println)结果(5,2)(5,3)(6,2)

## Spark key-value 类型算子总结 (图解和源码)

### 1. partitionBy() 按照 K 重新分区

#### 先看 partitionBy 函数

     def partitionBy(partitioner: Partitioner): RDD[(K, V)] = self.withScope {    if (keyClass.isArray && partitioner.isInstanceOf[HashPartitioner]) {      throw new SparkException("HashPartitioner cannot partition array keys.")    }    if (self.partitioner == Some(partitioner)) {      self    } else {      new ShuffledRDD[K, V, V](self, partitioner)    }  }

#### 功能说明

将 RDD\[K,V]中的 K 按照指定 Partitioner 重新进行分区。

如果原有的 RDD 和新的 RDD 是一致的话就不进行分区，否则会产生 Shuffle 过程。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1NGxiaT6qW1Zc6rXviaHYCwtQbT8licDs5SptpkRI1NE7dEcxUm6Cn03RQ/640?wx_fmt=png)

#### 代码演示（按 HashPartitioner 分区）

     val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1,"aaa"),(2,"bbb"),(3,"ccc")),3)    val mapRdd: RDD[(Int, String)] = rdd.partitionBy(new org.apache.spark.HashPartitioner(2))    mapRdd.mapPartitionsWithIndex{      (index,values)=>values.map((index,_))    }.collect().foreach(println)结果(0,(2,bbb))(1,(1,aaa))(1,(3,ccc))

#### 自定义分区规则

要实现自定义分区器，需要继承 org.apache.spark.Partitioner 类，并实现下面三个方法。

（1）numPartitions: Int: 返回创建出来的分区数。

（2）getPartition(key: Any):

Int: 返回给定键的分区编号 (0 到 numPartitions-1)。

（3）equals():Java 判断相等性的标准方法。这个方法的实现非常重要，Spark 需要用这个方法来检查你的分区器对象是否和其他分区器实例相同，这样 Spark 才可以判断两个 RDD 的分区方式是否相同。

    // main方法 val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1,"aaa"),(2,"bbb"),(3,"ccc")),3)    val mapRdd: RDD[(Int, String)] = rdd.partitionBy(new MyPartition(2))    mapRdd.mapPartitionsWithIndex{      (index,values)=>values.map((index,_))    }.collect().foreach(println)// 主要代码class MyPartition(num:Int) extends Partitioner{    override def numPartitions: Int = num    override def getPartition(key: Any): Int = {      if(key.isInstanceOf[Int]){        val i: Int = key.asInstanceOf[Int]        if(i%2==0){          0        }else{          1        }      }else{        0      }    }  }结果(0,(2,bbb))(1,(1,aaa))(1,(3,ccc))

### 2. reduceByKey() 按照 K 聚合 V

#### 先看 reduceByKey 函数

     def reduceByKey(func: (V, V) => V): RDD[(K, V)] = self.withScope {    reduceByKey(defaultPartitioner(self), func)  }   def reduceByKey(func: (V, V) => V, numPartitions: Int): RDD[(K, V)]

#### 功能说明

该操作可以将 RDD\[K,V]中的元素按照相同的 K 对 V 进行聚合。其存在多种重载形式，还可以设置新 RDD 的分区数。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1YAO8tiaADXsraakeuKu142uzpTqQ3lrugzgl3MyeAgeYs0K4JHBnb2w/640?wx_fmt=png)

#### 代码演示

     val rdd = sc.makeRDD(List(("a",1),("b",5),("a",5),("b",2)))    val mapRdd: RDD[(String, Int)] = rdd.reduceByKey((v1,v2)=>v1+v2)    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果(a,6)(b,7)

### 3. groupByKey() 按照 K 重新分组

#### 先看 groupByKey 函数

    def groupByKey(): RDD[(K, Iterable[V])]def groupByKey(numPartitions: Int): RDD[(K, Iterable[V])] = self.withScope {    groupByKey(new HashPartitioner(numPartitions))  }

#### 功能说明

groupByKey 对每个 key 进行操作，但只生成一个 seq，并不进行聚合。该操作可以指定分区器或者分区数（默认使用 HashPartitioner）。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1u9qMNc0brqdSrl4SiaIGgwiaD3CLVHfDEkbaL5zWk2f6U79ibDAPlW8ag/640?wx_fmt=png)

#### 代码演示

      val rdd = sc.makeRDD(List(("a",1),("b",5),("a",5),("b",2)))    val mapRdd: RDD[(String, Iterable[Int])] = rdd.groupByKey(2)    // 打印修改后的RDD中数据    mapRdd.mapPartitionsWithIndex{      (index,values)=>values.map((index,_))    }.collect().foreach(println)结果(0,(b,CompactBuffer(5, 2)))(1,(a,CompactBuffer(1, 5)))

### 4. aggregateByKey() 按照 K 处理分区内和分区间逻辑

#### 先看 aggregateByKey 函数

    def aggregateByKey[U: ClassTag](zeroValue: U)(seqOp: (U, V) => U,      combOp: (U, U) => U): RDD[(K, U)] = self.withScope {    aggregateByKey(zeroValue, defaultPartitioner(self))(seqOp, combOp)  }

#### 功能说明

1）zeroValue（初始值）：给每一个分区中的每一种 key 一个初始值。

这个初始值的理解：

如何求最大值，所有的值为正数，设置为 0，

如果有负值，设置为 负无穷，

这个初始值就是与第一个值进行比较，保证第一次对比下去。

（2）seqOp（分区内）：函数用于在每一个分区中用初始值逐步迭代 value。

（3）combOp（分区间）：函数用于合并每个分区中的结果。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1pI5WqCiaYX8SywZVLpiaobibo3IEiclEPemycOWYOuCXia9bRDzCFXG0uKw/640?wx_fmt=png)

#### 代码演示

     val sc: SparkContext = new SparkContext(conf)    val rdd: RDD[(String, Int)] = sc.makeRDD(List(("a", 3), ("a", 2), ("c", 4), ("b", 3), ("c", 6), ("c", 8)), 2)    val mapRdd: RDD[(String, Int)] = rdd.aggregateByKey(0)(math.max(_,_),_+_)    结果(b,3)(a,3)(c,12)

### 5. foldByKey() 分区内和分区间相同的 aggregateByKey()

#### 先看 foldByKey 函数

    def aggregateByKey[U: ClassTag](zeroValue: U)(seqOp: (U, V) => U,      combOp: (U, U) => U): RDD[(K, U)] = self.withScope {    aggregateByKey(zeroValue, defaultPartitioner(self))(seqOp, combOp)  }

#### 功能说明

参数 zeroValue：是一个初始化值，它可以是任意类型。

参数 func：是一个函数，两个输入参数相同。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1j0JibfBNekjvdsRopibpT6Ocwc5icMnAydNnQHVaoCQicbCNW203fduecw/640?wx_fmt=png)

#### 代码演示

      val list = List(("a",1),("a",1),("a",1),("b",1),("b",1),("b",1),("b",1),("a",1))    val rdd = sc.makeRDD(list,2)    rdd.foldByKey(0)(_+_).collect().foreach(println)结果(b,4)(a,4)

### 6. combineByKey() 转换结构后分区内和分区间操作

#### 先看 combineByKey 函数

    def combineByKey[C](      createCombiner: V => C,      mergeValue: (C, V) => C,      mergeCombiners: (C, C) => C): RDD[(K, C)] = self.withScope {    combineByKeyWithClassTag(createCombiner, mergeValue, mergeCombiners)(null)  }

#### 功能说明

（1）createCombiner（转换数据的结构）: combineByKey()

会遍历分区中的所有元素，因此每个元素的键要么还没有遇到过，要么就和之前的某个元素的键相同。如果这是一个新的元素，combineByKey() 会使用一个叫作 createCombiner() 的函数来创建那个键对应的累加器的初始值。

（2）mergeValue（分区内）:

如果这是一个在处理当前分区之前已经遇到的键，它会使用 mergeValue() 方法将该键的累加器对应的当前值与这个新的值进行合并。

（3）mergeCombiners（分区间）:

由于每个分区都是独立处理的，因此对于同一个键可以有多个累加器。如果有两个或者更多的分区都有对应同一个键的累加器，就需要使用用户提供的 mergeCombiners() 方法将各个分区的结果进行合并。

针对相同 K，将 V 合并成一个集合。

#### 代码演示（计算每个 key 出现的次数以及可以对应值的总和，再相除得到结果）

      val list: List[(String, Int)] = List(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98))    val input: RDD[(String, Int)] = sc.makeRDD(list, 2)    input.combineByKey(      (_,1),      (acc:(Int,Int),v)=>(acc._1+v,acc._2+1),      (acc1:(Int,Int),acc2:(Int,Int))=>(acc1._1+acc2._1,acc1._2+acc2._2)    ).map(item=>item._2._1/item._2._2.toDouble).collect().foreach(println)结果95.3333333333333391.33333333333333

#### reduceByKey、foldByKey、aggregateByKey、combineByKey 对比

reduceByKey 的底层源码。

    def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {    combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)  }   第一个初始值不变，也即不用直接给出初始值，分区内和分区间的函数相同

foldByKey 的底层源码。

    combineByKeyWithClassTag[V]((v: V) => cleanedFunc(createZero(), v),cleanedFunc, cleanedFunc, partitioner)初始值和分区内和分区间的函数（逻辑）相同

aggregateByKey 的底层源码。

     combineByKeyWithClassTag[U]((v: V) => cleanedSeqOp(createZero(), v),cleanedSeqOp, combOp, partitioner)初始值和分区内处理逻辑一致分区内和分区间的函数（逻辑）不相同

combineByKey 的底层源码。

    def combineByKeyWithClassTag[C](      createCombiner: V => C,      mergeValue: (C, V) => C,      mergeCombiners: (C, C) => C)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {    combineByKeyWithClassTag(createCombiner, mergeValue, mergeCombiners, defaultPartitioner(self))  }把第1个值变成特定的结构

### 7. sortByKey() 排序

#### 先看 sortByKey 函数

    def sortByKey(ascending: Boolean = true, numPartitions: Int = self.partitions.length)      : RDD[(K, V)] = self.withScope  {    val part = new RangePartitioner(numPartitions, self, ascending)    new ShuffledRDD[K, V, V](self, part)      .setKeyOrdering(if (ascending) ordering else ordering.reverse)  }

#### 功能说明

在一个 (K,V) 的 RDD 上调用，K 必须实现 Ordered 接口，返回一个按照 key 进行排序的 (K,V) 的 RDD。

ascending: Boolean = true 默认为升序。

false 为降序。

#### 代码演示

     val rdd: RDD[(Int, String)] = sc.makeRDD(Array((3,"aa"),(6,"cc"),(2,"bb"),(1,"dd")))    val mapRdd: RDD[(Int, String)] = rdd.sortByKey()    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)(1,dd)(2,bb)(3,aa)(6,cc)

### 8. mapValues() 只对 V 进行操作

#### 先看 mapValues 函数

    def mapValues[U](f: V => U): RDD[(K, U)] = self.withScope {    val cleanF = self.context.clean(f)    new MapPartitionsRDD[(K, U), (K, V)](self,      (context, pid, iter) => iter.map { case (k, v) => (k, cleanF(v)) },      preservesPartitioning = true)  }

#### 功能与代码演示

针对于 (K,V) 形式的类型只对 V 进行操作。

     val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1, "a"), (1, "d"), (2, "b"), (3, "c")))    val mapRdd: RDD[(Int, String)] = rdd.mapValues(_+">>>>")    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果(1,a>>>>)(1,d>>>>)(2,b>>>>)(3,c>>>>)

### 9. join() 连接

#### 先看 join 函数

    def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))] = self.withScope {    join(other, defaultPartitioner(self, other))  }

#### 功能说明

在类型为 (K,V) 和(K,W)的 RDD 上调用，返回一个相同 key 对应的所有元素对在一起的 (K,(V,W)) 的 RDD。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1cuPcuibs6CSl0JP3micruFIGVjJIepXe2Q3m5e4SA9icsgSkWho8Crn1Q/640?wx_fmt=png)

#### 代码演示

     val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1, "a"), (2, "b"), (3, "c")))    val rdd1: RDD[(Int, Int)] = sc.makeRDD(Array((1, 4), (2, 5), (4, 6)))    val mapRdd: RDD[(Int, (String, Int))] = rdd.join(rdd1)    // 打印修改后的RDD中数据     mapRdd.collect().foreach(println)结果(1,(a,4))(2,(b,5))

### 10. cogroup() 联合

#### 先看 cogroup 函数

    def cogroup[W](other: RDD[(K, W)]): RDD[(K, (Iterable[V], Iterable[W]))] = self.withScope {    cogroup(other, defaultPartitioner(self, other))  }

#### 功能说明

在类型为 (K,V) 和(K,W)的 RDD 上调用，返回一个 (K,(Iterable,Iterable)) 类型的 RDD。

![](https://mmbiz.qpic.cn/mmbiz_png/2gY1hzDz7dTxWK5pRiaOvz1qQKxSRv4r1YZwfC7JqROZ1IJn3AWtQDbURNIFJR1Agf169PfRpOWwyPm5mAvwWWQ/640?wx_fmt=png)

#### 代码演示

        val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1,"a"),(1,"b"),(3,"c")))    val rdd1: RDD[(Int, Int)] = sc.makeRDD(Array((1,4),(2,5),(3,6)))    rdd.cogroup(rdd1).collect().foreach(println)结果(1,(CompactBuffer(a, b),CompactBuffer(4)))(2,(CompactBuffer(),CompactBuffer(5)))(3,(CompactBuffer(c),CompactBuffer(6)))

 [https://mp.weixin.qq.com/s/jEoJ6RgIKINVuM_Gfva06g](https://mp.weixin.qq.com/s/jEoJ6RgIKINVuM_Gfva06g)
