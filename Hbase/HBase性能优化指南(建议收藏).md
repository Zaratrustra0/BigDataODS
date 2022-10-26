# HBase性能优化指南(建议收藏)
本文集合了小编在日常学习和生产实践中遇到的使用 Hbase 中的各种问题和优化方法，分别从表设计、rowkey 设计、内存、读写、配置等各个领域对 Hbase 常用的调优方式进行了总结，希望能对读者有帮助。本文参考结合自己实际优化经验，参考了大量官网和各个前辈的经验，调优后生产环境中的 Hbase 集群支撑了约 50 万 / s 的读和 25 万 / s 的写流量洪峰。感谢各位的经验和付出。

#### HBase 简介

HBase 是一个分布式的、面向列的开源数据库存储系统，是对 Google 论文 BigTable 的实现，具有高可靠性、高性能和可伸缩性，它可以处理分布在数千台通用服务器上的 PB 级的海量数据。BigTable 的底层是通过 GFS（Google 文件系统）来存储数据，而 HBase 对应的则是通过 HDFS（Hadoop 分布式文件系统）来存储数据的。

HBase 不同于一般的关系型数据库，它是一个适合于非结构化数据存储的数据库。HBase 不限制存储的数据的种类，允许动态的、灵活的数据模型。HBase 可以在一个服务器集群上运行，并且能够根据业务进行横向扩展。

Hbase 有以下优点：

-   海量存储：HBase 适合存储 PB 级别的海量数据，在 PB 级别的数据以及采用廉价 PC 存储的情况下，能在几十到百毫秒内返回数据。这与 HBase 的记忆扩展性息息相关。正是因为 HBase 的良好扩展性，才为海量数据的存储提供了便利。
-   列式存储：列式存储，HBase 是根据列族来存储数据的。列族下面可以有非常多的列，列族在创建表的时候就必须指定，而不用指定列。
-   极易扩展：HBase 的扩展性主要体现在两个方面，一个是基于上层处理能力（RegionServer）的扩展，一个是基于存储能力（HDFS）的扩展。
-   高并发：目前大部分使用 HBase 的架构，都是采用廉价 PC，因此单个 IO 的延迟其实并不小，一般在几十到上百 ms 之间。这里说的高并发，主要是在并发的情况下，HBase 的单个 IO 延迟下降并不多。
-   稀疏：稀疏主要是针对 HBase 列的灵活性，在列族中，可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWPtZDZZaZIkD2TNBBphGQ25K7o4vxwPa3nnQ92t10ceMG3lkCZhs9yA/640?wx_fmt=png)

从我们使用 Hbase 开始，开发和调优将会一直伴随在系统的整个生命周期。笔者将对自己熟悉和使用过的调优方式进行一一归纳和总结。

#### 表的设计之预分区优化

HBase 表在刚刚被创建时，只有 1 个分区（region），当一个 region 过大（达到 hbase.hregion.max.filesize 属性中定义的阈值，默认 10GB）时表将会进行 split，分裂为 2 个分区。表在进行 split 的时候，会耗费大量的资源，频繁的分区对 HBase 的性能有巨大的影响。HBase 提供了预分区功能，即用户可以在创建表的时候对表按照一定的规则分区。

如果业务要进行预分区，首先要明确 rowkey 的取值范围或构成逻辑，假设我们的 rowkey 组成为例：两位随机数 + 时间戳 + 客户号，两位随机数的范围从 00-99，于是我划分了 10 个 region 来存储数据, 每个 region 对应的 rowkey 范围如下：-10,10-20,20-30,30-40,40-50,50-60,60-70,70-80,80-90,90-。

#### 表的设计之 rowkey 优化

我们在之前的文章**[《HBase 优化笔记》](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247487378&idx=2&sn=f9d4400a0677a70a8df94cc0530831e3&chksm=fd3d4907ca4ac011d278ad227be1c902f1ab9408e227f27c6082cfc2580e74fe1788365f97b1&scene=21#wechat_redirect)**和**[《设计 HBase RowKey 需要注意的二三事》](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247487369&idx=1&sn=6ad49150d939209cc59965be922ae5ed&chksm=fd3d491cca4ac00a82671803f60c1848932bc45aa649dcfe4f0f765fe24cc364a1bab81ec550&scene=21#wechat_redirect)**中详细讲解过。

在 HBase 中，定位一条数据（即一个 Cell）需要 4 个维度的限定：行键（RowKey）、列族（Column Family）、列限定符（Column Qualifier）、时间戳（Timestamp）。其中，RowKey 是最容易出现问题的。除了根据业务和查询需求来设计之外，还需要注意以下三点。

###### 打散 RowKey

HBase 中的行是按照 RowKey 字典序排序的。这对 Scan 操作非常友好，因为 RowKey 相近的行总是存储在相近的位置，顺序读的效率比随机读要高。但是，如果大量的读写操作总是集中在某个 RowKey 范围，那么就会造成 Region 热点，拖累 RegionServer 的性能。因此，要适当地将 RowKey 打散。

###### 加盐（salting）+ 哈希（hashing）

这里的 “加盐” 与密码学中的 “加盐” 不是一回事。它是指在 RowKey 的前面增加一些前缀。加盐的前缀种类越多，RowKey 就被打得越散。前缀不可以是随机的，因为必须要让客户端能够完整地重构 RowKey。我们一般会拿原 RowKey 或其一部分计算 hash 值，然后再对 hash 值做运算作为前缀。

###### 反转固定格式的数值

以手机号为例，手机号的前缀变化比较少（如 152、185 等），但后半部分变化很多。如果将它反转过来，可以有效地避免热点。不过其缺点就是失去了有序性。反转时间 这个操作严格来讲不算 “打散”，但可以调整数据的时间排序。如果将时间按照字典序排列，最近产生的数据会排在旧数据后面。如果用一个大值减去时间（比如用 99999999 减去 yyyyMMdd，或者 Long.MAX_VALUE 减去时间戳），最新的数据就可以排在前面了。

###### 控制 RowKey 长度

在 HBase 中，RowKey、列族、列名等都是以 byte\[]形式传输的。RowKey 的最大长度限制为 64KB，但在实际应用中最多不会超过 100B。设计短 RowKey 有以下两方面考虑：

在 HBase 的底层存储 HFile 中，RowKey 是 KeyValue 结构中的一个域。假设 RowKey 长度 100B，那么 1000 万条数据中，只算 RowKey 就占用掉将近 1G 空间，会影响 HFile 的存储效率。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWqm4XCA3Vb6BVMKVSemGLdQBg3ibE7eokj9Q5HdickvEoQB8DBkHZsRdw/640?wx_fmt=png)

HBase 中设计有 MemStore 和 BlockCache，分别对应列族 / Store 级别的写入缓存，和 RegionServer 级别的读取缓存。如果 RowKey 过长，缓存中存储数据的密度就会降低，影响数据落地或查询效率。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWDLRTFjxouxc7FAvkllpPkASQ9zy8HVWrW2HbTmbTCFapPqHicYpI8BQ/640?wx_fmt=png)

另外，我们目前使用的服务器操作系统都是 64 位系统，内存是按照 8B 对齐的，因此设计 RowKey 时一般做成 8B 的整数倍，如 16B 或者 24B，可以提高寻址效率。同样地，列族、列名的命名在保证可读的情况下也应尽量短。HBase 官方不推荐使用 3 个以上列族，因此实际上列族命名几乎都用一个字母，比如‘c’或‘f’。

###### 保证 RowKey 唯一性

这个就是显而易见的了，不再赘述。

#### JVM 调优

这部分我们可以参考禅克大佬发表的一些关于 Hbase 内存设置的参数。

###### 合理配置 JVM 内存

首先涉及 HBase 服务的堆内存设置。一般刚部署的 HBase 集群，默认配置只给 Master 和 RegionServer 分配了 1G 的内存，RegionServer 中 MemStore 默认占 0.4 即 400MB 左右的空间，而一个 MemStore 刷写阈值默认 128M，所以一个 RegionServer 也就能正常管理 3 个 Region，多了就可能会产生小文件了，另外也容易发生 Full GC。因此建议合理调整 Master 和 RegionServer 的内存，比如：

export HBASE_MASTER_OPTS="$HBASE_MASTER_OPTS -Xms8g -Xmx8g"  
export HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS -Xms32g -Xmx32g"

这里也要根据实际集群资源进行配置，另外要牢记至少留 10% 的内存给操作系统使用。

###### 选择合适的 GC 策略

另一个重要方面是 HBase JVM 的 GC 优化，其实 HBase 读写路径上的很多设计都是围绕 GC 优化做的。选择合适的 GC 策略非常重要，对于 HBase 而言通常有两种可选 GC 方案：

-   ParallelGC 和 CMS 组合
-   G1

CMS 避免不了 Full GC，而且 Full GC 场景下会通过一次串行的完整垃圾收集来回收碎片化的内存，这个过程通常会比较长，应用线程会发生长时间的 STW 停顿，不响应任何请求；而 G1 适合大内存的场景，通过把堆内存划分为多个 Region（不是 HBase 中的 Region），然后对各个 Region 单独进行 GC，这样就具有了并行整理内存碎片的功能，可以最大限度的避免 Full GC 的到来，提供更加合理的停顿时间。

由于 Master 只是做一些管理操作，实际处理读写请求和存储数据的都是 RegionServer 节点，所以一般内存问题都出在 RegionServer 上。

这里给的建议是，小堆（4G 及以下）选择 CMS，大堆（32G 及以上）考虑用 G1，如果堆内存介入 4~32G 之间，可自行测试下两种方案。剩下来的就是 GC 参数调优了。

###### 开启 MSLAB 功能

这是 HBase 自己实现了一套以 MemStore 为最小单元的内存管理机制，称为 MSLAB（MemStore-Local Allocation Buffer），主要作用是为了减少内存碎片化，改善 Full GC 发生的情况。

MemStore 会在内部维护一个 2M 大小的 Chunk 数组，当写入数据时会先申请 2M 的 Chunk，将实际数据写入该 Chunk 中，当该 Chunk 满了以后会再申请一个新的 Chunk。这样 MemStore Flush 后会达到粗粒度化的内存碎片效果，可以有效降低 Full GC 的触发频率。

HBase 默认是开启 MSLAB 功能的，和 MSLAB 相关的配置包括：

-   hbase.hregion.memstore.mslab.enabled：MSLAB 开关，默认为 true，即打开 MSLAB。
-   hbase.hregion.memstore.mslab.chunksize：每个 Chunk 的大 小，默认为 2MB，建议保持默认值。
-   hbase.hregion.memstore.chunkpool.maxsize：内部 Chunk Pool 功能，默认为 0 ，即关闭 Chunk Pool 功能。设置为大于 0 的值才能开启，取值范围为 \[0,1]，表示 Chunk Pool 占整个 MemStore 内存大小的比例。
-   hbase.hregion.memstore.chunkpool.initialsize：表示初始化时申请多少个 Chunk 放到 Chunk Pool 中，默认为 0，即初始化时不申请 Chuck，只在写入数据时才申请。
-   hbase.hregion.memstore.mslab.max.allocation：表示能放入 MSLAB 的最大单元格大小，默认为 256KB，超过该大小的数据将从 JVM 堆分配空间而不是 MSLAB。

出于性能优化考虑，建议检查相关配置，确保 MSLAB 处于开启状态。

###### 考虑开启 BucketCache

这块涉及到读缓存 BlockCache 的策略选择。首先，BlockCache 是 RegionServer 级别的，一个 RegionServer 只有一个 BlockCache。BlockCache 的工作原理是读请求会首先检查 Block 是否存在于 BlockCache，存在就直接返回，如果不存在再去 HFile 和 MemStore 中获取，返回数据时把 Block 缓存到 BlockCache 中，后续同一请求或临近查询可以直接从 BlockCache 中获取，避免过多的昂贵 IO 操作。BlockCache 默认是开启的。

目前 BlockCache 的实现方案有三种：

(1) LRUBlockCache 最早的 BlockCache 方案，也是 HBase 目前默认的方案。LRU 是 Least Recently Used 的缩写，称为近期最少使用算法。LRUBlockCache 参考了 JVM 分代设计的思想，采用了缓存分层设计。

LRUBlockCache 将整个 BlockCache 分为 single-access（单次读取区）、multi-access（多次读取区）和 in-memory 三部分，默认分别占读缓存的 25%、50%、25%。其中设置 IN_MEMORY=true 的列族，Block 被读取后才会直接放到 in-memory 区，因此建议只给那些数据量少且访问频繁的列族设置 IN_MEMORY 属性。另外，HBase 元数据比如 meta 表、namespace 表也都缓存在 in-memory 区。

(2) SlabCache HBase 0.92 版本引入的一种方案，起初是为了避免 Full GC 而引入的一种堆外内存方案，并与 LRUBlockCache 搭配使用，后来发现它对 Full GC 的改善很小，以至于这个方案基本被弃用了。

(3) BucketCache HBase 0.96 版本引入的一种方案，它借鉴了 SlabCache 的设计思想，是一种非常高效的缓存方案。实际应用中，HBase 将 BucketCache 和 LRUBlockCache 搭配使用，称为组合模式（CombinedBlockCahce），具体地说就是把不同类型的 Block 分别放到 LRUBlockCache 和 BucketCache 中。

HBase 会把 Index Block 和 Bloom Block 放到 LRUBlockCache 中，将 Data Block 放到 BucketCache 中，所以读取数据会去 LRUBlockCache 查询一下 Index Block，然后再去 BucketCache 中查询真正的数据。

**BucketCache 涉及的常用参数有：** 

-   hbase.bucketcache.ioengine：使用的存储介质，可设置为 heap、offheap 或 file，其中 heap 表示空间从 JVM 堆中申请，offheap 表示使用 DirectByteBuffer 技术实现堆外内存管理，file 表示使用类似 SSD 等存储介质缓存数据。默认值为空，即关闭 BucketCache，一般建议开启 BucketCache。此外，HBase 2.x 不再支持 heap 选型。
-   hbase.bucketcache.combinedcache.enabled：是否打开 CombinedBlockCache 组合模式，默认为 true。此外，HBase 2.x 不再支持该参数。
-   hbase.bucketcache.size：BucketCache 大小，取值有两种，一种是\[0,1]之间的浮点数值，表示占总内存的百分比，另一种是大于 1 的值，表示占用内存大小，单位 MB。

根据上面的分析，一般建议开启 BucketCache，综合考虑成本和性能，建议比较合理的介质是：LRUBlockCache 使用内存，BuckectCache 使用 SSD，HFile 使用机械磁盘。

###### 合理配置读写缓存比例

HBase 为了优化性能，在读写路径上分别设置了读缓存和写缓存，参数分别是 hfile.block.cache.size 与 hbase.regionserver.global.memstore.size，默认值都是 0.4，表示读写缓存各占 RegionServer 堆内存的 40%。

在一些场景下，我们可以适当调整两部分比例，比如写多读少的场景下我们可以适当调大写缓存，让 HBase 更好的支持写业务，相反类似，总之两个参数要配合调整。

#### 读优化

我们在之前[**《HBase 生产环境优化不完全指南》**](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247487391&idx=2&sn=420d44df8c68349212df31fa4471576d&chksm=fd3d490aca4ac01c7536e0f628cd82be97d5d9dba1b9619a0c637795731813cc36acb8c52079&scene=21#wechat_redirect)一文中也提到过，大家可以参考这里。范欣欣前辈曾经就读写问题提过非常好的建议 ([http://hbasefly.com/](http://hbasefly.com/))，非常建议大家仔细读一读。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRW2hErNFFPfziaRxuic3ianUITtibLG2dFXaRJGh7Z1oYg334HKQANYCficYg/640?wx_fmt=png)

###### HBase 客户端优化

和大多数系统一样，客户端作为业务读写的入口，姿势使用不正确通常会导致本业务读延迟较高实际上存在一些使用姿势的推荐用法，这里一般需要关注四个问题：

**1. scan 缓存是否设置合理？**

优化原理：在解释这个问题之前，首先需要解释什么是 scan 缓存，通常来讲一次 scan 会返回大量数据，因此客户端发起一次 scan 请求，实际并不会一次就将所有数据加载到本地，而是分成多次 RPC 请求进行加载，这样设计一方面是因为大量数据请求可能会导致网络带宽严重消耗进而影响其他业务，另一方面也有可能因为数据量太大导致本地客户端发生 OOM。在这样的设计体系下用户会首先加载一部分数据到本地，然后遍历处理，再加载下一部分数据到本地处理，如此往复，直至所有数据都加载完成。数据加载到本地就存放在 scan 缓存中，默认 100 条数据大小。

通常情况下，默认的 scan 缓存设置就可以正常工作的。但是在一些大 scan（一次 scan 可能需要查询几万甚至几十万行数据）来说，每次请求 100 条数据意味着一次 scan 需要几百甚至几千次 RPC 请求，这种交互的代价无疑是很大的。因此可以考虑将 scan 缓存设置增大，比如设为 500 或者 1000 就可能更加合适。笔者之前做过一次试验，在一次 scan 扫描 10w + 条数据量的条件下，将 scan 缓存从 100 增加到 1000，可以有效降低 scan 请求的总体延迟，延迟基本降低了 25% 左右。

优化建议：大 scan 场景下将 scan 缓存从 100 增大到 500 或者 1000，用以减少 RPC 次数

**2. get 请求是否可以使用批量请求？**

优化原理：HBase 分别提供了单条 get 以及批量 get 的 API 接口，使用批量 get 接口可以减少客户端到 RegionServer 之间的 RPC 连接数，提高读取性能。另外需要注意的是，批量 get 请求要么成功返回所有请求数据，要么抛出异常。

优化建议：使用批量 get 进行读取请求

**3. 请求是否可以显示指定列族或者列？**

优化原理：HBase 是典型的列族数据库，意味着同一列族的数据存储在一起，不同列族的数据分开存储在不同的目录下。如果一个表有多个列族，只是根据 Rowkey 而不指定列族进行检索的话不同列族的数据需要独立进行检索，性能必然会比指定列族的查询差很多，很多情况下甚至会有 2 倍～3 倍的性能损失。

优化建议：可以指定列族或者列进行精确查找的尽量指定查找

**4. 离线批量读取请求是否设置禁止缓存？**

优化原理：通常离线批量读取数据会进行一次性全表扫描，一方面数据量很大，另一方面请求只会执行一次。这种场景下如果使用 scan 默认设置，就会将数据从 HDFS 加载出来之后放到缓存。可想而知，大量数据进入缓存必将其他实时业务热点数据挤出，其他业务不得不从 HDFS 加载，进而会造成明显的读延迟毛刺

优化建议：离线批量读取请求设置禁用缓存，scan.setBlockCache(false)

###### HBase 服务器端优化

一般服务端端问题一旦导致业务读请求延迟较大的话，通常是集群级别的，即整个集群的业务都会反映读延迟较大。可以从 4 个方面入手：

**5. 读请求是否均衡？**

优化原理：极端情况下假如所有的读请求都落在一台 RegionServer 的某几个 Region 上，这一方面不能发挥整个集群的并发处理能力，另一方面势必造成此台 RegionServer 资源严重消耗（比如 IO 耗尽、handler 耗尽等），落在该台 RegionServer 上的其他业务会因此受到很大的波及。可见，读请求不均衡不仅会造成本身业务性能很差，还会严重影响其他业务。当然，写请求不均衡也会造成类似的问题，可见负载不均衡是 HBase 的大忌。

观察确认：观察所有 RegionServer 的读请求 QPS 曲线，确认是否存在读请求不均衡现象

优化建议：RowKey 必须进行散列化处理（比如 MD5 散列），同时建表必须进行预分区处理

**6. BlockCache 是否设置合理？**

优化原理：BlockCache 作为读缓存，对于读性能来说至关重要。默认情况下 BlockCache 和 Memstore 的配置相对比较均衡（各占 40%），可以根据集群业务进行修正，比如读多写少业务可以将 BlockCache 占比调大。另一方面，BlockCache 的策略选择也很重要，不同策略对读性能来说影响并不是很大，但是对 GC 的影响却相当显著，尤其 BucketCache 的 offheap 模式下 GC 表现很优越。另外，HBase 2.0 对 offheap 的改造（HBASE-11425）将会使 HBase 的读性能得到 2～4 倍的提升，同时 GC 表现会更好！

观察确认：观察所有 RegionServer 的缓存未命中率、配置文件相关配置项一级 GC 日志，确认 BlockCache 是否可以优化

优化建议：JVM 内存配置量 &lt; 20G，BlockCache 策略选择 LRUBlockCache；否则选择 BucketCache 策略的 offheap 模式；期待 HBase 2.0 的到来！

**7. HFile 文件是否太多？**

优化原理：HBase 读取数据通常首先会到 Memstore 和 BlockCache 中检索（读取最近写入数据 & 热点数据），如果查找不到就会到文件中检索。HBase 的类 LSM 结构会导致每个 store 包含多数 HFile 文件，文件越多，检索所需的 IO 次数必然越多，读取延迟也就越高。文件数量通常取决于 Compaction 的执行策略，一般和两个配置参数有关：hbase.hstore.compactionThreshold 和 hbase.hstore.compaction.max.size，前者表示一个 store 中的文件数超过多少就应该进行合并，后者表示参数合并的文件大小最大是多少，超过此大小的文件不能参与合并。这两个参数不能设置太’松’（前者不能设置太大，后者不能设置太小），导致 Compaction 合并文件的实际效果不明显，进而很多文件得不到合并。这样就会导致 HFile 文件数变多。

观察确认：观察 RegionServer 级别以及 Region 级别的 storefile 数，确认 HFile 文件是否过多

优化建议：hbase.hstore.compactionThreshold 设置不能太大，默认是 3 个；设置需要根据 Region 大小确定，通常可以简单的认为 hbase.hstore.compaction.max.size = RegionSize / hbase.hstore.compactionThreshold

**8. Compaction 是否消耗系统资源过多？**

优化原理：Compaction 是将小文件合并为大文件，提高后续业务随机读性能，但是也会带来 IO 放大以及带宽消耗问题（数据远程读取以及三副本写入都会消耗系统带宽）。正常配置情况下 Minor Compaction 并不会带来很大的系统资源消耗，除非因为配置不合理导致 Minor Compaction 太过频繁，或者 Region 设置太大情况下发生 Major Compaction。

观察确认：观察系统 IO 资源以及带宽资源使用情况，再观察 Compaction 队列长度，确认是否由于 Compaction 导致系统资源消耗过多

优化建议：

（1）Minor Compaction 设置：hbase.hstore.compactionThreshold 设置不能太小，又不能设置太大，因此建议设置为 5～6；hbase.hstore.compaction.max.size = RegionSize / hbase.hstore.compactionThreshold

（2）Major Compaction 设置：大 Region 读延迟敏感业务（ 100G 以上）通常不建议开启自动 Major Compaction，手动低峰期触发。小 Region 或者延迟不敏感业务可以开启 Major Compaction，但建议限制流量；

（3）期待更多的优秀 Compaction 策略，类似于 stripe-compaction 尽早提供稳定服务

###### HBase 列族设计优化

HBase 列族设计对读性能影响也至关重要，其特点是只影响单个业务，并不会对整个集群产生太大影响。列族设计主要从两个方面检查：

**9. Bloomfilter 是否设置？是否设置合理？**

优化原理：Bloomfilter 主要用来过滤不存在待检索 RowKey 或者 Row-Col 的 HFile 文件，避免无用的 IO 操作。它会告诉你在这个 HFile 文件中是否可能存在待检索的 KV，如果不存在，就可以不用消耗 IO 打开文件进行 seek。很显然，通过设置 Bloomfilter 可以提升随机读写的性能。

Bloomfilter 取值有两个，row 以及 rowcol，需要根据业务来确定具体使用哪种。如果业务大多数随机查询仅仅使用 row 作为查询条件，Bloomfilter 一定要设置为 row，否则如果大多数随机查询使用 row+cf 作为查询条件，Bloomfilter 需要设置为 rowcol。如果不确定业务查询类型，设置为 row。

优化建议：任何业务都应该设置 Bloomfilter，通常设置为 row 就可以，除非确认业务随机查询类型为 row+cf，可以设置为 rowcol

###### HDFS 相关优化

HDFS 作为 HBase 最终数据存储系统，通常会使用三副本策略存储 HBase 数据文件以及日志文件。从 HDFS 的角度望上层看，HBase 即是它的客户端，HBase 通过调用它的客户端进行数据读写操作，因此 HDFS 的相关优化也会影响 HBase 的读写性能。这里主要关注如下三个方面：

**10. Short-Circuit Local Read 功能是否开启？**

优化原理：当前 HDFS 读取数据都需要经过 DataNode，客户端会向 DataNode 发送读取数据的请求，DataNode 接受到请求之后从硬盘中将文件读出来，再通过 TPC 发送给客户端。Short Circuit 策略允许客户端绕过 DataNode 直接读取本地数据。（具体原理参考此处）

优化建议：开启 Short Circuit Local Read 功能。

**11. Hedged Read 功能是否开启？**

优化原理：HBase 数据在 HDFS 中一般都会存储三份，而且优先会通过 Short-Circuit Local Read 功能尝试本地读。但是在某些特殊情况下，有可能会出现因为磁盘问题或者网络问题引起的短时间本地读取失败，为了应对这类问题，社区开发者提出了补偿重试机制 – Hedged Read。该机制基本工作原理为：客户端发起一个本地读，一旦一段时间之后还没有返回，客户端将会向其他 DataNode 发送相同数据的请求。哪一个请求先返回，另一个就会被丢弃。

优化建议：开启 Hedged Read 功能。

**12. 数据本地率是否太低？**

数据本地率：HDFS 数据通常存储三份，假如当前 RegionA 处于 Node1 上，数据 a 写入的时候三副本为 (Node1,Node2,Node3)，数据 b 写入三副本是 (Node1,Node4,Node5)，数据 c 写入三副本 (Node1,Node3,Node5)，可以看出来所有数据写入本地 Node1 肯定会写一份，数据都在本地可以读到，因此数据本地率是 100%。现在假设 RegionA 被迁移到了 Node2 上，只有数据 a 在该节点上，其他数据（b 和 c）读取只能远程跨节点读，本地率就为 33%（假设 a，b 和 c 的数据大小相同）。

优化原理：数据本地率太低很显然会产生大量的跨网络 IO 请求，必然会导致读请求延迟较高，因此提高数据本地率可以有效优化随机读性能。数据本地率低的原因一般是因为 Region 迁移（自动 balance 开启、RegionServer 宕机迁移、手动迁移等）, 因此一方面可以通过避免 Region 无故迁移来保持数据本地率，另一方面如果数据本地率很低，也可以通过执行 major_compact 提升数据本地率到 100%。

优化建议：避免 Region 无故迁移，比如关闭自动 balance、RS 宕机及时拉起并迁回飘走的 Region 等；在业务低峰期执行 major_compact 提升数据本地率

###### HBase 读性能优化归纳

读延迟较大无非三种常见的表象，单个业务慢、集群随机读慢以及某个业务随机读之后其他业务受到影响导致随机读延迟很大。了解完常见的可能导致读延迟较大的一些问题之后，我们将这些问题进行如下归类，大家可以在看到现象之后在对应的问题列表中进行具体定位：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWKYq31uwSA7yWYg9zyOdDn83ialOeLBGCUzRVOtsQSZReEKq7G8hxWcg/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWOIth6UoPZY6qlFJibDWsD50bMR7ysRCEfGnQnZJh9xIAmCflfiaQCEZw/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWzfuGicx25BKhl5Pq48EI4S6Z2lWgffJcmAxUBeqpI977O8owcCsQXog/640?wx_fmt=png)

#### 写优化

和读相比，HBase 写数据流程倒是显得很简单：数据先顺序写入 HLog，再写入对应的缓存 Memstore，当 Memstore 中数据大小达到一定阈值（128M）之后，系统会异步将 Memstore 中数据 flush 到 HDFS 形成小文件。HBase 数据写入通常会遇到两类问题，一类是写性能较差，另一类是数据根本写不进去。这两类问题的切入点也不尽相同，如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWsKKPMjKoexRrMUaD6dMwGonicyX84cIBlpVmGL5VibEMJBf0amMpgazA/640?wx_fmt=png)

**1. 是否需要写 WAL？WAL 是否需要同步写入？**

优化原理：数据写入流程可以理解为一次顺序写 WAL + 一次写缓存，通常情况下写缓存延迟很低，因此提升写性能就只能从 WAL 入手。WAL 机制一方面是为了确保数据即使写入缓存丢失也可以恢复，另一方面是为了集群之间异步复制。默认 WAL 机制开启且使用同步机制写入 WAL。首先考虑业务是否需要写 WAL，通常情况下大多数业务都会开启 WAL 机制（默认），但是对于部分业务可能并不特别关心异常情况下部分数据的丢失，而更关心数据写入吞吐量，比如某些推荐业务，这类业务即使丢失一部分用户行为数据可能对推荐结果并不构成很大影响，但是对于写入吞吐量要求很高，不能造成数据队列阻塞。这种场景下可以考虑关闭 WAL 写入，写入吞吐量可以提升 2x~3x。退而求其次，有些业务不能接受不写 WAL，但可以接受 WAL 异步写入，也是可以考虑优化的，通常也会带来 1x～2x 的性能提升。

优化推荐：根据业务关注点在 WAL 机制与写入吞吐量之间做出选择

其他注意点：对于使用 Increment 操作的业务，WAL 可以设置关闭，也可以设置异步写入，方法同 Put 类似。相信大多数 Increment 操作业务对 WAL 可能都不是那么敏感～

**2. Put 是否可以同步批量提交？**

优化原理：HBase 分别提供了单条 put 以及批量 put 的 API 接口，使用批量 put 接口可以减少客户端到 RegionServer 之间的 RPC 连接数，提高写入性能。另外需要注意的是，批量 put 请求要么全部成功返回，要么抛出异常。

优化建议：使用批量 put 进行写入请求

**3. Put 是否可以异步批量提交？**

优化原理：业务如果可以接受异常情况下少量数据丢失的话，还可以使用异步批量提交的方式提交请求。提交分为两阶段执行：用户提交写请求之后，数据会写入客户端缓存，并返回用户写入成功；当客户端缓存达到阈值（默认 2M）之后批量提交给 RegionServer。需要注意的是，在某些情况下客户端异常的情况下缓存数据有可能丢失。

优化建议：在业务可以接受的情况下开启异步批量提交

使用方式：setAutoFlush(false)

**4. Region 是否太少？**

优化原理：当前集群中表的 Region 个数如果小于 RegionServer 个数，即 Num(Region of Table) &lt; Num(RegionServer)，可以考虑切分 Region 并尽可能分布到不同 RegionServer 来提高系统请求并发度，如果 Num(Region of Table) > Num(RegionServer)，再增加 Region 个数效果并不明显。

优化建议：在 Num(Region of Table) &lt; Num(RegionServer) 的场景下切分部分请求负载高的 Region 并迁移到其他 RegionServer；

**5. 写入请求是否不均衡？**

优化原理：另一个需要考虑的问题是写入请求是否均衡，如果不均衡，一方面会导致系统并发度较低，另一方面也有可能造成部分节点负载很高，进而影响其他业务。分布式系统中特别害怕一个节点负载很高的情况，一个节点负载很高可能会拖慢整个集群，这是因为很多业务会使用 Mutli 批量提交读写请求，一旦其中一部分请求落到该节点无法得到及时响应，就会导致整个批量请求超时。因此不怕节点宕掉，就怕节点奄奄一息！

优化建议：检查 RowKey 设计以及预分区策略，保证写入请求均衡。

**6. 写入 KeyValue 数据是否太大？**

KeyValue 大小对写入性能的影响巨大，一旦遇到写入性能比较差的情况，需要考虑是否由于写入 KeyValue 数据太大导致。KeyValue 大小对写入性能影响曲线图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWiasLTn9UTOI8oZwUJ9RFdOhzygRy7FNADykKCH5Wt7Nibcjc24k5OJFw/640?wx_fmt=png)

图中横坐标是写入的一行数据（每行数据 10 列）大小，左纵坐标是写入吞吐量，右坐标是写入平均延迟（ms）。可以看出随着单行数据大小不断变大，写入吞吐量急剧下降，写入延迟在 100K 之后急剧增大。

说到这里，有必要和大家分享两起在生产线环境因为业务 KeyValue 较大导致的严重问题，一起是因为大字段业务写入导致其他业务吞吐量急剧下降，另一起是因为大字段业务 scan 导致 RegionServer 宕机。

**写异常问题检查点**

上述几点主要针对写性能优化进行了介绍，除此之外，在一些情况下还会出现写异常，一旦发生需要考虑下面两种情况（GC 引起的不做介绍）：Memstore 设置是否会触发 Region 级别或者 RegionServer 级别 flush 操作？

问题解析：以 RegionServer 级别 flush 进行解析，HBase 设定一旦整个 RegionServer 上所有 Memstore 占用内存大小总和大于配置文件中 upperlimit 时，系统就会执行 RegionServer 级别 flush，flush 算法会首先按照 Region 大小进行排序，再按照该顺序依次进行 flush，直至总 Memstore 大小低至 lowerlimit。这种 flush 通常会 block 较长时间，在日志中会发现 “Memstore is above high water mark and block 7452 ms”，表示这次 flush 将会阻塞 7s 左右。

问题检查点：

Region 规模与 Memstore 总大小设置是否合理？如果 RegionServer 上 Region 较多，而 Memstore 总大小设置的很小（JVM 设置较小或者 upper.limit 设置较小），就会触发 RegionServer 级别 flush。

列族是否设置过多，通常情况下表列族建议设置在 1～3 个之间，最好一个。如果设置过多，会导致一个 Region 中包含很多 Memstore，导致更容易触到高水位 upperlimit。

Store 中 HFile 数量是否大于配置参数 blockingStoreFile?

问题解析：对于数据写入很快的集群，还需要特别关注一个参数：hbase.hstore.blockingStoreFiles，此参数表示如果当前 hstore 中文件数大于该值，系统将会强制执行 compaction 操作进行文件合并，合并的过程会阻塞整个 hstore 的写入。通常情况下该场景发生在数据写入很快的情况下，在日志中可以发现”Waited 3722ms on a compaction to clean up ‘too many store files“

问题检查点：参数设置是否合理？hbase.hstore.compactionThreshold 表示启动 compaction 的最低阈值，该值不能太大，否则会积累太多文件，一般建议设置为 5～8 左右。hbase.hstore.blockingStoreFiles 默认设置为 7，可以适当调大一些。

#### 其他

最后我们讲一讲一些容易忽视的优化点：

###### 使用压缩

HBase 支持大量的压缩算法，可以从列簇级别上进行压缩。因为 CPU 压缩和解压缩消耗的时间往往比从磁盘读取和写入数据要快得多，所以使用压缩通常会带来很可观的性能提升

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2P4occQ1DKx9zmLF4sH7LRWuvHhWgjKToH16RUrMcTibAZFweLkme1Ut3e5fj98hvqEmvAUObuH6bQ/640?wx_fmt=png)

大家可以自行百度关于压缩算法在 HBase 中的集成安装，HBase 包含一个能够测试压缩设置是否正常的工具，我们可以输入：

./bin/hbase org.apache.hadoop.hbase.util.CompresstionTest  

进行检测。

然后启用检查，因为即使测试工具报告成功了，由于 JNI 需要先安装好本地库，如果缺失这一步将会在添加新服务器的时候出现问题，导致新的服务器使用本地库打开含有压缩列族的 region 失败。

我们可以在服务器启动的时候检查压缩库是否已经正确安装，如果没有则不会启动服务器：

<property>  
    <name>hbase.regionserver.codecs</name>  
    <value>snappy,lzo</value>  
</property>

这样一来 region 服务器在启动的时候将会检查 Snappy 和 LZO 压缩库是否已经正确安装

我们可以通过 shell 创建表的时候指定列族的压缩格式：

> create 'testtable',{NAME => 'colfam1',COMPRESSION => 'GZ'}

需要注意的是，如果用户要更改一个已经存在的表的压缩格式，要先将该表 disable 才能修改之后再 enable 重新上线。并且更改后的 region 只有在刷写存储文件的时候才会使用新的压缩格式，没有刷写之前保持原样，用户可以通过 shell 的 major_compact 来强制格式重写，但是此操作会占用大量资源。

###### 使用扫描缓存

如果 HBase 作为一个 MapReduce 作业的而输入源，最好将 MapReduce 作业的输入扫描器实例的缓存用 setCaching() 设置为比 1 大的多的值。例如设置为 500 的时候则一次可以传送 500 行数据到客户端进行处理。

###### 限定扫描范围

如果只处理少数列，则应当只有这些列被添加到 Scan 的输入中，因为如果没有做筛选，则会扫描其他的数据存储文件。

###### 关闭 ResultScanner

这不会带来性能提升，但是会避免一些性能问题。所以一定要在 try/catch 中关闭 ResultScanner。