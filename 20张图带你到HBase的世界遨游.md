# 20张图带你到HBase的世界遨游
**Python 实战社群**

**Java 实战社群**

**长按识别下方二维码，\*\***按需求添加 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/gSgKWFwN1dCLic3Ix6DsRrfXwiagtLXL9K8AUVTqM0GciauLcgQOsCO9zNrfZSRrSgfOFlqf6dDWzzjq6dkvTmT0Q/640?wx_fmt=png)

**扫码关注添加客服**

**进 Python 社群▲**

![](https://mmbiz.qpic.cn/mmbiz_png/gSgKWFwN1dCLic3Ix6DsRrfXwiagtLXL9KGGZcChkjswWTHcWPERSeG79tJYgq2VgxibhepiauTUCII6FVezkzCkVg/640?wx_fmt=png)

**扫码关注添加客服**

\***\* 进 Java 社群**▲\*\*

作者丨 SoWhat1412 来源丨 sowhat1412  
![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKDyZyItbRugygiaGuc7bMeKZqJ74ZYypib1jBfR1Ulmlatf2McGiaibxLicw/640?wx_fmt=png)

### 1.1 HBase 是啥

**HBase** 是一款面向列存储，用于存储处理海量数据的 **NoSQL** 数据库。它的理论原型是 **Google** 的 **BigTable**  论文。你可以认为 **HBase** 是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统。

**HBase** 的存储是基于`HDFS`的，**HDFS**有着高容错性的特点，被设计用来部署在低廉的硬件上，基于 **Hadoop** 意味着 **HBase** 与生俱来的超强的`扩展性`和`吞吐量`。

**HBase** 采用的时`key/value`的存储方式，这意味着，即使随着数据量的增大，也几乎不会导致查询性能的下降。**HBase**又是一个`面向列`存储的数据库，当表的字段很多时，可以把其中几个字段独立出来放在一部分机器上，而另外几个字段放到另一部分机器上，充分分散了负载的压力。如此**复杂的存储结构和分布式的存储方式**，带来的代价就是即便是**存储很少的数据，也不会很快**。

**HBase** 并不是足够快，只是数据量很大的时候慢的不明显。HBase 主要用在以下两种情况：

> 1.  单表数据量超过千万，而且并发量很大。
> 2.  数据分析需求较弱，或者不需要那么实时灵活。

### 1.2 HBase 的由来

我们知道  [Mysql](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488721&idx=1&sn=eead82d2b7a0fdf993beacc4dfd60313&scene=21#wechat_redirect)  是一个关系型数据库，学数据库的时第一个接触的就是 [MySQL](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488721&idx=1&sn=eead82d2b7a0fdf993beacc4dfd60313&scene=21#wechat_redirect) 了。但是 [MySQL](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488721&idx=1&sn=eead82d2b7a0fdf993beacc4dfd60313&scene=21#wechat_redirect) 的性能瓶颈是很大的，一般单个 table 行数不宜超过 500 万行，大小不宜超过 2G。

我们以互联网公司最核心用户表为例，当数据量达到千万甚至亿级别时候，尽管你可以通过各种优化来提速查询，但是对单条数据的检索耗时还是会超出你的预期！看下这个 User 表：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfK1IhafiaURSCiahicMicDCmHqiasfc2O58JqjfnM8LSoY4iaGEiaZiaX2wZ5wpw/640?wx_fmt=png)

假如查询 id=1 这条数据对应的用户 name，系统会给我们返回 aa。但由于 MySQL 是以行为位单位存储的，当查 name 时却需要查询一整行的数据，连 age 和 email 也会被查出来！如果列非常多，那么查询效率可想而知了。

我们称列过多的表为**宽表**，优化方法一般就是对列进行`竖直拆分`：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKsnZQAWI6Y9NB2tIkDl0oWTffgjhKic38oRcChnfO7jUWFw6xJAj0VHQ/640?wx_fmt=png)

此时查找 name 时只需要查找 user_basic 表，没有多余的字段，查询效率就会很快。如果一张表的`行过多`，会影响查询效率，我们将这样的表称之为**高表**，可以采用`水平拆表`的方式提高效率：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKMhxDWIJucWR2QVHscH2NV8d5DUCiaPde4bCqjHjkHAPFh3n9ePPcEFQ/640?wx_fmt=png)

这种水平拆分应用比较多的 场景就是`日志表`，日志信息每天产生很多，可以`按月 / 按日`进行水平拆分，这样就实现了高表变矮。

上述的拆分方式貌似可以解决宽表跟高表问题，但是如果有一天公司业务变更，比如原来没有微信，现在需加入用户的微信字段。这时候需要改变表的结构信息，该怎么办？最简单的想法是多加一列，像这样：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKZeGoCgBO0wmkk8z8X9kPV0AiaEMDjf7mqvqCMsq09zL9FrqC29Lg5vA/640?wx_fmt=png)

但是你要知道不是所有用户都要微信号的，微信号这一列是设置默认值还是采取其他的做法就得权衡一下了。如果需扩展很多列出来，但不是所有的用户都有这些属性，那么拓展起来就更加复杂了。这时可以用下 JSON 格式的字符串，将若干可选择填写信息汇总，而且属性字段可以动态拓展，于是有了下边做法：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKYfBoW0USRsNTBFjM612VXgMDCzh84NkqNRlsiaZeiaYW3sf1wRmKFouA/640?wx_fmt=png)
至此你可能认为这样存储数据它不挺好的嘛，用 **HBase** 出来干嘛？**Mysql** 有个致命缺点，就是当数据达到一定的阈值，无论怎么优化，它都无法达到高性能的发挥。而大数据领域的数据，动辄 **PB**级数据量，这种存储应用明显是不能很好的满足需求的！并且针对上边的问题，**HBase**都有很好的解决方案\~~。

### 1.3 HBase 设计思路

接着上边说到的几个问题：高表、宽表、数据列动态扩展，把提到的几个解决办法：`水平切分`、`垂直切分`、`列扩展方法` 杂糅在一起。

有张表，你怕它又宽又高跟动态扩展列，那么在设计之初，就把这个表给拆开，为了`列的动态拓展`，直接存储 JSON 格式：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKzbEIQrAib54lSqJabgPoicjvqDDJ2J09e0Kl2OK5fSyBZE7icoibSgcnFA/640?wx_fmt=png)
这样就解决了宽表跟列扩展问题，高表怎么办呢？一个表按行切分成**partition**，各存一部分行：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKeTkibXlMibKwRGeFa0AssibyaiaYuWibT0yutPcicW9T1YhIVjzib69Vro1uw/640?wx_fmt=png)

解决了**高表**、**宽表**、**动态扩展列** 的问题后你会发现数据量大了速度不够快咋办？用缓存呗，查询出的数据放缓存中，下次直接从缓存拿数据。插入数据怎么办呢？也可以这样理解，我把要插入的数据放进缓存中，再也不用管了，直接由数据库从缓存拿数据插入到数据库。此时程序**不需要**等待数据插入成功，提高了并行工作的效率。

你用缓存的考虑服务器宕机后缓存中数据没来得及插入到数据库中造成丢数据咋办？参考 [**Redis**](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488832&idx=1&sn=5999893d7fe773f54f7d097ac1c2074d&chksm=ebdef478dca97d6e2433abdeecf600669ffbb1b68eb2b744e7ed72aac4cd5c4cabf19b0d8f19&scene=21#wechat_redirect) 的持久化策略，可以插入数据这个操作添加一个操作日志，用于持久化插入操作，宕机重启后从日志恢复。这样设计架构就变成了这个样子：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKMAQDesGoNGCP24n92sMTJGPNlPtXamY9XxSkCC9mFiceyRgcKsuRcibw/640?wx_fmt=png)
这就是 **HBase** 实现的大致思路。接下来正式进入 **HBase** 设计解析。

**Hbase** 官网：[http://hbase.apache.org](http://hbase.apache.org)

### 2.1  HBase 特点

1.  海量存储

> HBase 适合存储 **PB**级别的海量数据，能在几十到百毫秒内返回数据。

2.  列式存储

> HBase 是根据`列族`来存储数据的。列族下面可以有非常多的列，**在创建表的时候列族就必须指定**。

3.  高并发

> 在并发的情况下，HBase 的单个 IO 延迟下降并不多，能获得高并发、低延迟的服务。

4.  稀疏性

> HBase 的列具有灵活性，在列族中，你可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间的。

5.  极易扩展

> 1.  基于 RegionServer 的扩展，通过横向添加 RegionSever 的机器，进行水平扩展，提升 HBase 上层的处理能力，提升 HBase 服务更多 Region 的能力。
> 2.  基于存储的扩展（HDFS）。

### 2.2 HBase 逻辑结构

逻辑思维层面 HBase 的存储模型如下：![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfK9VtSnm2PjEtBPEKMgOEdxfwkfRpwjD7vUZFrdtIDI7I6GIpwgYiajEg/640?wx_fmt=png)

1.  Table(表)：

> 表由一个或者多个`列族`构成。数据的属性如 name、age、TTL(超时时间) 等都在列族里边定义。定义完列族的表是个空表，只有添加了数据行以后，表才有数据。

2.  Column (列)：

> HBase 中的每个列都由 Column Family(列族) 和 Column Qualifier(列限定符) 进行限定，例如 info：name、info：age。建表时只需指明列族，而列限定符无需预先定义。

3.  Column Family(列族)：

> 1.  多个列`组合`成一个列族。建表时不用创建列，在 HBase 中列是`可增减变化`的！唯一要确定的是`列族`，表有几个列族在开始创建时就定好的。表的很多属性，比如数据过期时间、数据块缓存以及是否使用压缩等都是定义在列族上的。
> 2.  HBase 会把相同列族的几个列数据尽量放在同一台机器上。

4.  Row(行)：

> 一行包含多个列，这些列通过列族来分类。行中的数据所属的列族从该表所定义的列族中选取。由于 HBase 是一个面向列存储的数据库，所以一个`行中的数据可以分布在不同的服务器上`。

5.  RowKey(行键)：

> RowKey 类似 [MySQL](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488721&idx=1&sn=eead82d2b7a0fdf993beacc4dfd60313&scene=21#wechat_redirect) 中的主键，在 HBase 中 RowKey 必须有且 RowKey 是按照字典排序的，如果用户不指定 RowKey 系统会自动生成不重复字符串。查询数据时**只能根据 RowKey 进行检索**，所以 Table 的 RowKey 设计十分重要。

6.  Region(区域)：

> 1.  Region 就是若干行数据的集合。HBase 中的 Region 会根据数据量的大小动态分裂，Region 是基于 HDFS 实现的，关于 Region 的存取操作都是调用 HDFS 客户端完成的。同一个行键的 Region 不会被拆分到多个 Region 服务器上。
> 2.  Region 有一点像关系型数据的分区，数据存放在 Region 中，当然 Region 下面还有很多结构，确切来说数据存放在 MemStore 和 HFile 中。访问 HBase 时先去 HBase 系统表查找定位这条记录属于哪个 Region ，然后定位到这个 Region 属于哪个服务器，然后就到哪个服务器里面查找对应 Region 中的数据。

7.  RegionServer：

> RegionServer 就是存放 Region 的容器，直观上说就是服务器上的一个服务。负责管理维护 Region。

### 2.3 HBase 物理存储

以上只是一个基本的逻辑结构，底层的物理存储结构才是重中之重的内容，看下图![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKOQ8rdVNWNZZA8x7sq6x1AT7HhIhAmKSDGcBMxRicQ7De7mwcyuYnSjA/640?wx_fmt=png)

1.  NameSpace：

> 命名空间，类似关系型数据库 DatabBase 概念，每个命名空间下有多个表。HBase 有两个自带的命名空间，分别是`hbase`和`default`，hbase 中存放的是 HBase 内置的表，default 表是用户默认使用的命名空间。

3.  TimeStamp：

> 时间戳，用于标识数据的不同版本（version），每条数据写入时如果不指定时间戳，系统会自动添加为其写入 HBase 的时间。并且读取数据的时候一般只拿出数据的 Type 符合，时间戳最新的数据。之所以按照 Type 取数据是因为 HBase 的底层`HDFS 支持增删查，但不支持改`。

4.  Cell：

> 单元格，由 {rowkey, column Family：column Qualifier, time Stamp} 唯一确定的单元。cell 中的数据是`没有`类型的，全部是`字节码`形式存储。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKDMvGz9d8NOXsfbKWcE172zMgklGHXC3OS6KMTXpTQLB7icOhpd9RJNQ/640?wx_fmt=png)

### 

3.1 Client

Client 包含了访问 Hbase 的接口，另外 Client 还维护了对应的 cache 来加速 Hbase 的访问，比如缓存元数据的信息。

### 3.2 Zookeeper

HBase 通过 [Zookeeper](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489891&idx=1&sn=eb7b6a4d4f2560df31eb41e10dc66264&chksm=ebdef85bdca9714d89dcd84894ce8c3af4c2ad5bab43760adfa30384fe7345c6d93fe55b47d5&scene=21#wechat_redirect) 来做 Master 的高可用、RegionServer 的监控、元数据的入口以及集群配置的维护等工作。Zookeeper 职责如下：

1.  通过 Zoopkeeper 来保证集群中只有 1 个 Master 在运行，如果 Master 发生异常会通过竞争机制产生新的 Master 来提供服务。
2.  通过 Zoopkeeper 来监控 RegionServer 的状态，当 RegionSevrer 有异常的时候，通过回调的形式通知 MasterRegionServer 上下线的信息。
3.  通过 Zoopkeeper 存储元数据 hbase:meata 的统一入口地址。

### 3.3 Master

Master 在 HBase 中的地位比其他类型的集群弱很多！数据的读写操作与他没有关系，它挂了之后，集群照样运行。但是 Master 也不能宕机太久，有很多必要的操作，比如创建表、修改列族配置等 DDL 跟 Region 的分割与合并都需要它的操作。

1.  负责启动的时候分配 Region 到具体的 RegionServer。
2.  发现失效的 Region，并将失效的 Region 分配到正常的 RegionServer 上。
3.  管理 HRegion 服务器的负载均衡，调整 HRegion 分布。
4.  在 HRegion 分裂后，负责新 HRegion 的分配。

HBase 中可以启动多个 Master，通过 Zookeeper 的 Master Election 机制保证总有一个 Master 运行。

### 3.4 RegionServer

HregionServer 直接对接用户的读写请求，是真正的干活的节点。它的功能概括如下：

1.  管理 Master 为其分配的 Region。
2.  处理来自客户端的读写请求。
3.  负责和底层 HDFS 的交互，存储数据到 HDFS。
4.  负责 Region 变大以后的拆分。
5.  负责 StoreFile 的合并工作。

ZooKeeper 会监控 RegionServer 的上下线情况，当 ZK 发现某个 HRegionServer 宕机之后会通知 Master 进行失效备援。下线的 RegionServer 所负责的 Region 暂时停止对外提供服务，Master 会将该 RegionServer 所负责的 Region 转移到其他 RegionServer 上，并且会对 下线 RegionServer 上存在 MemStore 中还未持久化到磁盘中的数据由 WAL 重播进行恢复。

### 3.5 WAL

WAL (Write-Ahead-Log) 预写日志是 HBase 的 RegionServer 在处理数据插入和删除的过程中用来记录操作内容的一种日志。每次 Put、Delete 等一条记录时，首先将其数据写入到 RegionServer 对应的 HLog 文件中去。只有当 WAL 日志写入成功的时候，客户端才会被告诉提交数据成功。如果写 WAL 失败会告知客户端提交失败，这其实就是数据落地的过程。

WAL 是保存在 HDFS 上的持久化文件。数据到达 Region 时先写入 WAL，然后被加载到 MemStore 中。这样就算 Region 宕机了，操作没来得及执行持久化，也可以再重启的时候从 WAL 加载操作并执行。跟[Redis](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488832&idx=1&sn=5999893d7fe773f54f7d097ac1c2074d&scene=21#wechat_redirect)的 AOF 类似。

1.  在一个 RegionServer 上的所有 Region 都**共享**一个 HLog，一次数据的提交先写入 WAL，写入成功后，再写入 MenStore 之中。当 MenStore 的值达到一定的时候，就会形成一个个 StoreFile。
2.  WAL **默认是开启** 的，也可以手动关闭它，这样增删改操作会快一点。但是这样做牺牲的是数据的安全性。如果不想关闭 WAL, 又不想每次都耗费那么大的资源，每次改动都调用 HDFS 客户端，可以选择**异步**的方式写入 WAL(默认间隔 1 秒写入)
3.  如果你学过 Hadoop 中的 Shuffle(edits 文件) 机制的就可以猜测到 HBase 中的 WAL 也是一个滚动的日志数据结构，一个 WAL 实例包含多个 WAL 文件，WAL 被触发滚动的条件如下。

> 1.  WAL 的大小超过了一定的阈值。
> 2.  WAL 文件所在的 HDFS 文件块快要满了。
> 3.  WAL 归档和删除。

### 3.5 Region

每一个 Region 都有起始 RowKey 和结束 RowKey，代表了存储的 Row 的范围。从大图中可知一个 Region 有多个 Store，一个 Store 就是对应一个列族的数据，Store 由 MemStore 和 HFile 组成的。

### 3.6 Store

Store 由 MemStore 跟 HFile 两个重要的部分。

##### 3.6.1 MemStore

每个 Store 都有一个 MemStore 实例，数据写入到 WAL 之后就会被放入 MemStore 中。MemStore 是内存的存储对象，当 MemStore 的大小达到一个阀值（默认 64MB）时，MemStore 会被 flush 到文件，即生成一个快照。目前 HBase 会有一个线程来负责 MemStore 的 flush 操作。

##### 3.6.2 StoreFile

MemStore 内存中的数据写到文件后就是 StoreFile，StoreFile 底层是以 HFile 的格式保存。HBase 以 Store 的大小来判断是否需要切分 Region。

##### 3.6.3 HFile

在 Store 中有多个 HFile，每次刷写都会形成一个 HFile 文件落盘在 HDFS 上。HFile 文件也会动态合并，它是数据存储的实体。

这里提出一点疑问：操作到达 Region 时，数据进入 HFile 之前就已经被持久化到 WAL 了，而 WAL 就是在 HDFS 上的，为什么还要从 WAL 加载到 MemStore 中，再刷写成 HFile 呢？

> 1.  由于 HDFS 支持文件创建、追加、删除，但不能修改！但对数据库来说，数据的顺序非常重要！
> 2.  第一次 WAL 的持久化是为了保证数据的安全性，无序的。
> 3.  再读取到 MemStore 中，是为了排序后存储。
> 4.  所以 MemStore 的意义在于维持数据按照 RowKey 的字典序排列，而不是做一个缓存提高写入效率。

### 3.7 HDFS

HDFS 为 HBase 提供最终的底层数据存储服务，HBase 底层用 HFile 格式 (跟 hadoop 底层的数据存储格式类似) 将数据存储到 HDFS 中，同时为 HBase 提供高可用（Hlog 存储在 HDFS）的支持，具体功能概括如下：

> 1.  提供元数据和表数据的底层分布式存储服务
> 2.  数据多副本，保证的高可靠和高可用性

在 HBase 集群中如果我们做 DML 操作是不需要关心 HMaster 的，只需要从 ZooKeeper 中获得 hbase:meta 数据地址，然后从 RegionServer 中增删查数据即可。

### 4.1 HBase 写流程

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKGFCY04Yh8pqvoqicPYotJtS9abfGrf8zwC19D9apACHQglVL1OPNZxw/640?wx_fmt=png)

1.  Client 先访问 zookeeper，访问 /hbase/meta-region-server 获取 hbase:meta 表位于哪个 Region Server。
2.  访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 Region 信息以及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。
3.  与目标 Region Server 进行通讯。
4.  将数据顺序写入（追加）到 WAL。
5.  将数据写入对应的 MemStore，数据会在 MemStore 进行排序。
6.  向客户端发送 ack，此处可看到数据不是必须落盘的。
7.  等达到 MemStore 的刷写时机后，将数据刷写到 HFile
8.  在 web 页面查看的时候会随机的给每一个 Region 生成一个随机编号。

### 4.2 HBase 读流程

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKyianncmrRuKdDziclDvthYgt2Ltr228Mlb983zZNRjxjxibXP2b3iasHKg/640?wx_fmt=png)

1.  Client 先访问 ZooKeeper，获取 hbase:meta 表位于哪个 Region Server。
2.  访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey， 查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息以 及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。
3.  与目标 Region Server 进行通讯。
4.  分别在 Block Cache(读缓存)，MemStore 和 Store File(HFile)中查询目标数据，并将 查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本 (time stamp) 或者不同的类型(Put/Delete)。
5.  将从文件 HFile 中查询到的数据块 (Block，HFile 数据存储单元，默认大小为 64KB) 缓存到 Block Cache。
6.  将合并后的最终结果，然后返回时间最新的数据返回给客户端。

##### 4.2.1 Block Cache

HBase 在实现中提供了两种缓存结构 MemStore(写缓存) 和 BlockCache(读缓存)。写缓存前面说过不再重复。

1.  HBase 会将一次文件查找的 Block 块 缓存到 Cache 中，以便后续同一请求或者邻近数据查找请求，可以直接从内存中获取，避免昂贵的 IO 操作。
2.  BlockCache 是 Region Server 级别的，
3.  一个 Region Server 只有一个 Block Cache，在 Region Server 启动的时候完成 Block Cache 的初始化工作。
4.  HBase 对 Block Cache 的管理分为如下三种。

> 1.  **LRUBlockCache** 是最初的实现方案，也是默认的实现方案，将所有数据都放入 JVM Heap 中，交给 JVM 进行管理。
> 2.  **SlabCache** 实现的是堆外内存存储，不再由 JVM 管理数据内存。一般跟第一个组合使用，单它没有改善 GC 弊端，引入了堆外内存利用率低。
> 3.  **BucketCache** 缓存淘汰不再由 [JVM](http://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247489183&idx=1&sn=02ab3551c473bd2c8429862e3689a94b&chksm=ebdef7a7dca97eb17194c3d935c86ade240d3d96bbeaf036233a712832fb94af07adeafa098b&scene=21#wechat_redirect) 管理 降低了 Full GC 发生的频率。

**重点**：

> 读数据时`不要理解`为先从 MemStore 中读取，读不到再读 BlockCache 中，还读不到再从 HFile 中读取，然后将数据写入到 BlockCache 中。因为如果人为设置导致磁盘数据 new，内存数据 old。你读取的时候会出错的！

**结论**：

> HBase 把磁盘跟内存数据一起读，然后把磁盘数据放到 BlockCache 中，BlockCache 是磁盘数据的缓存。HBase 是个读比写慢的工具。

### 4.3 HBase 为什么写比读快

1.  HBase 能提供实时计算服务主要原因是由其架构和底层的数据结构决定的，即由**LSM-Tree**(Log-Structured Merge-Tree) + **HTable**(Region 分区) + **Cache**决定的。
2.  HBase 写入速度快是因为数据并不是真的立即落盘，而是先写入内存，随后异步刷入 HFile。所以在客户端看来，写入速度很快。
3.  HBase 存储到内存中的数据是有序的，内存数据刷写到 HFile 时也是有序的。并且多个有序的 HFile 还会进行**归并排序**生成更大的有序 HFile。性能测试发现顺序读写磁盘速度比随机读写磁盘快至少三个数量级！![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKKnLI7eh0OBgfLD5q3wyRfT85HKVRksKrRfibn56lJibjk36ICcAkY3kA/640?wx_fmt=png)
4.  读取速度快是因为它使用了**LSM**树型结构，因为磁盘寻址耗时远远大于磁盘顺序读取的时间，HBase 的架构设计导致我们可以将磁盘寻址次数控制在性能允许范围内。
5.  LSM 树原理把一棵大树拆分成 N 棵小树，它首先写入内存中，随着小树越来越大，内存中的小树会 flush 到磁盘中，磁盘中的树定期可以做 merge 操作来合并成一棵大树，以优化读性能。

##### 4.3.1 查询举例

1.  根据 RowKey 能快速找到行所在的 Region，假设有 10 亿条记录，占空间 1TB。分列成了 500 个 Region，那读取 2G 的记录，就能找到对应记录。
2.  数据是按照列族存储的，假设分为 3 个列族，每个列族就是 666M， 如果要查询的东西在其中 1 个列族上，1 个列族包含 1 个或者多个 HStoreFile，假设一个 HStoreFile 是 128M， 该列族包含 5 个 HStoreFile 在磁盘上. 剩下的在内存中。
3.  内存跟磁盘中数据是排好序的，你要的记录有可能在最前面，也有可能在最后面，假设在中间，我们只需遍历 2.5 个 HStoreFile 共 300M。
4.  每个 HStoreFile(HFile 的封装)，是以键值对 (KV) 方式存储，只要遍历一个个数据块中的 key 的位置，并判断符合条件可以了。一般 key 是有限的长度，假设 KV 比是 1:19，最终只需要 15M 就可获取的对应的记录，按照磁盘的访问 100M/S，只需 0.15 秒。加上 Block Cache 会取得更高的效率。
5.  大致理解读写思路后你会发现如果你在读写时设计的足够巧妙当然读写速度快的很咯。

### 5.1 Flush

对于用户来说数据写到 MemStore 中就算 OK，但对于底层代码来说只有数据刷到硬盘中才算彻底搞定了！因为数据是要写入到 WAL(Hlog) 中再写入到 MemStore 中的，flush 有如下几个时机。

1.  当 WAL 文件的数量超过设定值时 Region 会按照时间顺序依次进行刷写，直到 WAL 文件数量小于设定值。
2.  当 Region Server 中 MemStore 的总大小达到堆内存 40% 时，Region 会按照其所有 MemStore 的大小顺序（由大到小）依次进行阻塞刷写。直到 Region Server 中所有 MemStore 的总大小减小到上述值以下。当阻塞刷写到上个参数的 0.95 倍时，客户端可以继续写。
3.  当某个 MemStore 的大小达到了 128M 时，其所在 Region 的所有 MemStore 都会阻塞刷写。
4.  到达自动刷写的时间也会触发 MemStore 的 flush。自动刷新的时间间隔默认 1 小时。

### 5.2 StoreFile Compaction

由于 MemStore 每次刷写都会生成一个新的 HFile，且同一个字段的不同版本 (timestamp) 和不同类型(Put/Delete) 有可能会分布在不同的 HFile 中，因此查询时需要遍历所有的 HFile。为了减少 HFile 的个数跟清理掉过期和删除的数据，会进行 StoreFile Compaction。

Compaction 分为两种，分别是 **Minor Compaction** 和 **Major Compaction**。

> 1.  **Minor Compaction**会将临近的若干个较小的 HFile 合并成一个较大的 HFile，但不会清理过期和删除的数据。
> 2.  **Major Compaction** 会将一个 Store 下的所有的 HFile 合并成一个大 HFile，并且会清理掉过期和删除的数据。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfK67h5YbOwYGxXb979YuSOiatQ5FVqkslxuAyJqXRhMy52OAwLQg7L3yg/640?wx_fmt=png)

### 5.3 Region Split

每个 Table 起初只有一个 Region，随着不断写数据 Region 会自动进行拆分。刚拆分时，两个子 Region 都位于当前的 Region Server，但出于负载均衡的考虑， HMaster 有可能会将某个 Region 转移给其他的 Region Server。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKbpbhhKQbkibBzbKy6UWEdF8yKkYLUJQ9fYR9TsZgknsEYX04h4PNm5g/640?wx_fmt=png)

Region Split 时机:

1.  0.94 版本之前：

> 当 1 个 Region 中的某个 Store 下所有 StoreFile 的总大小超过 hbase.hregion.max.filesize(默认 10G)， 该 Region 就会进行拆分。

2.  0.94 版本之后:

> 当 1 个 Region 中的某个 Store 下所有 StoreFile 的总大小超过 Min(R^2 \* “hbase.hregion.memstore.flush.size=128M”,hbase.hregion.max.filesize")，该 Region 就会进行拆分，其 中 R 为当前 Region Server 中属于该 Table 的个数。

举例：

> 1.  第一次的阈值是 128，切分后结果 64 , 64。
> 2.  第二次阈值 512M，64,512 ⇒ 54 + 256 + 256
> 3.  最后会形成一个 64M…10G 的这样 Region 队列，会产生数据倾斜问题。
> 4.  解决方法：**提前做好 Region 组的规划**，0-1k,1k-2k,2k-3k 这样的。

**官方不建议用多个列族**，比如有 CF1，CF2，CF3，但是 CF1 数据很多而 CF2 跟 CF3 数据很少，那么当触发了 region 切分的时候，会把 CF2 跟 CF3 分成若干小份，不利于系统维护。

### 6.1 Hbase 中 RowKey 的设计原则

1.  RowKey  长度原则

> 二进制码流 RowKey 最大长度 64Kb，实际应用中一般为 10-100bytes，以 byte\[] 形式保存，一般设计定长。建议越短越好，因为 HFile 是按照 KV 存储的 Key 太大浪费空间。

2.  RowKey 散列原则

> RowKey 在设计时候要尽可能的实现可以将数据均衡的分布在每个 RegionServer 上。

3.  RowKey  唯一原则

> RowKey  必须在设计上保证其唯一性，RowKey  是按照字典顺序排序存储的，因此设计 RowKey 时可以将将经常读取的数据存储到一块。

### 6.2 HBase 在大数据体系位置

其实就简单的把 HBase 当成大数据体系下的 DataBase 来用就行，任何可以分析 HBase 的引擎比如 MR、Hive、Spark 等框架连接上 HBase 都可以实现控制。比如你可以把 Hive 跟 HBase 进行关联，Hive 中数据不再由 HDFS 存储而是存储到 HBase 中，并且关联后 Hive 中添加数据在 HBase 中可看到，HBase 中添加数据 Hive 也可看到。

### 6.3 HBase 优化方法

##### 6.3.1 减少调整

HBase 中有几个内容会动态调整，如 Region（分区）、HFile。通过一些方法可以减少这些会带来 I/O 开销的调整。

1.  Region

> 没有预建分区的话，随着 Region 中条数的增加，Region 会进行分裂，这将增加 I/O 开销，所以解决方法就是根据你的 RowKey 设计来进行预建分区，减少 Region 的动态分裂。

2.  HFile

> MemStore 执行 flush 会生成 HFile，同时 HFilewe 年过多时候也会进行 Merge， 为了减少这样的无谓的 I/O 开销，建议估计项目数据量大小，给 HFile 设定一个合适的值。

##### 6.3.2 减少启停

数据库事务机制就是为了更好地实现批量写入，较少数据库的开启关闭带来的开销，那么 HBase 中也存在频繁开启关闭带来的问题。

1.  关闭 Compaction。

> HBase 中自动化的 Minor Compaction 和 Major Compaction 会带来极大的 I/O 开销，为了避免这种不受控制的意外发生，建议关闭自动 Compaction，在闲时进行 compaction。

##### 6.3.3  减少数据量

1.  开启过滤，提高查询速度

> 开启 BloomFilter，BloomFilter 是列族级别的过滤，在生成一个 StoreFile 同时会生成一个 MetaBlock，用于查询时过滤数据

2.  使用压缩

> 一般推荐使用 Snappy 和 LZO 压缩

##### 6.3.4 合理设计

HBase 表格中 RowKey 和 ColumnFamily 的设计是非常重要，好的设计能够提高性能和保证数据的准确性。

1.  RowKey 设计

> 1.  **散列性**：散列性能够保证相同相似的 RowKey 聚合，相异的 RowKey 分散，有利于查询
> 2.  **简短性**：RowKey 作为 key 的一部分存储在 HFile 中，如果为了可读性将 rowKey 设计得过长，那么将会增加存储压力.
> 3.  **唯一性**：rowKey 必须具备明显的区别性。
> 4.  **业务性**：具体情况具体分析。

2.  列族的设计

> 1.  **优势**：HBase 中数据是按列进行存储的，那么查询某一列族的某一列时就不需要全盘扫描，只需要扫描某一列族，减少了读 I/O。
> 2.  **劣势**：多列族意味这一个 Region 有多个 Store，一个 Store 就有一个 MemStore，当 MemStore 进行 flush 时，属于同一个 Region 的 Store 中的 MemStore 都会进行 flush，增加 I/O 开销。

### 6.4 HBase 跟关系型数据库区别

| 指标   | 传统关系数据库     | HBase        |
| ---- | ----------- | ------------ |
| 数据类型 | 有丰富的数据类型    | 字符串          |
| 数据操作 | 丰富操作，复杂联表查询 | 简单 CRUD      |
| 存储模式 | 基于行存储       | 基于列存储        |
| 数据索引 | 复杂的多个索引     | 只有 RowKey 索引 |
| 数据维护 | 新覆盖旧        | 多版本          |
| 可伸缩性 | 难实现横向扩展     | 性能动态伸缩       |

### 6.5 HBase 批量导入

1.  通过 HBase API 进行批量写入数据。
2.  使用 Sqoop 工具批量导数到 HBase 集群。
3.  使用 MapReduce 批量导入。
4.  HBase BulkLoad 的方式。
5.  HBase 通过 Hive 关联导入数据。

大数据导入用 HBase API 跟 MapReduce 写入效率会很低，因为请求 RegionServer 将数据写入，这期间数据会先写入 WAL 跟 MemStore，MemStore 达到阈值后会刷写到磁盘生成 HFile 文件，HFile 文件过多时会发生 Compaction，如果 Region 大小过大时也会发生 Split。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKaUyOorJXr1VAOUlyxWz2tuWx0iaiasicmb1gxQpTgytgGaiaPGBaaxBDMA/640?wx_fmt=png)

BulkLoad 适合初次数据导入，以及 HBase 与 Hadoop 为同一集群。BulkLoad 是使用 MapReduce 直接生成 HFile 格式文件后，Region Servers 再将 HFile 文件移动到相应的 Region 目录下。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dWCod5zH8NPNkCNzSCicpFfKI7FthEFwice0SiaZEDYKia3SE5U0V7LXaysfJ8ItHtud8hWciaHU3SFk4g/640?wx_fmt=png)

> 1.  BlockCache 讲解：[https://blog.51cto.com/12445535/2363376?source=dra](https://blog.51cto.com/12445535/2363376?source=dra)
> 2.  LSM 原理：[https://www.zhihu.com/question/19887265](https://www.zhihu.com/question/19887265)
> 3.  HBase 教程：[http://c.biancheng.net/view/6499.html](http://c.biancheng.net/view/6499.html)

* * *

 [https://mp.weixin.qq.com/s/4_eQSFFvy63Uw1xQfhbAlQ](https://mp.weixin.qq.com/s/4_eQSFFvy63Uw1xQfhbAlQ)
