# Hive常用参数调优十二板斧
1. limit 限制调整  

一般情况下，Limit 语句还是需要执行整个查询语句，然后再返回部分结果。

有一个配置属性可以开启，避免这种情况 --- 对数据源进行抽样。

hive.limit.optimize.enable=true --- 开启对数据源进行采样的功能 hive.limit.row.max.size --- 设置最小的采样容量 hive.limit.optimize.limit.file --- 设置最大的采样样本数

缺点：有可能部分数据永远不会被处理到

#### 2. JOIN 优化

##### 1) 将大表放后头

Hive 假定查询中最后的一个表是大表。它会将其它表缓存起来，然后扫描最后那个表。因此通常需要将小表放前面，或者标记哪张表是大表：/\*streamtable(table_name) \*/

##### 2). 使用相同的连接键

当对 3 个或者更多个表进行 join 连接时，如果每个 on 子句都使用相同的连接键的话，那么只会产生一个 MapReduce job。

##### 3). 尽量尽早地过滤数据

减少每个阶段的数据量, 对于分区表要加分区，同时只选择需要使用到的字段。

##### 4). 尽量原子化操作

尽量避免一个 SQL 包含复杂逻辑，可以使用中间表来完成复杂的逻辑

#### 3. 本地模式

有时 hive 的输入数据量是非常小的。在这种情况下，为查询出发执行任务的时间消耗可能会比实际 job 的执行时间要多的多。对于大多数这种情况，hive 可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间会明显被缩短

> set hive.exec.mode.local.auto=true;

当一个 job 满足如下条件才能真正使用本地模式：　　- 1.job 的输入数据大小必须小于参数：hive.exec.mode.local.auto.inputbytes.max(默认 128MB) 　　- 2.job 的 map 数必须小于参数：hive.exec.mode.local.auto.tasks.max(默认 4) 　　- 3.job 的 reduce 数必须为 0 或者 1

可用参数 hive.mapred.local.mem(默认 0) 控制 child jvm 使用的最大内存数。

#### 4. 并行执行

hive 会将一个查询转化为一个或多个阶段，包括：MapReduce 阶段、抽样阶段、合并阶段、limit 阶段等。默认情况下，一次只执行一个阶段。不过，如果某些阶段不是互相依赖，是可以并行执行的。

set hive.exec.parallel=true, 可以开启并发执行。

set hive.exec.parallel.thread.number=16; // 同一个 sql 允许最大并行度，默认为 8。

会比较耗系统资源。

#### 5.strict 模式

对分区表进行查询，在 where 子句中没有加分区过滤的话，将禁止提交任务 (默认：nonstrict)

> set hive.mapred.mode=strict;

注：使用严格模式可以禁止 3 种类型的查询：（1）对于分区表，不加分区字段过滤条件，不能执行 （2）对于 order by 语句，必须使用 limit 语句 （3）限制笛卡尔积的查询（join 的时候不使用 on，而使用 where 的）

#### 6. 调整 mapper 和 reducer 个数

##### Map 阶段优化

map 执行时间：map 任务启动和初始化的时间 + 逻辑处理的时间。

1\. 通常情况下，作业会通过 input 的目录产生一个或者多个 map 任务。主要的决定因素有：input 的文件总个数，input 的文件大小，集群设置的文件块大小 (目前为 128M, 可在 hive 中通过 set dfs.block.size; 命令查看到，该参数不能自定义修改)；

2\. 举例：

a) 假设 input 目录下有 1 个文件 a, 大小为 780M, 那么 hadoop 会将该文件 a 分隔成 7 个块（6 个 128m 的块和 1 个 12m 的块），从而产生 7 个 map 数 b) 假设 input 目录下有 3 个文件 a,b,c, 大小分别为 10m，20m，130m，那么 hadoop 会分隔成 4 个块（10m,20m,128m,2m）, 从而产生 4 个 map 数 即，如果文件大于块大小 (128m), 那么会拆分，如果小于块大小，则把该文件当成一个块。

3\. 是不是 map 数越多越好？

答案是否定的。如果一个任务有很多小文件（远远小于块大小 128m）, 则每个小文件也会被当做一个块，用一个 map 任务来完成，而一个 map 任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的 map 数是受限的。

4\. 是不是保证每个 map 处理接近 128m 的文件块，就高枕无忧了？

答案也是不一定。比如有一个 127m 的文件，正常会用一个 map 去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果 map 处理的逻辑比较复杂，用一个 map 任务去做，肯定也比较耗时。

针对上面的问题 3 和 4，我们需要采取两种方式来解决：即减少 map 数和增加 map 数；如何合并小文件，减少 map 数？

假设一个 SQL 任务：Select count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' 该任务的 inputdir /group/p_sdo_data/p_sdo_data_etl/pt/popt_tbaccountcopy_mes/pt=2012-07-04 共有 194 个文件，其中很多是远远小于 128m 的小文件，总大小 9G，正常执行会用 194 个 map 任务。Map 总共消耗的计算资源：SLOTS_MILLIS_MAPS= 623,020 通过以下方法来在 map 执行前合并小文件，减少 map 数：

     set mapred.max.split.size=100000000; set mapred.min.split.size.per.node=100000000; set mapred.min.split.size.per.rack=100000000; set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

再执行上面的语句，用了 74 个 map 任务，map 消耗的计算资源：SLOTS_MILLIS_MAPS=333,500 对于这个简单 SQL 任务，执行时间上可能差不多，但节省了一半的计算资源。大概解释一下，100000000 表示 100M

> set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

这个参数表示执行前进行小文件合并，前面三个参数确定合并文件块的大小，大于文件块大小 128m 的，按照 128m 来分隔，小于 128m, 大于 100m 的，按照 100m 来分隔，把那些小于 100m 的（包括小文件和分隔大文件剩下的），进行合并, 最终生成了 74 个块。

如何适当的增加 map 数？当 input 的文件都很大，任务逻辑复杂，map 执行非常慢的时候，可以考虑增加 Map 数， 来使得每个 map 处理的数据量减少，从而提高任务的执行效率。假设有这样一个任务：

     Select data_desc,  count(1),  count(distinct id),  sum(case when …),  sum(case when ...),  sum(…)from a group by data_desc

如果表 a 只有一个文件，大小为 120M，但包含几千万的记录，如果用 1 个 map 去完成这个任务，肯定是比较耗时的，这种情况下，我们要考虑将这一个文件合理的拆分成多个，这样就可以用多个 map 任务去完成。

       set mapred.reduce.tasks=10;   create table a_1 as    select * from a    distribute by rand(123);

这样会将 a 表的记录，随机的分散到包含 10 个文件的 a_1 表中，再用 a_1 代替上面 sql 中的 a 表，则会用 10 个 map 任务去完成。每个 map 任务处理大于 12M（几百万记录）的数据，效率肯定会好很多。

看上去，貌似这两种有些矛盾，一个是要合并小文件，一个是要把大文件拆成小文件，这点正是重点需要关注的地方，根据实际情况，控制 map 数量需要遵循两个原则：使大数据量利用合适的 map 数；使单个 map 任务处理合适的数据量。

控制 hive 任务的 reduce 数：

1.Hive 自己如何确定 reduce 数：

reduce 个数的设定极大影响任务执行效率，不指定 reduce 个数的情况下，Hive 会猜测确定一个 reduce 个数，基于以下两个设定：hive.exec.reducers.bytes.per.reducer（每个 reduce 任务处理的数据量，默认为 1000^3=1G） hive.exec.reducers.max（每个任务最大的 reduce 数，默认为 999）

计算 reducer 数的公式很简单 N=min(参数 2，总输入数据量 / 参数 1)

即，如果 reduce 的输入（map 的输出）总大小不超过 1G, 那么只会有一个 reduce 任务，如：

select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt;

/group/p_sdo_data/p_sdo_data_etl/pt/popt_tbaccountcopy_mes/pt=2012-07-04 总大小为 9G 多，

因此这句有 10 个 reduce

2\. 调整 reduce 个数方法一：

调整 hive.exec.reducers.bytes.per.reducer 参数的值；set hive.exec.reducers.bytes.per.reducer=500000000; （500M） select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 这次有 20 个 reduce

3\. 调整 reduce 个数方法二

set mapred.reduce.tasks = 15; select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 这次有 15 个 reduce

4.reduce 个数并不是越多越好；

同 map 一样，启动和初始化 reduce 也会消耗时间和资源；另外，有多少个 reduce, 就会有多少个输出文件，如果生成了很多个小文件， 那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

5\. 什么情况下只有一个 reduce；

很多时候你会发现任务中不管数据量多大，不管你有没有设置调整 reduce 个数的参数，任务中一直都只有一个 reduce 任务；其实只有一个 reduce 任务的情况，除了数据量小于 hive.exec.reducers.bytes.per.reducer 参数值的情况外，还有以下原因：

a) 没有 group by 的汇总，比如把 select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 写成 select count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04'; 这点非常常见，希望大家尽量改写。

b) 用了 Order by

c) 有笛卡尔积

通常这些情况下，除了找办法来变通和避免，我们暂时没有什么好的办法，因为这些操作都是全局的，所以 hadoop 不得不用一个 reduce 去完成。同样的，在设置 reduce 个数的时候也需要考虑这两个原则：

-   使大数据量利用合适的 reduce 数
-   使单个 reduce 任务处理合适的数据量

##### Reduce 阶段优化

调整方式：

-   set mapred.reduce.tasks=?
-   set hive.exec.reducers.bytes.per.reducer = ?

一般根据输入文件的总大小, 用它的 estimation 函数来自动计算 reduce 的个数：reduce 个数 = InputFileSize / bytes per reducer

#### 7.JVM 重用

用于避免小文件的场景或者 task 特别多的场景，这类场景大多数执行时间都很短，因为 hive 调起 mapreduce 任务，JVM 的启动过程会造成很大的开销，尤其是 job 有成千上万个 task 任务时，JVM 重用可以使得 JVM 实例在同一个 job 中重新使用 N 次

> set mapred.job.reuse.jvm.num.tasks=10; --10 为重用个数

#### 8. 动态分区调整

动态分区属性：设置为 true 表示开启动态分区功能（默认为 false）

> hive.exec.dynamic.partition=true;

动态分区属性：设置为 nonstrict, 表示允许所有分区都是动态的（默认为 strict） 设置为 strict，表示必须保证至少有一个分区是静态的

> hive.exec.dynamic.partition.mode=strict;

动态分区属性：每个 mapper 或 reducer 可以创建的最大动态分区个数

> hive.exec.max.dynamic.partitions.pernode=100;

动态分区属性：一个动态分区创建语句可以创建的最大动态分区个数

> hive.exec.max.dynamic.partitions=1000;

动态分区属性：全局可以创建的最大文件个数

> hive.exec.max.created.files=100000;

控制 DataNode 一次可以打开的文件个数 这个参数必须设置在 DataNode 的 $HADOOP_HOME/conf/hdfs-site.xml 文件中

    <property>    <name>dfs.datanode.max.xcievers</name>    <value>8192</value></property>

#### 9. 推测执行

目的：是通过加快获取单个 task 的结果以及进行侦测将执行慢的 TaskTracker 加入到黑名单的方式来提高整体的任务执行效率

（1）修改 $HADOOP_HOME/conf/mapred-site.xml 文件

             <property>                   <name>mapred.map.tasks.speculative.execution </name>                   <value>true</value>         </property>         <property>                   <name>mapred.reduce.tasks.speculative.execution </name>                   <value>true</value>         </property>

（2）修改 hive 配置

> set hive.mapred.reduce.tasks.speculative.execution=true;

#### 10. 数据倾斜

表现：任务进度长时间维持在 99%（或 100%），查看任务监控页面，发现只有少量（1 个或几个）reduce 子任务未完成。因为其处理的数据量和其他 reduce 差异过大。单一 reduce 的记录数与平均记录数差异过大，通常可能达到 3 倍甚至更多。最长时长远大于平均时长。

原因

1)、key 分布不均匀

2)、业务数据本身的特性

3)、建表时考虑不周

4)、某些 SQL 语句本身就有数据倾斜

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M1bha9Ujb44VbibK2PN4aDfyCwktxlwL3qn4ibQen2v5SsiaWtDicoIkUmIdY4Rnf3y0goVOyf8NyLAg/640?wx_fmt=png)

解决方案：参数调节

> hive.map.aggr=true

#### 11. 其他参数调优

开启 CLI 提示符前打印出当前所在的数据库名

> set hive.cli.print.current.db=true;

让 CLI 打印出字段名称

> hive.cli.print.header=true;

设置任务名称，方便查找监控

> SET mapred.job.name=P_DWA_D_IA_S_USER_PROD;

决定是否可以在 Map 端进行聚合操作

> set hive.map.aggr=true;

有数据倾斜的时候进行负载均衡

> set hive.groupby.skewindata=true;

对于简单的不需要聚合的类似 SELECT col from table LIMIT n 语句，不需要起 MapReduce job，直接通过 Fetch task 获取数据

> set hive.fetch.task.conversion=more;

#### 12、小文件问题

##### 小文件是如何产生的

1\. 动态分区插入数据，产生大量的小文件，从而导致 map 数量剧增。

2.reduce 数量越多，小文件也越多 (reduce 的个数和输出文件是对应的)。

3\. 数据源本身就包含大量的小文件。

#### 小文件问题的影响

1\. 从 Hive 的角度看，小文件会开很多 map，一个 map 开一个 JVM 去执行，所以这些任务的初始化，启动，执行会浪费大量的资源，严重影响性能。

2\. 在 HDFS 中，每个小文件对象约占 150byte，如果小文件过多会占用大量内存。这样 NameNode 内存容量严重制约了集群的扩展。

#### 小文件问题的解决方案

从小文件产生的途经就可以从源头上控制小文件数量，方法如下：

1\. 使用 Sequencefile 作为表存储格式，不要用 textfile，在一定程度上可以减少小文件

2\. 减少 reduce 的数量 (可以使用参数进行控制)

3\. 少用动态分区，用时记得按 distribute by 分区

对于已有的小文件，我们可以通过以下几种方案解决：

1\. 使用 hadoop archive 命令把小文件进行归档

2\. 重建表，建表时减少 reduce 数量

3\. 通过参数进行调节，设置 map/reduce 端的相关参数，如下：

设置 map 输入合并小文件的相关参数：

    //每个Map最大输入大小(这个值决定了合并后文件的数量)  set mapred.max.split.size=256000000;    //一个节点上split的至少的大小(这个值决定了多个DataNode上的文件是否需要合并)  set mapred.min.split.size.per.node=100000000;  //一个交换机下split的至少的大小(这个值决定了多个交换机上的文件是否需要合并)    set mapred.min.split.size.per.rack=100000000;  //执行Map前进行小文件合并  set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

设置 map 输出和 reduce 输出进行合并的相关参数：

    //设置map端输出进行合并，默认为true  set hive.merge.mapfiles = true  //设置reduce端输出进行合并，默认为false  set hive.merge.mapredfiles = true  //设置合并文件的大小  set hive.merge.size.per.task = 256*1000*1000  //当输出文件的平均大小小于该值时，启动一个独立的MapReduce任务进行文件merge。set hive.merge.smallfiles.avgsize=16000000

设置如下参数取消一些限制 (HIVE 0.7 后没有此限制)：

hive.merge.mapfiles=false

默认值：true 描述：是否合并 Map 的输出文件，也就是把小文件合并成一个 map

hive.merge.mapredfiles=false

默认值：false 描述：是否合并 Reduce 的输出文件，也就是在 Map 输出阶段做一次 reduce 操作，再输出.

> set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

这个参数表示执行前进行小文件合并，

前面三个参数确定合并文件块的大小，大于文件块大小 128m 的，按照 128m 来分隔，小于 128m, 大于 100m 的，按照 100m 来分隔，把那些小于 100m 的（包括小文件和分隔大文件剩下的），进行合并, 最终生成了 74 个块。 
 [https://mp.weixin.qq.com/s/xUtYr1x5R767CC7lLsE4Ng](https://mp.weixin.qq.com/s/xUtYr1x5R767CC7lLsE4Ng)
