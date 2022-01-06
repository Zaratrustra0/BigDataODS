# 数据分析工具篇——HQL原理及优化
HQL 是数据分析过程中的必备技能，随着数据量增加，这一技能越来越重要，熟练应用的同时会带来效率的问题，动辄十几亿的数据量如果处理不完善的话有可能导致一个作业运行几个小时，更严重的还有可能因占用过多资源而引发生产问题，所以 HQL 优化就变得非常重要，本文我们就深入 HQL 的原理中，探索 HQL 优化的方法和逻辑。  

**group by\*\***的计算原理 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

**代码为：** 

```sql
SELECT uid, SUM(COUNT) FROM logs GROUP BY uid;
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1xsjfCicuegGgpeicQeFt4j30Pd2VXgjrSReWbiao5X8bWbOWHMRczy6icw/640?wx_fmt=png)

可以看到，group by 本身不是全局变量，任务会被分到各个 map 中进行分组，然后再在 reduce 中聚合。

默认设置了 hive.map.aggr=true，所以会在 mapper 端先 group by 一次，最后再把结果 merge 起来，为了减少 reducer 处理的数据量。注意看 explain 的 mode 是不一样的。mapper 是 hash，reducer 是 mergepartial。如果把 hive.map.aggr=false，那将 groupby 放到 reducer 才做，他的 mode 是 complete。

**优化点：** 

Group by 主要是面对数据倾斜的问题。

很多聚合操作可以现在 map 端进行，最后在 Reduce 端完成结果输出：

```javascript
Set hive.map.aggr = true；  # 是否在Map端进行聚合，默认为true；
Set hive.groupby.mapaggr.checkinterval = 1000000；  # 在Map端进行聚合操作的条目数目；
```

当使用 Group by 有数据倾斜的时候进行负载均衡：

```javascript
Set hive.groupby.skewindata = true； # hive自动进行负载均衡；
```

策略就是把 MR 任务拆分成两个 MR Job：第一个先做预汇总，第二个再做最终汇总；

**第一个 Job：** 

Map 输出结果集中缓存到 maptask 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同 Group by Key 有可能被分到不同的 reduce 中，从而达到负载均衡的目的；

**第二个 Job：** 

根据第一阶段处理的数据结果按照 group by key 分布到 reduce 中，保证相同的 group by key 分布到同一个 reduce 中，最后完成最终的聚合操作。

**join\*\***的优化原理 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

**代码为：** 

```sql
SELECT a.id,a.dept,b.age FROM a join b ON (a.id = b.id);
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1wbQQJlGicibMO9bAcMS0MIDneSkbjRjOlp0d5ahkZHW1rspAr9I7U6iaA/640?wx_fmt=png)

**1\*\***）Map 阶段：\*\* 

读取源表的数据，Map 输出时候以 Join on 条件中的列为 key，如果 Join 有多个关联键，则以这些关联键的组合作为 key；

Map 输出的 value 为 join 之后所关心的 (select 或者 where 中需要用到的) 列；同时在 value 中还会包含表的 Tag 信息，用于标明此 value 对应哪个表；

按照 key 进行排序；

**2\*\***）\***\*Shuffle 阶段：** 

根据 key 的值进行 hash, 并将 key/value 按照 hash 值推送至不同的 reduce 中，这样确保两个表中相同的 key 位于同一个 reduce 中。

**3\*\***）Reduce 阶段：\*\* 

根据 key 的值完成 join 操作，期间通过 Tag 来识别不同表中的数据。

在多表 join 关联时：

如果 **Join 的 key 相同**，不管有多少个表，都会合并为一个 **Map-Reduce**，例如：

```sql
SELECT pv.pageid, u.age 
FROM page_view p 
JOIN user u 
ON (pv.userid = u.userid) 
JOIN newuser x 
ON (u.userid = x.userid);
```

如果 **Join** 的 **key**不同，**Map-Reduce** 的任务数目和 **Join** 操作的数目是对应的，例如：

```sql
SELECT pv.pageid, u.age 
FROM page_view p 
JOIN user u 
ON (pv.userid = u.userid) 
JOIN newuser x 
on (u.age = x.age);
```

**优化点：**   

1）应该将条目少的表 / 子查询放在 **Join** 操作符的左边。

2）我们知道文件数目小，容易在文件存储端造成瓶颈，给 **HDFS** 带来压力，影响处理效率。对此，可以通过合并**Map**和**Reduce**的结果文件来消除这样的影响。用于设置合并属性的参数有：

```javascript
合并Map输出文件：hive.merge.mapfiles=true（默认值为真）
合并Reduce端输出文件：hive.merge.mapredfiles=false（默认值为假）
合并文件的大小：hive.merge.size.per.task=256*1000*1000（默认值为 256000000）
```

3） **Common join**即普通的**join**，性能较差，因为涉及到了**shuffle**的过程（**Hadoop/spark**开发的过程中，有一个原则：能避免不使用**shuffle**就不使用**shuffle**），可以转化成**map join**。

```swift
hive.auto.convert.join=true；# 表示将运算转化成map join方式
```

使用的前提条件是需要的数据在 **Map** 的过程中可以访问到。  

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1H1PZQ8Gp77bpibSlXUP1BT27539HicNVJibkrYH555OIIuaS2U3hkr24w/640?wx_fmt=png)

**1\*\***）启动 \***\*Task A**：**Task A**去启动一个**MapReduce**的**local task**；通过该**local task**把 small table data 的数据读取进来；之后会生成一个 HashTable Files；之后将该文件加载到分布式缓存（Distributed Cache）中来；

**2\*\***）启动 \*\*MapJoin Task：去读大表的数据，每读一个就会去和 Distributed Cache 中的数据去关联一次，关联上后进行输出。

整个阶段，没有 reduce 和 shuffle，问题在于如果小表过大，可能会出现 OOM。

**Union\*\***与 union all 优化原理 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

union 将多个结果集合并为一个结果集，结果集去重。代码为：

```sql
select id,name 
from t1 
union 
select id,name 
from t2 
union 
select id,name 
from t3
```

   对应的运行逻辑为：

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1ay6KQwfXAdrUgYPpPWHvkmcdAoia9XsBnZx2e7FOmQP3A84dw5zo1Pg/640?wx_fmt=png)

union all 将多个结果集合并为一个结果集，结果集不去重。使用时多与 group by 结合使用，代码为：

```sql
select all.id, all.name 
from(
   select id,name 
   from t1 
   union all 
   select id,name 
   from t2 
   union all 
   select id,name 
   from t3
)all 
group by all.id ,all.name
```

对应的运行逻辑为：

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1LnibcjvCS0EUTMxTRecB9pK8kufVm3fXJLUtDYVPibjic23MqmAZGwoQA/640?wx_fmt=png)

从上面的两个逻辑图可以看到，第二种写法性能要好。union 写法每两份数据都要先合并去重一次，再和另一份数据合并去重，会产生较多次的 reduce。第二种写法直接将所有数据合并再一次性去重。

对 union all 的操作除了与 group by 结合使用还有一些细节需要注意：

1）对 union all 优化只局限于非嵌套查询。

原代码：job 有 3 个：

```sql
SELECT * 
FROM
(
  SELECT * 
  FROM t1 
  GROUP BY c1,c2,c3 
  UNION ALL 
  SELECT * 
  FROM t2 
  GROUP BY c1,c2,c3
)t3
GROUP BY c1,c2,c3
```

这样的结构是不对的，应该修改为：job 有 1 个：

```sql
SELECT * 
FROM 
(
  SELECT * 
  FROM t1 
  UNION ALL 
  SELECT * 
  FROM t2
)t3 
GROUP BY c1,c2,c3
```

这样的修改可以减少 job 数量，进而提高效率。

2）语句中出现 count(distinct …) 结构时：

原代码为：

```sql
SELECT * 
FROM
(
  SELECT * FROM t1
  UNION ALL
  SELECT c1,c2,c3,COUNT(DISTINCT c4) 
  FROM t2 GROUP BY c1,c2,c3
) t3
GROUP BY c1,c2,c3;
```

修改为：(采用临时表消灭 COUNT(DISTINCT) 作业不但能解决倾斜问题，还能有效减少 jobs)。

```sql
INSERT t4 SELECT c1,c2,c3,c4 FROM t2 GROUP BY c1,c2,c3;

SELECT c1,c2,c3,SUM(income),SUM(uv) FROM
(
SELECT c1,c2,c3,income,0 AS uv FROM t1
UNION ALL
SELECT c1,c2,c3,0 AS income,1 AS uv FROM t2
) t3
GROUP BY c1,c2,c3;
```

**Order by\*\***的优化原理 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

如果指定了 hive.mapred.mode=strict（默认值是 nonstrict）, 这时就必须指定 limit 来限制输出条数，原因是：所有的数据都会在同一个 reducer 端进行，数据量大的情况下可能不能出结果，那么在这样的严格模式下，必须指定输出的条数。

所以数据量大的时候能不用 order by 就不用，可以使用 sort by 结合 distribute by 来进行实现。

sort by 是局部排序；

distribute by 是控制 map 怎么划分 reducer。

```cs
cluster by=distribute by + sort by
```

被 distribute by 设定的字段为 KEY，数据会被 HASH 分发到不同的 reducer 机器上，然后 sort by 会对同一个 reducer 机器上的每组数据进行局部排序。

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1p65k7Wo9Ol9rbFqjiaB3auIaSk9QHUVFia5sYzOm231bYdqfIYjaicdKQ/640?wx_fmt=png)

例如：

```sql
select mid, money, name 
from store 
cluster by mid

select mid, money, name 
from store 
distribute by mid 
sort by mid
```

如果需要获得与上面的中语句一样的效果：

```sql
select mid, money, name 
from store 
cluster by mid 
sort by money
```

注意被 cluster by 指定的列只能是降序，不能指定 asc 和 desc。

不过即使是先 distribute by 然后 sort by 这样的操作，如果某个分组数据太大也会超出 reduce 节点的存储限制，常常会出现 137 内存溢出的错误，对大数据量的排序都是应该避免的。

**Count\*\***（distinct …）优化 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

如下的 sql 会存在性能问题：

```sql
SELECT COUNT( DISTINCT id ) FROM TABLE_NAME WHERE ...;
```

主要原因是 COUNT 这种 “全聚合(full aggregates)” 计算时，它会忽略用户指定的 Reduce Task 数，而强制使用 1，这会导致最终 Map 的全部输出由单个的 ReduceTask 处理。这唯一的 Reduce Task 需要 Shuffle 大量的数据，并且进行排序聚合等处理，这使得它成为整个作业的 IO 和运算瓶颈。

图形如下：

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia18Gj37AkFK01jLu5niaiaibC1geZncVkdSDEhERiaEuf0W5aUgiaO6qf9smg/640?wx_fmt=png)

为了避免这一结构，我们采用嵌套的方式优化 sql：

```sql
SELECT COUNT(*) 
FROM (
  SELECT DISTINCT id FROM TABLE_NAME WHERE … 
) t;
```

这一结构会将任务切分成两个，第一个任务借用多个 reduce 实现 distinct 去重并进行初步 count 计算，然后再将计算结果输出到第二个任务中进行计数。

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1RlykwJicH9SuLPWEO9g79mwnVckTibdINxXLyOnzPzZ0vQPV0cibhDfHg/640?wx_fmt=png)

另外，再有的方法就是用 group by() 嵌套代替 count(distinct a)。

如果能用 group by 的就尽量使用 group by，因为 group by 性能比 distinct 更好。

**HiveSQL\*\***细节优化 \*\*

![](https://mmbiz.qpic.cn/mmbiz_gif/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia1oxWJgzCHE8RibDTMmNIbRyMvqiahbPHm0kjIibl3Mt1yoFUxtoXdwhzlQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple30xsE8EVMY9uEIibiaRIZMia16uxJzzEupicbSMrFXtLug3dQTQqGNEA3RhFWSu4CjqKECcib5kxbroCw/640?wx_fmt=png)

1） 设置合理的 mapreduce 的 task 数，能有效提升性能。

```swift
set mapred.reduce.tasks=n
```

2） 在 sql 中 or 的用法需要加括号，否则可能引起无分区限制：

```sql
from t 
where ds=d1 
and (province=’gd’ or province=’gx’)
```

3） 对运算结果进行压缩：

```sql
set hive.exec.compress.output=true;
```

4） 减少生成的 mapreduce 步骤:

4.1）使用 CASE…WHEN… 代替子查询；

4.2）尽量尽早地过滤数据，减少每个阶段的数据量, 对于分区表要加分区，同时只选择需要使用到的字段；

5） 在 map 阶段读取数据前，FileInputFormat 会将输入文件分割成 split。split 的个数决定了 map 的个数。

```css
mapreduce.input.fileinputformat.split.minsize 默认值 0
mapreduce.input.fileinputformat.split.maxsize 默认值 Integer.MAX_VALUE
dfs.blockSize 默认值 128M，所以在默认情况下 map的数量=block数
```

6） 常用的参数：

```python
hive.exec.reducers.bytes.per.reducer=1000000；
```

设置每个 reduce 处理的数据量，reduce 个数 = map 端输出数据总量 / 参数；

```sql
set hive.mapred.mode=nonstrict;
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set mapred.job.name=p_${v_date};
set mapred.job.priority=HIGH;
set hive.groupby.skewindata=true;
set hive.merge.mapredfiles=true;
set hive.exec.compress.output=true;
set mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
set mapred.output.compression.type=BLOCK;
set mapreduce.map.memory.mb=4096;
set mapreduce.reduce.memory.mb=4096;
set hive.hadoop.supports.splittable.combineinputformat=true;
set mapred.max.split.size=16000000;
set mapred.min.split.size.per.node=16000000;
set mapred.min.split.size.per.rack=16000000;
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
set hive.exec.reducers.bytes.per.reducer=128000000;
```

7）设置 map 个数：  
  map 个数和来源表文件压缩格式有关，.gz 格式的压缩文件无法切分，每个文件会生成一个 map；

```swift
set hive.hadoop.supports.splittable.combineinputformat=true; 只有这个参数打开，下面的3个参数才能生效
set mapred.max.split.size=16000000; 每个map负载；
set mapred.min.split.size.per.node=100000000; 每个节点map的最小负载，这个值必须小于set mapred.max.split.size的值；
set mapred.min.split.size.per.rack=100000000; 每个机架map的最小负载；
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

**欢迎大家关注公众号：** 

![](https://mmbiz.qpic.cn/mmbiz_jpg/MzF2oGuple2eOwBBXgW7h33zk7oo27XqW1H9xcrztjw8SKM5HlvyFuxWcvbJCpHUibeGBEfiaJn4u14ibGpCd8T5g/640?wx_fmt=jpeg)

来都来了，点个关注再走呗～ 
 [https://mp.weixin.qq.com/s/xK0hddk8BV25yEEs86TZmg](https://mp.weixin.qq.com/s/xK0hddk8BV25yEEs86TZmg)
