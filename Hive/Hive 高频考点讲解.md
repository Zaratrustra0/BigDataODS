# Hive 高频考点讲解
因为申请较晚，本公众号没留言，想交流的欢迎添加私人微信，一起相互吹捧，共同进步。

Hive 是 FaceBook 开源的一款基于 Hadoop 数据仓库工具，它可以将结构化的数据文件映射为一张表，并提供类 SQL 查询功能。

> The Apache Hive ™ data warehouse software facilitates reading, writing, and managing large datasets residing in distributed storage using SQL 。

### 1.1 Hive 优缺点

##### 1.1.1 优点

1.  操作接口采用类 SQL 语法，提供快速开发的能力（简单、容易上手）。
2.  避免了去写 MapReduce，减少开发人员的学习成本。
3.  Hive 的执行延迟比较高，因此 Hive 常用于数据分析，对实时性要求不高的场合。
4.  Hive 优势在于处理大数据，对于处理小数据没有优势，因为 Hive 的执行延迟比较高。
5.  Hive 支持用户自定义函数，用户可以根据自己的需求来实现自己的函数。

##### 1.1.2 缺点

1.  Hive 的 HQL 表达能力有限，无法表达迭代式算法，不擅长数据挖掘方面。
2.  Hive 的效率比较低，Hive 自动生成的 MapReduce 作业，通常情况下不够智能化。
3.  Hive 查询无法做到跟 MySQL 一样毫秒返回。

### 1.2 Hive 跟 MySQL 比较

##### 1.2.1 对比

Hive 采用了类 SQL 的查询语言 HQL(Hive Query Language)，因此很容易将 Hive 理解为数据库。其实从结构上来看，Hive 和数据库除了拥有类似的查询语言，再无类似之处。本文将从多个方面来阐述 Hive 和数据库的差异。数据库可以用在 Online 的应用中，但是 Hive 是为数据仓库而设计的，清楚这一点，有助于从应用角度理解 Hive 的特性。

| 方向     | Hive         | MySQL   |
| ------ | ------------ | ------- |
| 应用方向   | 数仓           | Online  |
| 查询语言   | HQL          | SQL     |
| 数据存储位置 | HDFS         | 本地文件系统  |
| 数据更新   | 读多写少，无法修改    | 正常 CRUD |
| 索引     | 无索引，暴力查询     | 各种索引    |
| 执行     | 底层 MapReduce | 自己执行引擎  |
| 延迟     | 高延迟          | 低延迟     |
| 可扩展性   | 优秀扩展能力       | 扩展力有限   |
| 数据量    | 超大规模         | 小规模     |

### 1.2.2 Hive 不支持那些

1.  支持等值查询，不支持非等值连接
2.  支持 and 多条件过滤，不支持 or 多条件过滤。
3.  不支持 update 跟 delete。

### 1.3 Hive 底层

Hive 底层是 MapReduce 计算框架，Hive 只是将通读性强且容易编程的 SQL 语句通过 Hive 软件转换成 MapReduce 程序在集群上执行，Hive 可以看做 MapReduce 客户端。操作的数据还是存储在 HDFS 上的，而用户定义的表结构等元信息被存储到 MySQL 上了。以前要写八股文 MapReduce 程序，现在只需要 HQL 查询就可！  

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMAuzhuty98Jl1iaGDM68NrrlgqmAj5W6IicReoD8OEt14GSAaYU0ADEwg/640?wx_fmt=png)

Hive 整体框架

1.  用户接口 Client

> CLI（hive shell）、JDBC/ODBC(java 访问 hive)、WEBUI（浏览器访问 hive）
>
> 2.  元数据 Metastore
> 3.  元数据包括 表名、表所属的数据库（默认是 default）、表的拥有者、列 / 分区字段、表的类型（是否是外部表）、表的数据所在目录等。
> 4.  默认存储在自带的 derby 数据库中 (单客户连接)，推荐使用 MySQL 存储 Metastore。
> 5.  Hadoop  
>     使用 HDFS 进行存储，使用 MapReduce 进行计算。

4.  驱动器 Driver

> 1.  解析器 SQL Parser：将 SQL 字符串转换成**抽象语法树**AST，这一步一般都用第三方工具库完成，比如 antlr；对 AST 进行语法分析，比如表是否存在、字段是否存在、SQL 语义是否有误。
> 2.  编译器 Physical Plan：将 AST 编译生成逻辑执行计划。
> 3.  优化器 Query Optimizer：对逻辑执行计划进行优化。
> 4.  执行器 Execution：把逻辑执行计划转换成可以运行的物理计划。对于 Hive 来说就是 MR/Spark。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMgq06r2r76sFd4HPqgCPFNXQUicibibjMLibh94Xk8mozMMqRlic5gqPnDbg/640?wx_fmt=png)

HQL 执行流程

不要把 Hive 想的多么神秘，你可以用简单的 load 方式将数据加载到创建的表里，也可以直接用 hadoop 指令将数据放入到指定目录，这两种方式都可以直接让你通过 SQL 查询到数据。

### 1.4  HQL 底层执行举例

##### 1.4.1 Join

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMIiaXqLHNnabYYaXsBcGlZDjGnWYrbJL81NI8QSK48ia8AkBSbZ8NEhKw/640?wx_fmt=png)

Join 流程

##### 1.4.2 group by

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMNjF1AEjV3QcLWsjAfzWpU9cjQyIOM8cVZL4lEftr4Ggyz9vCic3zhTg/640?wx_fmt=png)

group by 流程

##### 1.4.3 distinct

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgMZEbMqvODNs1CUb5cr690xFjH4wWA2U9MuzD4OMXElSU1MVyDru5HoQ/640?wx_fmt=png)

distinct 流程

有时想要同时显示聚集前后的数据，这时引入了窗口函数，在 SQL 处理中，窗口函数都是`最后一步执行`，而且仅位于 Order by 字句之前。

### 2.1  数据准备

`name,orderdate,cost  
jack,2017-01-01,10  
tony,2017-01-02,15  
jack,2017-02-03,23  
tony,2017-01-04,29  
jack,2017-01-05,46  
jack,2017-04-06,42  
tony,2017-01-07,50  
jack,2017-01-08,55  
mart,2017-04-08,62  
mart,2017-04-09,68  
neil,2017-05-10,12  
mart,2017-04-11,75  
neil,2017-06-12,80  
mart,2017-04-13,94`

建表 导数据

    create table business(name string, orderdate string,cost int) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';load data local inpath "/opt/module/datas/business.txt" into table business;

### 2.2 用法

相关函数说明

1.  OVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化
2.  CURRENT ROW：当前行
3.  n PRECEDING：往前 n 行数据
4.  n FOLLOWING：往后 n 行数据
5.  UNBOUNDED：起点，UNBOUNDED PRECEDING 表示从前面的起点， UNBOUNDED FOLLOWING 表示到后面的终点

上面写 over 里面，下面写 over 前面。

1.  LAG(col,n)：往前`第` n 行数据
2.  LEAD(col,n)：往后`第` n 行数据
3.  NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从 1 开始，对于每一行，NTILE 返回此行所属的组的编号。注意：n 必须为 int 类型。

### 2.3 开窗函数 demo

1.  查询在 2017 年 4 月份购买过的顾客及总人数

`select name,count(*) over() from business where substring(orderdate,1,7) = '2017-04' group by name;  
结果：  
mart    2  
jack    2  
`

1.  查询顾客的购买明细及月购买总额

`select name,orderdate,cost,sum(cost) over(partition by month(orderdate))   
from business;   
解释：按照月划分数据  然后统计这个月的 cost 总和  
jack    2017-01-01  10  205  
jack    2017-01-08  55  205  
tony    2017-01-07  50  205  
jack    2017-01-05  46  205  
tony    2017-01-04  29  205  
tony    2017-01-02  15  205  
jack    2017-02-03  23  23  
mart    2017-04-13  94  341  
jack    2017-04-06  42  341  
mart    2017-04-11  75  341  
mart    2017-04-09  68  341  
mart    2017-04-08  62  341  
neil    2017-05-10  12  12  
neil    2017-06-12  80  80  
`

1.  查看顾客上次的购买时间

`select name,orderdate,cost,   
lag(orderdate,1,'defaulttime') over(partition by name order by orderdate ) as time1,   
lag(orderdate,2,'defaulttime') over (partition by name order by orderdate) as time2  from business;  
结果 :   
姓名      日期      价格  前一天日期    前两天日期  
jack    2017-01-01  10  defaulttime defaulttime  
jack    2017-01-05  46  2017-01-01  defaulttime  
jack    2017-01-08  55  2017-01-05  2017-01-01  
jack    2017-02-03  23  2017-01-08  2017-01-05  
jack    2017-04-06  42  2017-02-03  2017-01-08  
mart    2017-04-08  62  defaulttime defaulttime  
mart    2017-04-09  68  2017-04-08  defaulttime  
mart    2017-04-11  75  2017-04-09  2017-04-08  
mart    2017-04-13  94  2017-04-11  2017-04-09  
neil    2017-05-10  12  defaulttime defaulttime  
neil    2017-06-12  80  2017-05-10  defaulttime  
tony    2017-01-02  15  defaulttime defaulttime  
tony    2017-01-04  29  2017-01-02  defaulttime  
tony    2017-01-07  50  2017-01-04  2017-01-02  
`

1.  查询前 20% 时间的订单信息

` select * from (select name,orderdate,cost, ntile(5) over(order by orderdate) sorted from business ) t where sorted = 1;  
结果 :   
jack    2017-01-01  10  1  
tony    2017-01-02  15  1  
tony    2017-01-04  29  1  
jack    2017-01-05  46  2  
tony    2017-01-07  50  2  
jack    2017-01-08  55  2  
jack    2017-02-03  23  3  
jack    2017-04-06  42  3  
mart    2017-04-08  62  3  
mart    2017-04-09  68  4  
mart    2017-04-11  75  4  
mart    2017-04-13  94  4  
neil    2017-05-10  12  5  
neil    2017-06-12  80  5  
`

**以下实验均关注最后一列**

1. 所有行相加

`select name,orderdate,cost,sum(cost) over() as sample1 from business;   
结果 :   
mart    2017-04-13  94  661  
neil    2017-06-12  80  661  
mart    2017-04-11  75  661  
neil    2017-05-10  12  661  
mart    2017-04-09  68  661  
`

2. 按 name 分组，组内数据相加

`select name,orderdate,cost,sum(cost) over(partition by name) as sample2  
from business;  
结果 :   
jack    2017-01-05  46  176  
jack    2017-01-08  55  176  
jack    2017-01-01  10  176  
jack    2017-04-06  42  176  
jack    2017-02-03  23  176  
...  
tony    2017-01-04  29  94  
tony    2017-01-02  15  94  
tony    2017-01-07  50  94  
`

3. 按 name 分组，组内数据累加

`select name,orderdate,cost,  
sum(cost) over(partition by name order by orderdate) as sample3  
from business;   
跟下面类似  
select name,orderdate,cost,  
sum(cost) over(distribute by name sort by orderdate) as sample3   
from business;   
jack    2017-01-01  10  10  
jack    2017-01-05  46  56  
jack    2017-01-08  55  111  
jack    2017-02-03  23  134  
jack    2017-04-06  42  176  
...  
`

4. 和 sample3 一样, 由起点到当前行的聚合

`select name,orderdate,cost,  
sum(cost) over(partition by name order by orderdate rows  
between UNBOUNDED PRECEDING and current row ) as sample4   
from business;   
结果 :   
jack    2017-01-01  10  10  
jack    2017-01-05  46  56  
jack    2017-01-08  55  111  
jack    2017-02-03  23  134  
jack    2017-04-06  42  176  
...  
`

5. 当前行和前面一行做聚合

`select name,orderdate,cost,  
sum(cost) over(partition by name order by orderdate rows   
between 1 PRECEDING and current row) as sample5   
from business;  
结果 :   
jack    2017-01-01  10  10  
jack    2017-01-05  46  56 = 46 + 10  
jack    2017-01-08  55  101 = 44 + 46  
jack    2017-02-03  23  78  = 23 + 55  
jack    2017-04-06  42  65  = 42 + 23  
...  
tony    2017-01-02  15  15  
tony    2017-01-04  29  44  
tony    2017-01-07  50  79  
`

6. 当前行和前边一行及后面一行  

`select name,orderdate,cost,  
sum(cost) over(partition by name order by orderdate rows  
between 1 PRECEDING AND 1 FOLLOWING ) as sample6  
from business;  
结果 :   
jack    2017-01-01  10  56  = 10 + 46  
jack    2017-01-05  46  111 = 46 + 10 + 55  
jack    2017-01-08  55  124 = 55 + 46 + 23  
jack    2017-02-03  23  120 = 23 + 55 + 42  
jack    2017-04-06  42  65  = 42 + 23  
...  
tony    2017-01-02  15  44  
tony    2017-01-04  29  94  
tony    2017-01-07  50  79  
`

7. 当前行及后面所有行  

`select name,orderdate,cost,sum(cost) over(partition by name order by orderdate rows between current row and UNBOUNDED FOLLOWING ) as sample7 from business;  
结果 :   
jack    2017-01-01  10  176 = 10 + 46 + 55 + 23 + 42  
jack    2017-01-05  46  166 = 46 + 55 + 23 + 42  
jack    2017-01-08  55  120 = 55 + 23 + 42  
jack    2017-02-03  23  65  =  23 + 42  
jack    2017-04-06  42  42  =  42  
mart    2017-04-08  62  299  
mart    2017-04-09  68  237  
mart    2017-04-11  75  169  
mart    2017-04-13  94  94  
neil    2017-05-10  12  92  
neil    2017-06-12  80  80  
tony    2017-01-02  15  94  
tony    2017-01-04  29  79  
tony    2017-01-07  50  50  
`

### 2.4 Rank

函数说明

> **rank()**：排序相同时会重复，总数不会变  
> **dense_rank()**：排序相同时会重复，总数会减少  
> **row_number()**：会根据顺序计算

`select name,subject,score,  
rank() over(partition by subject order by score desc) rp,  
dense_rank() over(partition by subject order by score desc) drp,  
row_number() over(partition by subject order by score desc) rmp  
from score;  
结果 :   
name   subject score    rp      drp     rmp  
孙悟空   数学    95      1       1       1  
宋宋    数学    86      2       2       2  
婷婷    数学    85      3       3       3  
大海    数学    56      4       4       4  
宋宋    英语    84      1       1       1  
大海    英语    84      1       1       2  
婷婷    英语    78      3（跳过 2）2        3  
孙悟空  英语    68      4       3(总数少)  4  
大海    语文    94      1       1       1  
孙悟空  语文    87      2        2        2  
婷婷    语文    65      3       3       3  
宋宋    语文    64      4       4       4`

#### 2.5 行转列

1.  CONCAT(string A, string B)：

> 返回输入字符串`连接后`的结果，支持任意个输入字符串;
>
> 2.  CONCAT_WS(separator, str1, str2,…)：  
>     特殊形式的 CONCAT()。第一个参数剩余参数间的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将为 NULL。
> 3.  COLLECT_SET(col)：  
>     只接受基本数据类型，主要作用是将某字段的值进行去重汇总，产生 array 类型字段。多行汇总成一个 array 类型。

### 2.6 列转行

1.  EXPLODE(col)：

> 将 hive 一列中复杂的 array 或者 map 结构拆分成多行。
>
> 2.  LATERAL VIEW  
>     用法：LATERAL VIEW udtf(expression) table Alias AS columnAlias  
>     解释：用于和 split, explode 等 UDTF 一起使用，它能够将一列数据拆成多行数据，在此基础上可以对拆分后的数据进行聚合。

### 3.1 MapJoin

如果不指定 MapJoin 或者不符合 MapJoin 的条件，那么 Hive 解析器会将 Join 操作转换成 Common Join，也就是在 Reduce 阶段完成 join。容易发生数据倾斜。可以用 MapJoin 把小表全部加载到内存在 map 端进行 join，避免 reducer 处理。

### 3.2 行列过滤

列处理：在 SELECT 时只拿需要的列，尽量使用分区过滤，少用 SELECT \*。  
行处理：在分区剪裁中，当使用外关联时，如果将副表的过滤条件写在 Where 后面，那么就会先全表关联，之后再过滤。

### 3.3 合理设置 Map 数跟 Reduce 数

##### 3.3.1  map 数不是越多越好

如果一个任务有很多小文件（远远小于块大小 128m），则每个小文件也会被当做一个块，用一个 map 任务来完成，而一个 map 任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费 。而且，同时可执行的 map 数是受限的。此时我们就应该减少 map 数量。

##### 3.3.2 Reduce 数不是越多越好

1.  过多的启动和初始化 Reduce 也会消耗时间和资源；
2.  有多少个 Reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；
3.  Reduce 个数设置考虑这两个原则：处理大数据量利用合适的 Reduce 数；使单个 Reduce 任务处理数据量大小要合适；

### 3.4 严格模式

严格模式 strict 下会有以下特点：

1.  对于分区表，用户不允许扫描所有分区。
2.  使用了 order by 语句的查询，要求必须使用 limit 语句。
3.  限制笛卡尔积的查询。

### 3.5 开启 map 端 combiner

在不影响最终业务逻辑前提下，手动开启 set hive.map.aggr=true；

### 3.6 压缩

设置 map 端输出中间结果压缩，加速网络传输。

### 3.7 小文件进行合并

在 Map 执行前合并小文件，减少 Map 数，**CombineHiveInputFormat** 具有对小文件进行合并的功能（系统默认的格式）。HiveInputFormat 没有对小文件合并功能。

### 3.8 其他

1.  **Fetch 抓取**：指的是 Hive 中对某些情况的查询可以不必使用 MapReduce 计算。
2.  **本地模式**：Hive 可以通过本地模式在单台机器上处理所有的任务。
3.  **数据分区**：数据细化存储方便访问。
4.  **JVM 重用**：JVM 实例在同一个 job 中重新使用 N 次。
5.  **推测执行**：根据一定的法则推测出拖后腿的任务，并为这样的任务启动一个备份任务。
6.  **并行执行**：一个 Hive 查询被分解成多个阶段，阶段之间并非完全互相依赖的。

### 4.1 数据倾斜

##### 4.1.1 定义

数据分布不平衡，某些地方特别多，某些地方又特别少，导致的在处理数据的时候，有些很快就处理完了，而有些又迟迟未能处理完，导致整体任务最终迟迟无法完成，这种现象就是数据倾斜。

##### 4.1.2 产生

1.  key 的分布不均匀或者说某些 key 太集中
2.  业务数据自身的特性，例如不同数据类型关联产生数据倾斜
3.  SQL 语句导致的数据倾斜

##### 4.1.3 解决

1.  不影响最终业务逻辑前提下开启 map 端 combiner。
2.  开启数据倾斜时负载均衡。
3.  手动抽查做好分区规则。
4.  使用 mapjoin，小表进内存 在 Map 端完成 Reduce。

### 4.2 分区表和分桶表对比？

##### 4.2.1 分区表

1.  分区使用的是`表外`字段，需要指定字段类型
2.  分区通过`关键字` **partitioned** by(partition_name string) 声明
3.  分区划分粒度`较粗`
4.  将数据按区域划分开，查询时不用扫描无关的数据，加快查询速度

##### 4.2.2 分桶表

分桶逻辑：对分桶字段求哈希值，用哈希值与分桶的数量取余决定数据放到哪个桶里。

1.  分桶使用的是`表内`字段，已经知道字段类型，不需要再指定。
2.  分桶表通过关键字 **clustered** by(column_name) into … buckets 声明
3.  分桶是`更细`粒度的划分、管理数据，可以对表进行先分区再分桶的划分策略
4.  优点在于用于数据取样时候能够起到优化加速的作用

### 4.3 动态分区

1.  静态分区与动态分区的主要区别在于静态分区是手动指定，而动态分区是通过数据来进行判断。
2.  静态分区的列是在编译时期，通过用户传递来决定的，动态分区只有在 SQL 执行时才能决定。
3.  系统默认开启，非严格模式，动态分区最大值。

### 4.4 Hive 中视图跟索引

##### 4.4.1 视图

视图是一种使用查询语句定义的虚拟表，是数据的一种逻辑结构，创建视图时不会把视图存储到磁盘上，定义视图的查询语句只有在执行视图的语句时才会被执行。视图是只读的，不能向视图中插入或是加载数据

##### 4.4.2 Hive 索引

Hive 支持在表中建立索引。但是索引需要额外的存储空间，因此在创建索引时需要考虑索引的必要性。

Hive 不支持直接使用 DROP TABLE 语句删除索引表。如果创建索引的表被删除了，则其对应的索引和索引表也会被删除；如果表的某个分区被删除了，则该分区对应的分区索引也会被删除。

### 4.5 Sort By、Order By、Distrbute By、Cluster By

1.  Sort By：分区内有序
2.  Order By：全局排序，只有一个 Reducer
3.  Distrbute By：类似 MR 中 Partition，进行分区，结合 sort by 使用
4.  Cluster By：当 Distribute by 和 Sorts by 字段相同时，可以使用 Cluster by 方式。Cluster by 还兼具 Sort by 的功能，但只能是升序排序。

### 4.6  内部表 跟外部表

##### 4.6.1 内部表

如果 Hive 中没有特别指定，则默认创建的表都是管理表，也称内部表。由 Hive 负责管理表中的数据，管理表不共享数据。删除管理表时，会删除管理表中的数据和元数据信息。

##### 4.6.2 外部表

当一份数据需要被共享时，可以创建一个外部表指向这份数据。删除该表并不会删除掉原始数据，删除的是表的元数据。

### 4.7 UDF 、UDAF、UDTF

1.  UDF ：一进一出，类似 upper，trim
2.  UDAF：多进一出，聚集函数，类似 count、max、min。
3.  UDTF：一进多出，如 lateral view explore()

### 4.8 HQL 如何转变为 MapReduce

1.  Antlr 定义 SQL 语法规则，完成 SQL 词法，语法解析，SQL 转化为 抽象语法树 AST Tree。
2.  遍历 AST Tree，抽象出查询的基本组成单元 QueryBlock。
3.  遍历 QueryBlock，翻译为执行操作树 OperatorTree。
4.  逻辑层优化 OperatorTree 变换，合并不必要的 ReduceSinkOperator，减少 Shuffle 数量。
5.  遍历 OperatorTree 翻译为 MapReduce 任务。
6.  物理层优化器进行 MapReduce 任务变换，生成最终执行计划。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dUIzkNGFzMg64FLrfTWrHgM9J2iaLic42gnMXicMsTLO6o83icLIhbBn6LNzbA8FIKWicyFm6XFYWibibwGg/640?wx_fmt=png)

源网侵删 
 [https://mp.weixin.qq.com/s/1A7pg091IW54YX97zh5Q_w](https://mp.weixin.qq.com/s/1A7pg091IW54YX97zh5Q_w)
