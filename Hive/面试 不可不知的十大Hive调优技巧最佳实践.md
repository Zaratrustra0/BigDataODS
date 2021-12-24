# 面试|不可不知的十大Hive调优技巧最佳实践
Apache Hive 是建立在 Apache Hadoop 之上的数据仓库软件项目，用于提供数据查询和分析。Hive 是 Hadoop 在 HDFS 上的 SQL 接口，它提供了类似于 SQL 的接口来查询存储在与 Hadoop 集成的各种数据库和文件系统中的数据。可以说从事数据开发工作，无论是在平时的工作中，还是在面试中，Hive 具有举足轻重的地位，尤其是 Hive 的性能调优方面，不仅能够在工作中提升效率而且还可以在面试中脱颖而出。在本文中，我将分享十个性能优化技术，全文如下。

默认情况下，Hive 会执行多次表扫描。因此，如果要在某张 hive 表中执行多个操作，建议使用一次扫描并使用该扫描来执行多个操作。

比如将一张表的数据多次查询出来装载到另外一张表中。如下面的示例，表 my_table 是一个分区表，分区字段为 dt，如果需要在表中查询 2 个特定的分区日期数据，并将记录装载到 2 个不同的表中。

`INSERT INTO temp_table_20201115 SELECT * FROM my_table WHERE dt ='2020-11-15';  
INSERT INTO temp_table_20201116 SELECT * FROM my_table WHERE dt ='2020-11-16';  
`

在以上查询中，Hive 将扫描表 2 次，为了避免这种情况，我们可以使用下面的方式：

`FROM my_table  
INSERT INTO temp_table_20201115 SELECT * WHERE dt ='2020-11-15'  
INSERT INTO temp_table_20201116 SELECT * WHERE dt ='2020-11-16'  
`

这样可以确保只对 my_table 表执行一次扫描，从而可以大大减少执行的时间和资源。

对于一张比较大的表，将其设计成分区表可以提升查询的性能，对于一个特定分区的查询，只会加载对应分区路径的文件数据，因此，当用户使用特定分区列值执行选择查询时，将仅针对该特定分区执行查询，由于将针对较少的数据量进行扫描，所以可以提供更好的性能。值得注意的是，分区字段的选择是影响查询性能的重要因素，尽量避免层级较深的分区，这样会造成太多的子文件夹。

现在问题来了，该使用哪些列进行分区呢？一条基本的法则是：选择低基数属性作为 “分区键”，比如“地区” 或“日期”等。

一些常见的分区字段可以是：

-   日期或者时间

比如 year、month、day 或者 hour，当表中存在时间或者日期字段时，可以使用些字段。

-   地理位置

比如国家、省份、城市等

-   业务逻辑

比如部门、销售区域、客户等等

`CREATE TABLE table_name (  
    col1 data_type,  
    col2 data_type)  
PARTITIONED BY (partition1 data_type, partition2 data_type,….);  
`

通常，当很难在列上创建分区时，我们会使用分桶，比如某个经常被筛选的字段，如果将其作为分区字段，会造成大量的分区。在 Hive 中，会对分桶字段进行哈希，从而提供了中额外的数据结构，进行提升查询效率。

与分区表类似，分桶表的组织方式是将 HDFS 上的文件分割成多个文件。分桶可以加快数据采样，也可以提升 join 的性能 (join 的字段是分桶字段)，因为分桶可以确保某个 key 对应的数据在一个特定的桶内 (文件)，所以巧妙地选择分桶字段可以大幅度提升 join 的性能。通常情况下，分桶字段可以选择经常用在过滤操作或者 join 操作的字段。

我们可以使用 set.hive.enforce.bucketing = true 启用分桶设置。

当使用分桶表时，最好将 bucketmapjoin 标志设置为 true，具体配置参数为：

SET hive.optimize.bucketmapjoin = true

`CREATE TABLE table_name   
PARTITIONED BY (partition1 data_type, partition2 data_type,….) CLUSTERED BY (column_name1, column_name2, …)   
SORTED BY (column_name [ASC|DESC], …)]   
INTO num_buckets BUCKETS;  
`

复杂的 Hive 查询通常会转换为一系列多阶段的 MapReduce 作业，并且这些作业将由 Hive 引擎链接起来以完成整个查询。因此，此处的 “中间输出” 是指上一个 MapReduce 作业的输出，它将用作下一个 MapReduce 作业的输入数据。

压缩可以显著减少中间数据量，从而在内部减少了 Map 和 Reduce 之间的数据传输量。

我们可以使用以下属性在中间输出上启用压缩。

`set hive.exec.compress.intermediate=true;  
set hive.intermediate.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;  
set hive.intermediate.compression.type=BLOCK;  
`

为了将最终输出到 HDFS 的数据进行压缩，可以使用以下属性：

`set hive.exec.compress.output=true;  
`

下面是一些可以使用的压缩编解码器

`org.apache.hadoop.io.compress.DefaultCodec  
org.apache.hadoop.io.compress.GzipCodec  
org.apache.hadoop.io.compress.BZip2Codec  
com.hadoop.compression.lzo.LzopCodec  
org.apache.hadoop.io.compress.Lz4Codec  
org.apache.hadoop.io.compress.SnappyCodec  
`

map 端 join 适用于当一张表很小 (可以存在内存中) 的情况，即可以将小表加载至内存。Hive 从 0.7 开始支持自动转为 map 端 join，具体配置如下：

`SET hive.auto.convert.join=true; --  hivev0.11.0 之后默认 true  
SET hive.mapjoin.smalltable.filesize=600000000; -- 默认 25m  
SET hive.auto.convert.join.noconditionaltask=true; -- 默认 true，所以不需要指定 map join hint  
SET hive.auto.convert.join.noconditionaltask.size=10000000; -- 控制加载到内存的表的大小  
`

一旦开启 map 端 join 配置，Hive 会自动检查小表是否大于`hive.mapjoin.smalltable.filesize`配置的大小，如果大于则转为普通的 join，如果小于则转为 map 端 join。

关于 map 端 join 的原理，如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsgnzUcvuDib2oWyG2AQf5B5aDS2icThNcshpTR7ecpVcOOicuw3HBjSiapbx0WlFGqhbKBvfXImxTrq5Q/640?wx_fmt=png)

首先，Task A(客户端本地执行的 task) 负责读取小表 a，并将其转成一个 HashTable 的数据结构，写入到本地文件，之后将其加载至分布式缓存。

然后，Task B 任务会启动 map 任务读取大表 b，在 Map 阶段，根据每条记录与分布式缓存中的 a 表对应的 hashtable 关联，并输出结果

注意：map 端 join 没有 reduce 任务，所以 map 直接输出结果，即有多少个 map 任务就会产生多少个结果文件。

Hive 中的向量化查询执行大大减少了典型查询操作（如扫描，过滤器，聚合和连接）的 CPU 使用率。

标准查询执行系统一次处理一行，在处理下一行之前，单行数据会被查询中的所有运算符进行处理，导致 CPU 使用效率非常低。在向量化查询执行中，数据行被批处理在一起（默认 => 1024 行），表示为一组列向量。

要使用向量化查询执行，必须以 ORC 格式（CDH 5）存储数据，并设置以下变量。

`SET hive.vectorized.execution.enabled=true  
`

在 CDH 6 中默认启用 Hive 查询向量化，启用查询向量化后，还可以设置其他属性来调整查询向量化的方式，具体可以参考 cloudera 官网。

默认生成的执行计划会在可见的位置执行过滤器，但在某些情况下，某些过滤器表达式可以被推到更接近首次看到此特定数据的运算符的位置。

比如下面的查询：

`select  
    a.*,  
    b.*   
from   
    a join b on (a.col1 = b.col1)  
where a.col1 > 15 and b.col2 > 16  
`

如果没有谓词下推，则在完成 JOIN 处理之后将执行过滤条件**(a.col1> 15 和 b.col2> 16)**。因此，在这种情况下，JOIN 将首先发生，并且可能产生更多的行，然后在进行过滤操作。

使用谓词下推，这两个谓词**(a.col1> 15 和 b.col2> 16)**将在 JOIN 之前被处理，因此它可能会从 a 和 b 中过滤掉连接中较早处理的大部分数据行，因此，建议启用谓词下推。

通过将 hive.optimize.ppd 设置为 true 可以启用谓词下推。

`SET hive.optimize.ppd=true  
`

Hive 支持 TEXTFILE, SEQUENCEFILE, AVRO, RCFILE, ORC, 以及 PARQUET 文件格式，可以通过两种方式指定表的文件格式：

-   CREATE TABLE … STORE AS : 即在建表时指定文件格式，默认是 TEXTFILE
-   ALTER TABLE … \[PARTITION partition_spec] SET FILEFORMAT : 修改具体表的文件格式

如果未指定文件存储格式，则默认使用的是参数 hive.default.fileformat 设定的格式。

如果数据存储在小于块大小的小文件中，则可以使用 SEQUENCE 文件格式。如果要以减少存储空间并提高性能的优化方式存储数据，则可以使用 ORC 文件格式，而当列中嵌套的数据过多时，Parquet 格式会很有用。因此，需要根据拥有的数据确定输入文件格式。

如果要查询分区的 Hive 表，但不提供分区谓词（分区列条件），则在这种情况下，将针对该表的所有分区发出查询，这可能会非常耗时且占用资源。因此，我们将下面的属性定义为 strict，以指示在分区表上未提供分区谓词的情况下编译器将引发错误。

`SET hive.partition.pruning=strict  
`

Hive 在提交最终执行之前会优化每个查询的逻辑和物理执行计划。基于成本的优化会根据查询成本进行进一步的优化，从而可能产生不同的决策：比如如何决定 JOIN 的顺序，执行哪种类型的 JOIN 以及并行度等。

可以通过设置以下参数来启用基于成本的优化。

`set hive.cbo.enable=true;  
set hive.compute.query.using.stats=true;  
set hive.stats.fetch.column.stats=true;  
set hive.stats.fetch.partition.stats=true;  
`

可以使用统计信息来优化查询以提高性能。基于成本的优化器（CBO）还使用统计信息来比较查询计划并选择最佳计划。通过查看统计信息而不是运行查询，效率会很高。

收集表的列统计信息：

`ANALYZE TABLE mytable COMPUTE STATISTICS FOR COLUMNS;  
`

查看 my_db 数据库中 my_table 中 my_id 列的列统计信息：

`DESCRIBE FORMATTED my_db.my_table my_id  
`

本文主要分享了 10 个 Hive 优化的基本技巧，希望能够为你优化 Hive 查询提供一个基本的思路。再次感谢你的阅读，希望本文对你有所帮助。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXeFzuCNYd7EAPsScZvZ6KhcWvvLhicX9SVRibv8pY7MtyrEYicyrfXfae9E6OQbUtIUr4bicpKBJSFNmA/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/bYBd8tdsnwuUzuPoMV4LwQ](https://mp.weixin.qq.com/s/bYBd8tdsnwuUzuPoMV4LwQ)
