# 再次分享！Hive调优，数据工程师成神之路
[![](https://mmbiz.qpic.cn/mmbiz_jpg/1OYP1AZw0W2zy3AvMTZoXvRQBugxEjD3OvYoaJj7ZP5icOvmlMdIravHk4YibYO9V73ia4GonJuGiavHfhvttpBqgw/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493670&idx=2&sn=8e91e0a20da1930cd61ff807f5153e7c&chksm=cf37da2bf840533d1cc6fdadc94978e2d367db002676b1832915b947e7a402aa3018ecbf0780&scene=21#wechat_redirect)

热文回顾：[美团外卖离线数仓建设与实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493670&idx=2&sn=8e91e0a20da1930cd61ff807f5153e7c&chksm=cf37da2bf840533d1cc6fdadc94978e2d367db002676b1832915b947e7a402aa3018ecbf0780&scene=21#wechat_redirect)

**1**

**前言**

       毫不夸张的说，有没有掌握 hive 调优，是判断一个数据工程师是否合格的重要指标 

       hive 调优涉及到压缩和存储调优，参数调优，sql 的调优，数据倾斜调优，小文件问题的调优等

**2**

**数据的压缩与存储格式**

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLf2NkDmRBWy0iaL7ib4TL1s6LHq1ZuCwrictwyAMB7Yy59Qyyw3RNsztwmxvJC42sUqb3evH8QR7ylmA/640?wx_fmt=png)

1. map 阶段输出数据压缩 ，在这个阶段，优先选择一个低 CPU 开销的算法。

```cpp
set hive.exec.compress.intermediate=true
set mapred.map.output.compression.codec= org.apache.hadoop.io.compress.SnappyCodec
set mapred.map.output.compression.codec=com.hadoop.compression.lzo.LzoCodec;
```

2. 对最终输出结果压缩

```bash
set hive.exec.compress.output=true 
set mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec
 
## 当然，也可以在hive建表时指定表的文件格式和压缩编码
```

结论，一般选择 orcfile/parquet + snappy 方式

**3**

**合理利用分区分 \*\***桶 \*\*

      分区是将表的数据在物理上分成不同的文件夹，以便于在查询时可以精准指定所要读取的分区目录，从来降低读取的数据量

分桶是将表数据按指定列的 hash 散列后分在了不同的文件中，将来查询时，hive 可以根据分桶结构，快速定位到一行数据所在的分桶文件，从来提高读取效率

**4**

**hive 参数优化**

```swift
// 让可以不走mapreduce任务的，就不走mapreduce任务
hive> set hive.fetch.task.conversion=more;
 
// 开启任务并行执行
 set hive.exec.parallel=true;
// 解释：当一个sql中有多个job时候，且这多个job之间没有依赖，则可以让顺序执行变为并行执行（一般为用到union all的时候）
 
 // 同一个sql允许并行任务的最大线程数 
set hive.exec.parallel.thread.number=8;
 
// 设置jvm重用
// JVM重用对hive的性能具有非常大的 影响，特别是对于很难避免小文件的场景或者task特别多的场景，这类场景大多数执行时间都很短。jvm的启动过程可能会造成相当大的开销，尤其是执行的job包含有成千上万个task任务的情况。
set mapred.job.reuse.jvm.num.tasks=10; 
 
// 合理设置reduce的数目
// 方法1：调整每个reduce所接受的数据量大小
set hive.exec.reducers.bytes.per.reducer=500000000; （500M）
// 方法2：直接设置reduce数量
set mapred.reduce.tasks = 20
// map端聚合，降低传给reduce的数据量
set hive.map.aggr=true  
set hive.groupby.skewindata=true
```

**5**

**sql 优化**

1

where 条件优化

优化前（关系数据库不用考虑会自动优化）

```sql
select m.cid,u.id from order m join customer u on( m.cid =u.id )where m.dt='20180808';
```

优化后 (where 条件在 map 端执行而不是在 reduce 端执行）

```sql
select m.cid,u.id from （select * from order where dt='20180818'） m join customer u on( m.cid =u.id);
```

2

union 优化

尽量不要使用 union （union 去掉重复的记录）而是使用 union all 然后在用 group by 去重

3

count distinct 优化

不要使用 count (distinct  cloumn) , 使用子查询

```sql
select count(1) from (select id from tablename group by id) tmp;
```

4

用 in 来代替 join

如果需要根据一个表的字段来约束另为一个表，尽量用 in 来代替 join . in 要比 join 快

```sql
select id,name from tb1  a join tb2 b on(a.id = b.id);
 
select id,name from tb1 where id in(select id from tb2);
```

5

优化子查询

消灭子查询内的 group by 、 COUNT(DISTINCT)，MAX，MIN。可以减少 job 的数量。

6

join 优化

Common/shuffle/Reduce JOIN 连接发生的阶段，发生在 reduce 阶段， 适用于大表 连接 大表 (默认的方式)

Map join ：连接发生在 map 阶段 ， 适用于小表 连接 大表  
                       大表的数据从文件中读取  
                       小表的数据存放在内存中（hive 中已经自动进行了优化，自动判断小表，然后进行缓存）

```cs
set hive.auto.convert.join=true;
```

SMB join  
   Sort -Merge -Bucket Join  对大表连接大表的优化，用桶表的概念来进行优化。在一个桶内发生笛卡尔积连接（需要是两个桶表进行 join）

```sql
 set hive.auto.convert.sortmerge.join=true;  
 set hive.optimize.bucketmapjoin = true;  
 set hive.optimize.bucketmapjoin.sortedmerge = true;  
set hive.auto.convert.sortmerge.join.noconditionaltask=true;
```

**6**

**数据倾斜**

表现：任务进度长时间维持在 99%（或 100%），查看任务监控页面，发现只有少量（1 个或几个）reduce 子任务未完成。因为其处理的数据量和其他 reduce 差异过大。

原因：某个 reduce 的数据输入量远远大于其他 reduce 数据的输入量

1

sql 本身导致的倾斜

1）group by

如果是在 group by 中产生了数据倾斜，是否可以讲 group by 的维度变得更细，如果没法变得更细，就可以在原分组 key 上添加随机数后分组聚合一次，然后对结果去掉随机数后再分组聚合

在 join 时，有大量为 null 的 join key，则可以将 null 转成随机值，避免聚集

2）count(distinct)

情形：某特殊值过多

后果：处理此特殊值的 reduce 耗时；只有一个 reduce 任务

解决方式：count distinct 时，将值为空的情况单独处理，比如可以直接过滤空值的行，

在最后结果中加 1。如果还有其他计算，需要进行 group by，可以先将值为空的记录单独处理，再和其他计算结果进行 union。

3）不同数据类型关联产生数据倾斜

情形：比如用户表中 user_id 字段为 int，log 表中 user_id 字段既有 string 类型也有 int 类型。当按照 user_id 进行两个表的 Join 操作时。

后果：处理此特殊值的 reduce 耗时；只有一个 reduce 任务

默认的 Hash 操作会按 int 型的 id 来进行分配，这样会导致所有 string 类型 id 的记录都分配

到一个 Reducer 中。

解决方式：把数字类型转换成字符串类型

select \* from users a

left outer join logs b

on a.usr_id = cast(b.user_id as string)

4）mapjoin

2

业务数据本身的特性 (存在热点 key)

join 的每路输入都比较大，且长尾是热点值导致的，可以对热点值和非热点值分别进行处理，再合并数据

3

key 本身分布不均

可以在 key 上加随机数，或者增加 reduceTask 数量  

开启数据倾斜时负载均衡  

set hive.groupby.skewindata=true;

思想：就是先随机分发并处理，再按照 key group by 来分发处理。

操作：当选项设定为 true，生成的查询计划会有两个 MRJob。

第一个 MRJob 中，Map 的输出结果集合会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 GroupBy Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；

第二个 MRJob 再根据预处理的数据结果按照 GroupBy Key 分布到 Reduce 中（这个过程可以保证相同的原始 GroupBy Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

4

控制空值分布

将为空的 key 转变为字符串加随机数或纯随机数，将因空值而造成倾斜的数据分不到多个 Reducer。

注：对于异常值如果不需要的话，最好是提前在 where 条件里过滤掉，这样可以使计算量大大减少

**7**

**合并小文件**

小文件的产生有三个地方，map 输入，map 输出，reduce 输出，小文件过多也会影响 hive 的分析效率：

设置 map 输入的小文件合并

```swift
set mapred.max.split.size=256000000;  
//一个节点上split的至少的大小(这个值决定了多个DataNode上的文件是否需要合并)
set mapred.min.split.size.per.node=100000000;
//一个交换机下split的至少的大小(这个值决定了多个交换机上的文件是否需要合并)  
set mapred.min.split.size.per.rack=100000000;
//执行Map前进行小文件合并
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

设置 map 输出和 reduce 输出进行合并的相关参数：

```cs
//设置map端输出进行合并，默认为true
set hive.merge.mapfiles = true
//设置reduce端输出进行合并，默认为false
set hive.merge.mapredfiles = true
//设置合并文件的大小
set hive.merge.size.per.task = 256*1000*1000
//当输出文件的平均大小小于该值时，启动一个独立的MapReduce任务进行文件merge。
set hive.merge.smallfiles.avgsize=16000000
```

**8**

**查看 sql 的执行计划**

```sql
explain sql 
```

学会查看 sql 的执行计划，优化业务逻辑 ，减少 job 的数据量。对调优也非常重要！  

![](https://mmbiz.qpic.cn/mmbiz_gif/dXCnejTRMLfaPKxaibsZ7cCiaozWvibvo25R8yoqsvvTmiaG2PAMap0daZ8F31icEtXsianE5bA67SQ5Lh1fAbT9yLvA/640?wx_fmt=gif)

往期推荐

\[

美团外卖离线数仓建设与实践

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493670&idx=2&sn=8e91e0a20da1930cd61ff807f5153e7c&chksm=cf37da2bf840533d1cc6fdadc94978e2d367db002676b1832915b947e7a402aa3018ecbf0780&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493670&idx=2&sn=8e91e0a20da1930cd61ff807f5153e7c&chksm=cf37da2bf840533d1cc6fdadc94978e2d367db002676b1832915b947e7a402aa3018ecbf0780&scene=21#wechat_redirect))

\[

深度 | 传统数仓和大数据数仓的区别是什么？

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493661&idx=1&sn=52a30be740211b2175fa287a0b25dd95&chksm=cf37da10f8405306757030c4a391f5a38c431d3457c4dbaed418c078ae5afd526fdd4a2af8dd&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493661&idx=1&sn=52a30be740211b2175fa287a0b25dd95&chksm=cf37da10f8405306757030c4a391f5a38c431d3457c4dbaed418c078ae5afd526fdd4a2af8dd&scene=21#wechat_redirect))

\[

数据湖 VS 数据仓库之争？阿里提出大数据架构新概念：湖仓一体！

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493644&idx=2&sn=ffeb7a413f8366caa18fbf58d3aafc25&chksm=cf37da01f84053175b482531a7664bdcb0167941ea0aa201a700a3446051458bf8e5a08041c5&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493644&idx=2&sn=ffeb7a413f8366caa18fbf58d3aafc25&chksm=cf37da01f84053175b482531a7664bdcb0167941ea0aa201a700a3446051458bf8e5a08041c5&scene=21#wechat_redirect))

\[

深度 | 一文带你了解 Hadoop 大数据原理与架构（文末赠书）

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493622&idx=1&sn=88a5ccb0fcd04bcc39b6c29f1ad1b071&chksm=cf37d5fbf8405ced0e405066d36e6b068ac804a90d9324db138af866fb79b2fe0478e24e83f2&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493622&idx=1&sn=88a5ccb0fcd04bcc39b6c29f1ad1b071&chksm=cf37d5fbf8405ced0e405066d36e6b068ac804a90d9324db138af866fb79b2fe0478e24e83f2&scene=21#wechat_redirect))

\[

详解数据仓库的实施步骤

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493558&idx=2&sn=2e091d6d90ca857459f80bf18e30a781&chksm=cf37d5bbf8405cadf9ce154e5bd61d02c39b62080372763a6d3bf8d86e68f39969a177de008d&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493558&idx=2&sn=2e091d6d90ca857459f80bf18e30a781&chksm=cf37d5bbf8405cadf9ce154e5bd61d02c39b62080372763a6d3bf8d86e68f39969a177de008d&scene=21#wechat_redirect))

\[

进阶 ｜Hive 复杂数据类型

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg3NjIyNjQwMg==&mid=2247493543&idx=2&sn=319599c8acf531bd6e0f558e7a75b02d&chksm=cf37d5aaf8405cbc2fd52ed6f38bbe7cc9489572df57a0c88a15f1ec44b324b9524fd1642659&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493543&idx=2&sn=319599c8acf531bd6e0f558e7a75b02d&chksm=cf37d5aaf8405cbc2fd52ed6f38bbe7cc9489572df57a0c88a15f1ec44b324b9524fd1642659&scene=21#wechat_redirect))

**欢迎点赞 + 收藏 + 转发朋友圈素质三连**

看完本文有收获？请转发分享给更多人

**大数据爱好者社区**

![](https://mmbiz.qpic.cn/mmbiz_jpg/pDnsbKziazM5Uiaia0AN0Pn31CC3BTDEpKcIX9udwhjnRMBoBNib8OiaCtxKKgShIhyMrwIiaCF5UTEEE2tmDzuH36yQ/640?wx_fmt=jpeg)

\***\* 文章不错？\*\***点个【在看】吧！**\*\***👇**\*\*** 
 [https://mp.weixin.qq.com/s/OsT2Sgjn47HbhVyRau2vOw](https://mp.weixin.qq.com/s/OsT2Sgjn47HbhVyRau2vOw)
