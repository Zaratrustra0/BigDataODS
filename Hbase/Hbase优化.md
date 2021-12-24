# Hbase优化
本文对 hbase 集群进行优化，主要涵盖硬件和操作系统，网络通信，JVM，查询，写入，核心服务，配置参数，zookeeper，表设计等多方面。  

我们对 hbase 的应用主要是用户画像，根据自身使用场景做一些优化。难免有片面之处。

**一、软硬件优化：** 

1. 配置内存，cpu

HBase 的 LSM 树结构，缓存机制和日志机制对内存消耗非常大，所以内存越大越好。

其中过滤器，数据压缩，多条件组合扫描等场景都是 cpu 密集型的，所以 cpu 也要够强悍

2. 操作系统

选择主流 linux 发行版，JVM 推荐用 Sun HotSpot64 位的，能发挥 hadoop 最好的性能

使用 noatime 挂载磁盘：一般数据库的挂载磁盘没有特殊要求情况下最好都设置位为 noatime 以提高性能

关闭系统交换区: Linux 内存反复交换会影响 JVM 性能，典型的异常就是导致 zookeeper 超时。所以设置 vm.swappiness 设置的比较低较好。

3.  网络通信

由于 hdfs 对集群网络吞吐有很高的要求，所以网络必须保证低延迟高吞吐

添加机架感知：机架感知是提升 hadoop 的写入和读取本地化。在 core-site.xml 中配置 topology.script.file.name

4. JVM 优化

根据网络上很多成熟引用验证比较优秀的垃圾回收器搭配组合 CMS+ParNew

**二、进入主题：Hbase 本身优化**

1. Hbase 查询优化：

a. 设置 scan 缓存：scan 的时候 setCaching 来设置缓存大小

b. 确定所需要的列：scan 时候 addColumn 来添加所需要的列减少数据的传输

c. 如果批量进行全表扫描请禁用块缓存，因为全表扫描每条记录只读取一遍

d. 优化行键查询：全表 scan 时，如果只需要行键，可以使用过滤器来减少服务器返回的数据量。

e. 通过 HBaseTool 访问：HTable 对象对于客户端读写数据来说不是线程安全的，多线程时要为每个线程创建一个 HBase 对象。而 HBaseTool 链接线程池机制可以解决线程安全问题，同事维持一定数量的 HBase

f. 使用批量读：HTable.get(List<Get>)

g. 使用 Coprocessor 统计行数: 具体原理请看协处理器原理

h. 缓存查询结果：对于查询频繁的应用场景

2. HBase 写入优化：

a. 关闭 WAL 日志：如果能容忍一定的数据丢失风险，则可以关闭 WAL

b. 设置 AutoFlush: 关闭此功能等 put 到达到缓存阀值时候才提交到服务器

c. 预创建 Region: 预先创建 region 来避免写入时 region 到达一定阀值而 split 影响性能，和 mongodb 预分片原理一致

d. 延迟 WAL flush：如果开启 WAL 则可以将 WAL flush 到磁盘的时间间隔调大一些来提高性能

e. 使用批量写 HTable.put(List<Put>)

3. HBase 基本核心服务优化

a. 优化分裂操作： 如果写多读少的场景则可以调高 hbase.hregion.max.filesize 来减少 region 分裂

b. 优化合并操作：大合并非常消耗资源，且合并时候会阻塞写操作。应该在集群不繁忙的时候进行大合并

4. Hbase 配置参数优化：

a. 设置 regionserver handler 数量：如果写请求比较多则可以适当调高 hbase.regionserver.handler.count 的数量以提高写吞吐。此参数调高很消耗内存，请注意。

b. 调整 blockCache 大小：hfile.block.cache.size 来设置 regionserver 查询的内存设置。默认 0.25 指读缓存占用堆内存 25%。读场景比较多可以适当调高。

c. 设置 MemStore 的上下限：hbase.regionserver.global.memstore.upperLimit 表示 regionserver 上所有 region 的 Memstore 的大小上限，超过上限会引发全局 flush，这个参数主要防止 regionserver 内存占用过大被 OOM Kill 掉。

读为主的集群中，可以调小此参数，调高 blockCache; 写则相反

d. 调整影响合并的文件数：hbase.hstore.blockingStoreFiles 值用于控制超过此值的 storefile 则会出发合并。可以调大此值减少合并次数

e. 调整 MemStore 的 flush 因子：当 Memstore 占用内存大小超过 hbase.hregion.memstore.flush.size 倍数时将阻塞 region 所有请求，出发 flush，释放内存。如果正常不会出现写入或写入数据量突然增大则可以保持默认，否则要调高此值。

f. 调整单个文件大小：hbase.hregion.max.filesize 用于定义单个 hstorefile 大小，超过此值则引发 region 文件 split。 Region 比较小则合并和 split 都很快，当然会造成集群响应时间波动。 大合并和 split 则造成较长时间阻塞。应该根据自己场景来定义

5. 分布式协调系统 zookeeper 的优化：zookeeper 的优化方法也很多，我就主要讲 hbase 优化。只是说明下 zookeeper 优化也非常重要。

6. 表设计优化

a. 开启布隆过滤器：布隆过滤器可以减少读盘次数以降低延迟。原理和 redis 的 hyperloglog 一样 (我们以前有用此功能对用户数量进行估算)

b. 调整列族块大小：较小的块大小可以提高随机读的速度，同时导致块索引变大。

c. 设置 in memory 属性：对于经常访问的列族可以设置 in memory，但是要考虑消耗内存的问题

d. 调整列族最大版本数量：数量大占用磁盘空间，且导致集群变大。根据自己应用场景来选择。像我们做画像由于要统计用户场景变化，所以版本数量有根据自己需求设置

e. 设置 TTL 属性：超过 TTL 的列将自动删除。这个也根据自己场景选择。我们做用户画像时会将某些用户行为超过时间的就认为没有必要在进行存储分析了，所以可以设置 TTL 来自动删除

7. 关闭 mapreduce 的预测执行功能：若使用 mapreduce 来访问 hbase 集群应该关闭，否则有可能导致 hbase 客户端链接数陡增影响集群运行

8. 修改负载均衡执行周期：当集群写入频繁时，可以调小，否则可以调大。 
 [https://mp.weixin.qq.com/s/IL8IHhTOtBBxhBuvGyshxg](https://mp.weixin.qq.com/s/IL8IHhTOtBBxhBuvGyshxg)
