# 一文带你走进HIVE的世界(1.8W字建议收藏)
文章底部有该文 PDF 的目录，关注公众号并且加微信好友，即可获得数据仓库体系文档，HBase 详细文档及 hive 详细文档，并可进群交流。

**HIVE 简介**

1

什么是 HIVE

      hive 是基于 Hadoop 构建的一套数据仓库分析系统，它提供了丰富的 SQL 查询方式来分析存储在 Hadoop 分布式文件系统中的数据：

 可以将结构化的数据文件映射为一张数据库表，并提供完整的 SQL 查询功能；

 可以将 SQL 语句转换为 MapReduce 任务运行，通过自己的 SQL 查询分析需要的内容，这套 SQL 简称 Hive SQL，使不熟悉 mapreduce 的用户可以很方便地利用 SQL 语言查询、汇总和分析数据。

2

HIVE 特点

-   可扩展: Hive 可以自由的扩展集群的规模，一般情况下不需要重启服务。
-   延展性: Hive 支持用户自定义函数，用户可以根据自己需求来实现自己的函数。
-   容错: 良好的容错性，节点出现问题 SQL 仍可完成执行。

**2**

**HIVE 架构**

1

架构图

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAklzkW16n2cX6Du6tVyC9LlMEkY6OGkSNLd54QuYtQp5xoCueYjljug/640?wx_fmt=png)

### **用户接口：Client**

-   CLI（hive shell）
-   JDBC/ODBC(java 访问 hive)
-   WEBUI（浏览器访问 hive）

### **元数据：Metastore**

元数据包括：表名、表所属的数据库（默认是 default）、表的拥有者、列 / 分区字段、表的类型（是否是外部表）、表的数据所在目录等；

默认存储在自带的 derby 数据库中，推荐使用 MySQL 存储 Metastore

### **Hadoop**

使用 HDFS 进行存储，使用 MapReduce 进行计算。

### **驱动器：Driver**

1. 解析器（SQL Parser）：将 SQL 字符串转换成抽象语法树 AST，这一步一般都用第三方工具库完成，比如 antlr；对 AST 进行语法分析，比如表是否存在、字段是否存在、SQL 语义是否有误。

2. 编译器（Physical Plan）：将 AST 编译生成逻辑执行计划。

3. 优化器（Query Optimizer）：对逻辑执行计划进行优化。

4. 执行器（Execution）：把逻辑执行计划转换成可以运行的物理计划。对于 Hive 来说，就是 MR/Spark。

Hive 通过给用户提供的一系列交互接口，接收到用户的指令 (SQL)，使用自己的 Driver，结合元数据 (MetaStore)，将这些指令翻译成 MapReduce，提交到 Hadoop 中执行，最后，将执行返回的结果输出到用户交互接口。

2

HIVE 与传统数据库的对比

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quADaI2nEfSJQrY34lUynicT2UfJKgPia94iasKDdAuJwIke4JSgQM8RwNjQ/640?wx_fmt=png)

总结：hive 具有 sql 数据库的外表，但应用场景完全不同，hive 只适合用来做批量数据统计分析

3

HIVE 的数据存储

     Hive 中所有的数据都存储在 HDFS 中，没有专门的数据存储格式（可支持 Text，SequenceFile，ParquetFile，RCFILE 等）

只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据。

Hive 中包含以下数据模型：DB、Table，External Table，Partition，Bucket。

-   db：在 hdfs 中表现为 ${hive.metastore.warehouse.dir} 目录下一个文件夹
-   table：在 hdfs 中表现所属 db 目录下一个文件夹
-   external table：与 table 类似，不过其数据存放位置可以在任意指定路径
-   partition：在 hdfs 中表现为 table 目录下的子目录
-   bucket：在 hdfs 中表现为同一个表目录下根据 hash 散列之后的多个文件

**3**

**HIVE 使用方式**

1

最基本使用

启动一个 hive 交互 shell

设置一些基本参数，让 hive 使用起来更便捷，比如：

让提示符显示当前库：

hive>set hive.cli.print.current.db=true;

显示查询结果时显示字段名称：

hive>set hive.cli.print.header=true;

但是这样设置只对当前会话有效，重启 hive 会话后就失效，解决办法：

在 linux 的当前用户目录中，编辑一个. hiverc 文件，将参数写入其中：

vi .hiverc

set hive.cli.print.header=true;

set hive.cli.print.current.db=true;

2

## **启动 Hive 服务使用**

启动 hive 的服务：

上述启动，会将这个服务启动在前台，如果要启动在后台，则命令如下：

不把日志记录在服务器磁盘：

nohup bin/hiveserver2 1>/dev/null 2>&1 &

记录日志到服务器磁盘：

nohup bin/hiveserver2 1>/var/log/hiveserver.log 2>/var/log/hiveserver.err &

启动成功后，可以在别的节点上用 beeline 去连接

方式（1）

回车，进入 beeline 的命令界面

beeline> !connect jdbc:hive2://hdp01:10000

（hadoop01 是 hiveserver2 所启动的那台主机名，端口默认是 10000）

 方式（2）

启动时直接连接：

bin/beeline -u jdbc:hive2://hdp01:10000 -n root

3

## **脚本化运行**

大量的 hive 查询任务，如果用交互式 shell 来进行输入的话，显然效率及其低下，因此，生产中更多的是使用脚本化运行机制：

该机制的核心点是：hive 可以用一次性命令的方式来执行给定的 hql 语句

### **hive -e**

hive -e "insert into table t_test select \* from t_test_1;"

### **hive -f**

Vim test.sql

Select count(1) from test;

Select name from test;

**4**

**HIVE 建库建表与数据导入**

1

建库

hive 中有一个默认的库：default    库目录：hdfs://hdp01:9000/user/hive/warehouse

新建库：

库建好后，在 hdfs 中会生成一个库目录：

hdfs://hdp01:9000/user/hive/warehouse/test.db

2

建表

### **建表语法**

CREATE \[EXTERNAL] TABLE \[IF NOT EXISTS] table_name   \[(col_name data_type \[COMMENT col_comment], ...)]   \[COMMENT table_comment]   \[PARTITIONED BY (col_name data_type \[COMMENT col_comment], ...)]   \[CLUSTERED BY (col_name, col_name, ...)   \[SORTED BY (col_name \[ASC|DESC], ...)] INTO num_buckets BUCKETS]   \[ROW FORMAT row_format]   \[STORED AS file_format]   \[LOCATION hdfs_path]

说明：

CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

EXTERNAL 关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION），Hive 创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

LIKE 允许用户复制现有的表结构，但是不复制数据。

ROW FORMAT

       DELIMITED \[FIELDS TERMINATED BY char] \[COLLECTION ITEMS             TERMINATED BY char]

       \[MAP KEYS TERMINATED BY char] \[LINES TERMINATED BY char]

 SERDE serde_name \[WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 ROW FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive 通过 SerDe 确定表的具体的列的数据。

 STORED AS SEQUENCEFILE|TEXTFILE|RCFILE

如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

CLUSTERED BY

对于每一个表（table）或者分区， Hive 可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive 也是 针对某一列进行桶的组织。Hive 采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。

把表（或者分区）组织成桶（Bucket）有两个理由：

（1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如 JOIN 操作。对于 JOIN 操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行 JOIN 操作就可以，可以大大较少 JOIN 的数据量。

（2）使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

### **基本建表语句**

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

;

这样建表的话，hive 会认为表数据文件中的字段分隔符为 ^A（\\001）

指定分隔符建表语句为：

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

row format delimited fields terminated by ','

;

这样就指定了，表数据文件中的字段分隔符为 ","

注意：hive 是不会检查用户导入表中的数据的！如果数据的格式跟表定义的格式不一致，hive 也不会做任何处理（能解析就解析，解析不了就是 null）；

### **删除表**

drop table wedw_tmp.lzc_test;

删除表的效果是：

-    hive 会从元数据库中清除关于这个表的信息；
-    hive 还会从 hdfs 中删除这个表的表目录；

### **内部表与外部表**

-   内部表 (MANAGED_TABLE)：表目录按照 hive 的规范来部署，位于 hive 的仓库目录 / user/hive/warehouse 中
-   外部表 (EXTERNAL_TABLE)：表目录由建表用户自己指定

drop table if exists wedw_tmp.lzc_test;

CREATE external TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

row format delimited fields terminated by ','

location '/data/hive'

;

外部表和内部表的特性差别：

-   内部表的目录在 hive 的仓库目录中 VS 外部表的目录由用户指定
-   drop 一个内部表时：hive 会清除相关元数据，并删除表数据目录
-   drop 一个外部表时：hive 只会清除相关元数据；

一个 hive 的数据仓库，最底层的表，一定是来自于外部系统，为了不影响外部系统的工作逻辑，在 hive 中可建 external 表来映射这些外部系统产生的数据目录；

然后，后续的 etl 操作，产生的各种中间表建议用 managed_table（内部表）

什么时候用外部表什么时候用内部表？

-   每天采集的 ng 日志和埋点日志, 在存储的时候建议使用外部表，因为日志数据是采集程序实时采集进来的，一旦被误删，恢复起来非常麻烦。而且外部表方便数据的共享。
-   抽取过来的业务数据，其实用外部表或者内部表问题都不大，就算被误删，恢复起来也是很快的，如果需要对数据内容和元数据进行紧凑的管理, 那还是建议使用内部表
-   在做统计分析时候用到的中间表，结果表可以使用内部表，因为这些数据不需要共享，使用内部表更为合适。并且很多时候结果分区表我们只需要保留最近 3 天的数据，用外部表的时候删除分区时无法删除数据。

### **分区表**

分区表的实质是：在表目录中为数据文件创建分区子目录，以便于在查询时，MR 程序可以针对指定的分区子目录中的数据进行处理，缩减读取数据的范围，提高效率！

比如，网站每天产生的浏览记录，浏览记录应该建一个表来存放，但是，有时候，我们可能只需要对某一天的浏览记录进行分析

这时，就可以将这个表建为分区表，每天的数据导入其中的一个分区；

当然，每日的分区目录，应该有一个目录名（分区字段）

#### **单个字段分区**

创建带一个分区字段的表

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

partitioned by(date_id string)

row format delimited fields terminated by ','

;

测试数据：

insert overwrite table wedw_tmp.lzc_test partition(date_id = '2020-10-22')

select

'001' as id

,'xiaoli' as user_name

,19 as user_age

;

hdfs dfs -ls /data/hive/warehouse/wedw/tmp/lzc_test

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quA9JAyicTSH5nPTqAx5u9KrkjuB90zdaWZEwv3zwc6p7bCW7j9p4OBSbw/640?wx_fmt=png)

hdfs dfs -ls /data/hive/warehouse/wedw/tmp/lzc_test/date_id=2020-10-22

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAdHK68iacMwXG9tMeZppxLibETz1c6KLaIkauWy1DfnPodQM8gibPicAkOQ/640?wx_fmt=png)

#### **多个分区字段**

建表：

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

partitioned by(date_id string,province_id string)

row format delimited fields terminated by ','

;

测试数据

insert overwrite table wedw_tmp.lzc_test partition(date_id = '2020-10-22',province_id ='jiangxi')

select

'001' as id

,'xiaoli' as user_name

,19 as user_age

;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quA3rsng8aWKlK3w21tSxQY5bh0Giav1wX1tR0OUIr0T7Jt06w7xJuF8lw/640?wx_fmt=png)

#### **动态分区**

测试数据

1,2020-10-01, 上海, app

2,2020-10-01, 上海, h5

3,2020-10-01, 上海, h5

4,2020-10-01, 上海, app

5,2020-10-01, 上海, web

6,2020-10-01, 上海, app

7,2020-10-01, 上海, h5

8,2020-10-01, 上海, h5

9,2020-10-01, 上海, app

10,2020-10-01, 上海, h5

建表语句

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id       string COMMENT '用户 id'

,visit_date    string COMMENT '访问日期'

,city_name     string COMMENT '城市名'

,type_id       string COMMENT '设备类型'

)

COMMENT'日志表'

row format delimited fields terminated by ','

;

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAUGvsZAOiapatVicPll0TJQvUXiaMUZsIw9HSbCuY9iaDLIODPjWbA68crg/640?wx_fmt=png)

动态分区

drop table if exists wedw_tmp.lzc_test_info;

CREATE TABLE wedw_tmp.lzc_test_info(

 user_id       string COMMENT '用户 id'

,visit_date    string COMMENT '访问日期'

,city_name     string COMMENT '城市名'

)

COMMENT'日志表'

partitioned by (type_id string)

row format delimited fields terminated by ','

;

set hive.exec.dynamic.partition.mode=nonstrict;

set hive.exec.dynamic.partition=true;

\-- 每个节点可以创建的最大分区数 默认值: 100

set hive.exec.max.dynamic.partitions.pernode=10000;

\-- 每个 mr 可以创建的最大分区数 默认值: 1000

set hive.exec.max.dynamic.partitions=10000;

insert overwrite table wedw_tmp.lzc_test_info PARTITION(type_id)

select

 user_id    

,visit_date

,city_name  

,type_id

from

wedw_tmp.lzc_test

;

select \* from wedw_tmp.lzc_test_info

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAAeXBJomicqVIZFomic0nn0Buibic12mqKxHECDhibmVFgYYyk0qzQTkicdxg/640?wx_fmt=png)

show partitions wedw_tmp.lzc_test_info;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quARqeOkAibpGyEpEIXyggiblqicYKnuicQBGdVtP6MMlgYySDNWuick39RiaHQ/640?wx_fmt=png)

### **CTAS 建表语法**

可以通过已存在表来建表：

create table wedw_tmp.lzc_test2 like wedw_tmp.lzc_test;

新建的 t_user_2 表结构定义与源表 t_user 一致，但是没有数据

在建表的同时插入数据

create table  wedw_tmp.lzc_test2 as select \*  from wedw_tmp.lzc_test;

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quATGjSto7Q8WZriaRlGwkp0pfdL9VtrDicPqkicssZxQZFSFYKX3TuUaic7Q/640?wx_fmt=png)

3

## **将数据文件导入 hive 的表**

方式 1：导入数据的一种方式：

手动用 hdfs 命令，将文件放入表目录；

hdfs dfs -put /data/a.txt  /data/hive/warehouse/wedw/tmp/lzc_test/date_id=2020-10-22

方式 2：在 hive 的交互式 shell 中用 hive 命令来导入本地数据到表目录

hive>load data local inpath '/data/a.txt'  into table wedw_tmp.lzc_test;

方式 3：用 hive 命令导入 hdfs 中的数据文件到表目录

hive>load data inpath '/a.log' into table  wedw_tmp.lzc_test;

注意：导本地文件和导 HDFS 文件的区别：

-   本地文件导入表：复制
-   hdfs 文件导入表：移动

方式 4：如果目标表是一个分区表

hive> load data \[local] inpath ‘......’  into table t_dest partition(p=’value’);

### **将 hive 表中的数据导出到指定路径的文件**

将 hive 表中的数据导入 HDFS 的文件

insert overwrite directory '/root/test-data' row format delimited fields terminated by ','select \* fromwedw_tmp.lzc_test;

将 hive 表中的数据导入本地磁盘文件

insert overwrite local directory '/root/test-data'row format delimited fields terminated by ','select \* from wedw_tmp.lzc_test ;

### **hive 文件格式**

HIVE 支持很多种文件格式：SEQUENCE FILE | TEXT FILE | PARQUET FILE | RC FILE

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

partitioned by(date_id string,province_id string)

row format delimited fields terminated by ','

stored as textfile;

;

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

partitioned by(date_id string,province_id string)

row format delimited fields terminated by ','

stored as sequencefile;

;

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string COMMENT 'id'

,user_name    string COMMENT '用户姓名'

,user_age     int    COMMENT '用户年龄'

)

COMMENT'用户表'

partitioned by(date_id string,province_id string)

row format delimited fields terminated by ','

stored as parquetfile;

;

**5**

**数据类型**

1

## **数字类型**

TINYINT (1 字节整数)

SMALLINT (2 字节整数)

INT/INTEGER (4 字节整数)

BIGINT (8 字节整数)

FLOAT (4 字节浮点数)

DOUBLE (8 字节双精度浮点数)

2

## **时间类型**

TIMESTAMP (时间戳) (包含年月日时分秒毫秒的一种封装)

DATE (日期)（只包含年月日）

3

## **字符串类型**

STRING

4

## **其他类型**

BOOLEAN（布尔类型）：true  false

5

## **复合 (集合) 类型**

### **array 数组类型**

arrays: ARRAY&lt;data_type> )

示例：array 类型的应用

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 subject_id      string COMMENT '课程 id'

,subject_name    array<string> COMMENT '课程名称'

)

COMMENT'课程表'

row format delimited fields terminated by ','

collection items terminated by ':';

stored as textfile;

;

准备数据

1,spark:flink

2,hadoop,kafka,scala

导入数据：

/usr/local/hadoop-current/bin/hdfs dfs -put aaaa.txt /data/hive/warehouse/wedw/tmp/lzc_test/

查询：

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quA9HHIOfw2UtHfYqH5Dsc5wH8rkKezR48SYQGCvZWfibfUaBrvanJmticw/640?wx_fmt=png)

### **map 类型**

建表

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_name      string             COMMENT '用户 id'

,user_info      map&lt;string,string> COMMENT '用户信息'

)

COMMENT'用户信息表'

ROW FORMAT DELIMITED FIELDS TERMINATED BY ','

COLLECTION ITEMS TERMINATED BY '#'

MAP KEYS TERMINATED BY ":"

STORED AS TEXTFILE

;

数据准备

xiaoming,age:19#score:20#hight:180

xiaohong,age:21#score:100#hight:168

查询

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAQP1BAEBsMfOniaVR4um2D6LYRP5jmd5kGCPuotNhXznydarayIYoUcQ/640?wx_fmt=png)

取 map 字段的指定 key 的值

select user_name,user_info\['age'] from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAoHN1GwNaFvPHPcN2lg8T3083af6Niaf0ibfMH86JvBmp2GNa4ADibESgA/640?wx_fmt=png)

取 map 字段的所有 key

select user_name,map_keys(user_info) from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAnbDKVneVOzHfzyR4TDiawnmhrBkFicbEKLqwCBXj1vJAyS9ibicVnmqEYA/640?wx_fmt=png)

取 map 字段的所有 value

select user_name,map_values(user_info) from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAK9wBhVCdomPLA3ericC35BFw2vH3BVGbnCeOBmib26JOJHcScdYsrWhQ/640?wx_fmt=png)

### **struct 类型**

建表

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string                                 COMMENT '用户 id'

,user_info    struct&lt;age:int,sex:string,addr:string> COMMENT '用户姓名'

)

COMMENT'课程表'

row format delimited fields terminated by ','

collection items terminated by ':'

stored as textfile;

;

数据准备

1,zhangsan,18:male:beijing

2,lisi,28:female:shanghai

3\. 查询

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAicGpibCvfiaoq5TGeVGwk1cdJJDM7ic9VWeGN1ssgsWlyAmx8rmo7JQqNw/640?wx_fmt=png)

**6**

**DDL 语言**

仅修改 Hive 元数据，不会触动表中的数据，用户需要确定实际的数据布局符合元数据的定义。

1

## **修改表名**

ALTER TABLE table_name RENAME TO new_table_name

2

## **修改分区名**

alter table t_partition partition(department='xiangsheng',sex='male',howold=20) rename to partition(department='1',sex='1',howold=20);

3

## **添加分区**

// 一次添加一个分区

ALTER TABLE table_name ADD IF NOT EXISTS PARTITION (date_id='2019-01-07') LOCATION '/user/hadoop/warehouse/table_name/date_id=20190107';

 // 一次添加多个分区

ALTER TABLE page_view ADD PARTITION (date_id='2019-01-07', country='us') location '/path/to/us/part1' PARTITION (date_id='2019-01-07', country='us') location '/path/to/us/part2'; 

4

## **删除分区**

ALTER TABLE tableName  DROP IF EXISTS PARTITION (date_id='2020-07-26');

5

## **修改表的文件格式定义**

ALTER TABLE table_name \[PARTITION partitionSpec] SET FILEFORMAT file_format

6

## **修改表的某个文件格式定义**

alter table t_partition partition(department='2',sex='0',howold=40) set fileformat sequencefile;

7

## **修改列名定义**

ALTER TABLE tableName CHANGE COLUMN column newColumn STRING COMMENT '这里是列注释!'; 

8

## **增加 / 替换列**

ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type\[COMMENT col_comment], ...)

alter table t_user add columns (sex string); // 在所有存在的列后面，在分区列之前添加一列

alter table t_user replace columns (id string,age int,price float);

9

## **修改分区**

ALTER TABLE tableName PARTITION (date_id='2020-07-26') SET LOCATION "new location";

ALTER TABLE tableName PARTITION (date_id='2020-07-26') RENAME TO PARTITION (date_id='20200726');

10

## **修改表属性**

alter table tableName set TBLPROPERTIES ('EXTERNAL'='TRUE');  // 内部表转外部表 alter table tableName set TBLPROPERTIES ('EXTERNAL'='FALSE');  // 外部表转内部表

11

## **查看分区**

describe formatted tableName partition(date_id="2020-07-26");

12

## **查看 table 在 hdfs 上的存储路径及建表语句**

show create table  tableName ;

13

## **剔除不可见符号**

regexp_replace("内容",'&lt;\[^>^&lt;]\*>|&|nbsp|rdquo|&|"|\\n|\\r|\[\\\\000-\\\\037]','')

14

## **修改表注释**

ALTER TABLE tableName SET TBLPROPERTIES('comment' = '这是表注释!');

15

## **修改字段注释**

ALTER TABLE tableName CHANGE COLUMN column newColumn STRING COMMENT '这里是列注释!';

**7**

**HIVE 函数**

1

## **常用内置函数**

### **数学运算函数**

select round(5.4);   ## 5  四舍五入

select round(5.1345,3) ;  ##5.135

select ceil(5.4) ; //

select ceiling(5.4) ;   ## 6  向上取整

select floor(5.4);  ## 5  向下取整

select abs(-5.4) ;  ## 5.4  绝对值

select greatest(id1,id2,id3) ;  ## 6  单行函数

select least(3,5,6) ;  ## 求多个输入参数中的最小值

select max(age) from t_person group by ..;    分组聚合函数

select min(age) from t_person group by...;    分组聚合函数

### **字符串函数**

substr(string str, int start)   ## 截取子串

substring(string str, int start)

示例：select substr("abcdefg",2) ;

substr(string, int start, int len)

substring(string, int start, int len)

示例：select substr("abcdefg",2,3) ;  ## bcd

concat(string A, string B...)  ## 拼接字符串

concat_ws(string SEP, string A, string B...)

示例：select concat("ab","xy") ;  ## abxy

select concat_ws(".","192","168","33","44") ; ## 192.168.33.44

length(string A)

示例：select length("192.168.33.44");  ## 13

split(string str, string pat)  ## 切分字符串，返回数组

示例：select split("192.168.33.44",".") ; 错误的，因为. 号是正则语法中的特定字符

select split("192.168.33.44","\\.") ;

upper(string str)  ## 转大写

lower(string str)  ## 转小写

### **时间函数**

select current_timestamp(); ## 返回值类型：timestamp，获取当前的时间戳 (详细时间信息)

select current_date;   ## 返回值类型：date，获取当前的日期

\## unix 时间戳转字符串格式——from_unixtime

from_unixtime(bigint unixtime\[, string format])

示例：select from_unixtime(unix_timestamp());

select from_unixtime(unix_timestamp(),"yyyy/MM/dd HH:mm:ss");

\## 字符串格式转 unix 时间戳——unix_timestamp：返回值是一个长整数类型

\## 如果不带参数，取当前时间的秒数时间戳 long--(距离格林威治时间 1970-1-1 0:0:0 秒的差距)

select unix_timestamp();

unix_timestamp(string date, string pattern)

select unix_timestamp("2020-07-26 17:50:30");

select unix_timestamp("2020-07-26 17:50:30","yyyy-MM-dd HH:mm:ss");

\## 将字符串转成日期 date

select to_date("2020-07-26 16:58:32");

日期其他函数

date_add(string startdate, intdays) -- 返回开始日期 startdate 增加 days 天后的日期 date_sub (string startdate,int days) datediff(string enddate,string startdate) -- 日期差

trunc(string date,'MM') -- 返回当前月份的第一天

 select date_add('2020-06-27',2) as dt;

select date_sub('2015-05-14',7);

select datediff('2020-05-13','2020-05-02') as dd;

select trunc('2020-05-13','MM');

### **条件控制函数**

IF

select id,if(age>25,'working','worked') from t_user;

 CASE WHEN

语法：

CASE   \[ expression ]

       WHEN condition1 THEN result1

       WHEN condition2 THEN result2

       ...

       WHEN conditionn THEN resultn

       ELSE result

END

 COALESCE

这个参数使用的场合为：假如某个字段默认是 null，你想其返回的不是 null，而是比如 0 或其他默认值，可以使用这个函数 

SELECT COALESCE(field_name'-99’) as value from table;

### **集合函数**

array(3,5,8,9)   构造一个整数数组

array(‘hello’,’moto’,’semense’,’chuizi’,’xiaolajiao’)   构造一个字符串数组

array_contains(Array<T>, value)  返回 boolean 值

size(Map&lt;K.V>)  返回一个 imap 的元素个数，int 值

size(array<T>)   返回一个数组的长度, int 值

map_keys(Map&lt;K.V>)  返回一个 map 字段的所有 key，结果类型为：数组

map_values(Map&lt;K.V>) 返回一个 map 字段的所有 value，结果类型为：数组

### **常见分组聚合函数**

sum(字段)  :  求这个字段在一个组中的所有值的和

avg(字段)  ：求这个字段在一个组中的所有值的平均值

max(字段)  ：求这个字段在一个组中的所有值的最大值

min(字段)  ：求这个字段在一个组中的所有值的最小值

collect_set() : 将某个字段在一组中的所有值形成一个集合（数组）返回

### **表生成函数**

对一个值能生成多个值（explode 多行，json_tuple 多列）

表生成函数 lateral view explode()

drop table if exists wedw_tmp.lzc_test;

CREATE TABLE wedw_tmp.lzc_test(

 user_id      string         COMMENT 'id'

,user_name    string         COMMENT '用户姓名'

,cource_name  array<string>  COMMENT '课程名称'

)

COMMENT'用户表'

row format delimited fields terminated by ','

collection items terminated by ':'

stored as textfile;

;

select \* from wedw_tmp.lzc_test;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAEtISsvzibKkd7XVnI2c6YlIkpSzLWV0KNfQS2ZyU2As2DQgj8Izlz7g/640?wx_fmt=png)

select

 user_id

,user_name

,tmp.sub

from

wedw_tmp.lzc_test

lateral view explode(cource_name) tmp as sub

;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quATlhjljJXIJs8eluzgaSOomj9zUKeUJF7R0WPSmPvmD7IAWibtNU2LdA/640?wx_fmt=png)

理解：lateral view 相当于两个表在 join

左表：是原表

右表：是 explode(某个集合字段) 之后产生的表

而且：这个 join 只在同一行的数据间进行

### **json 解析函数：表生成函数**

1\. 利用 json_tuple 进行 json 数据解析

select json_tuple(json,'id','name') as (id,name) from t_json limit 10;

2.get_json_object()

get_json_object 函数第一个参数填写 json 对象变量，第二个参数使用 $ 表示 json 变量标识，然后用 . 或 \[] 读取对象或数组；

select 

get_json_object(content,'$. 测试') as Testcontent 

from testTableName;

### **窗口分析函数**

#### **row_number() over()**

有如下数据：

江西, 高安, 100

江西, 南昌, 200

江西, 丰城, 100

江西, 上高, 80

江西, 宜春, 150

江西, 九江, 180

湖北, 黄冈, 130

湖北, 武汉, 210

湖北, 宜昌, 140

湖北, 孝感, 90

湖南, 长沙, 170

湖南, 岳阳, 120

湖南, 怀化, 100

需要查询出每个省下人数最多的 2 个市

create table wedw_tmp.t_rn(

 province_name string COMMENT '省份'

,city_name     string COMMENT '市'

,pc_cnt        bigint COMMENT '人数'

)

row format delimited fields terminated by ',';

使用 row_number 函数，对表中的数据按照省份分组，按照人数倒序排序并进行标记

select

 province_name

,city_name    

,pc_cnt       

,row_number() over(partition by province_name order by pc_cnt desc) as rn

from

wedw_tmp.t_rn

;

产生结果：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAu8gHiajcWy2APoAueQpVTNOD28YCLjOCciax1zXG7XIXQtEtjgvgOichQ/640?wx_fmt=png)

然后，利用上面的结果，查询出 rn&lt;=2 的即为最终需求

select

 tmp.province_name

,tmp.city_name    

,tmp.pc_cnt

from

(

select

 province_name

,city_name    

,pc_cnt       

,row_number() over(partition by province_name order by pc_cnt desc) as rn

from

wedw_tmp.t_rn

) tmp

where tmp.rn &lt;= 2

;

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAHwXFb3ORI1Dt1HzPmmic4XjMDWfYdicQqgoq6FkUtib5rIjrVXtzwFTFQ/640?wx_fmt=png)

**sum() over() -- 级联求和**

数据准备

A,2020-01,15

A,2020-02,19

A,2020-03,12

A,2020-04,5

A,2020-05,29

B,2020-01,8

B,2020-02,6

B,2020-03,13

B,2020-04,5

B,2020-05,24

C,2020-01,16

C,2020-02,2

C,2020-03,33

C,2020-04,51

C,2020-05,54

建表

create table wedw_tmp.t_sum_over(

 user_name       string COMMENT '姓名'

,month_id        string COMMENT '月份'

,sale_amt        int    COMMENT '销售额'

)

row format delimited fields terminated by ',';

对于每个人的一个月的销售额和累计到当前月的销售总额

select

user_name

,month_id

,sale_amt

,sum(sale_amt) over(partition by user_name order by month_id rows between unbounded preceding and current row) as all_sale_amt

from wedw_tmp.t_sum_over;

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quASDzmH9fKzXibCCnKBo8A4KNOicibN1rIkDNXklRahqZMOqyLXKbGoIqLg/640?wx_fmt=png)

**lag() over() --（取出前 n 行数据）**

数据准备

create table t_hosp(

 user_name string

,age int

,in_hosp date

,out_hosp date)

row format delimited fields terminated by ',';

xiaohong,25,2020-05-12,2020-06-03

xiaoming,30,2020-06-06,2020-06-15

xiaohong,25,2020-06-14,2020-06-19

xiaoming,30,2020-06-20,2020-07-02

user_name: 用户名

age: 年龄

in_hosp: 住院日期

out_hosp：出院日期

需求：求同一个患者每次住院与上一次出院的时间间隔

第一步：

select

user_name

,age

,in_hosp

,out_hosp

,LAG(out_hosp,1,in_hosp) OVER(PARTITION BY user_name ORDER BY out_hosp asc) AS pre_out_date

from

t_hosp

;

其中，LAG(out_hosp,1,in_hosp) OVER(PARTITION BY user_name ORDER BY out_hosp asc)

表示根据 user_name 分组按照 out_hosp 升序取每条数据的上一条数据的 out_hosp，

如果上一条数据为空，则使用默认值 in_hosp 来代替

结果：

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAGHrUTINuXSJHnTQTUyemnmuyiamUiaIUCksbv0wDy4EESLAia7SXPdnlQ/640?wx_fmt=png)

第二步：每条数据的 in_hosp 与 pre_out_date 的差值即本次住院日期与上次出院日期的间隔

select

user_name

,age

,in_hosp

,out_hosp

,datediff(in_hosp,LAG(out_hosp,1,in_hosp) OVER(PARTITION BY user_name ORDER BY out_hosp asc)) as days

from

t_hosp

;

结果：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAfkfq5mTBeIibu1ickCBef5SulCpj0rPBia9Wk1fIpQGQiajI6DunSxDMmw/640?wx_fmt=png)

2

## **自定义函数**

### **为什么要自定义函数**

有时候 hive 自带的函数不能满足当前需要，需要自定义函数来解决问题

### **UDF，UDAF，UDTF 的比较**

-   UDF 操作作用于单个数据行，并且产生一个数据行作为输出。大多数函数都属于这一类（比如数学函数和字符串函数）。
-   UDAF 接受多个输入数据行，并产生一个输出数据行。像 COUNT 和 MAX 这样的函数就是聚集函数。
-   UDTF 操作作用于单个数据行，并且产生多个数据行，一个表作为输出。lateral view explore()

简单来说：

-   UDF: 返回对应值，一对一
-   UDAF：返回聚类值，多对一
-   UDTF：返回拆分值，一对多

### **UDF 函数开发**

1 代码

pom.xml

&lt;project xmlns="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0)"xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"

         xsi:schemaLocation="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0) [http://maven.apache.org/xsd/maven-4.0.0.xsd">](http://maven.apache.org/xsd/maven-4.0.0.xsd">)

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>

    <artifactId>udf_demo</artifactId>

    <version>1.1</version>

    <packaging>jar</packaging>

    <name>udf_demo</name>

    <properties>

        &lt;project.build.sourceEncoding>UTF-8&lt;/project.build.sourceEncoding>

        &lt;hadoop.version>2.6.0-cdh5.8.2&lt;/hadoop.version>

        &lt;hive.version>1.1.0-cdh5.8.2&lt;/hive.version>

    </properties>

    <dependencies>

        <dependency>

            <groupId>org.apache.hadoop</groupId>

            <artifactId>hadoop-common</artifactId>

            <version>2.6.0</version>

        </dependency>

        <dependency>

            <groupId>org.apache.hive</groupId>

            <artifactId>hive-exec</artifactId>

            <version>1.1.0</version>

        </dependency>

        <dependency>

            <groupId>org.apache.hive</groupId>

            <artifactId>hive-jdbc</artifactId>

            <version>1.1.0</version>

        </dependency>

        <dependency>

            <groupId>org.apache.hadoop</groupId>

            <artifactId>hadoop-hdfs</artifactId>

            <version>2.6.0</version>

        </dependency>

        <dependency>

            <groupId>log4j</groupId>

            <artifactId>log4j</artifactId>

            <version>1.2.17</version>

        </dependency>

    </dependencies>

    <build>

        <finalName>udf_demo</finalName>

        <plugins>

            <plugin>

                <artifactId>maven-assembly-plugin</artifactId>

                <configuration>

                    <archive>

                        <manifest>

                            <mainClass></mainClass>

                        </manifest>

                    </archive>

                    <descriptorRefs>

                        <descriptorRef>jar-with-dependencies</descriptorRef>

                    </descriptorRefs>

                </configuration>

            </plugin>

            <plugin>

                <groupId>org.apache.maven.plugins</groupId>

                <artifactId>maven-compiler-plugin</artifactId>

                <configuration>

                    <source>7</source>

                    <target>7</target>

                </configuration>

            </plugin>

        </plugins>

    </build>

</project>

base64 加密函数

package com.wedoctor;

import org.apache.commons.lang.StringUtils;

import org.apache.hadoop.hive.ql.exec.UDF;

import sun.misc.BASE64Encoder;

import java.io.UnsupportedEncodingException;

/\*\*

 \* Created by Liuzuochang on 2020/10/07.

 \* base64 加密 UDF

 \*/

public class Base64Encrypt extends UDF {

    public  String evaluate(String msg) throws Exception {

        // 判断传进来的参数是否为空

        if(StringUtils.isBlank(msg)){

            return "";

        }

        //base64 加密

        byte\[]  bt = null;

        String newMsg = null;

        try {

            bt = msg.getBytes("utf-8");

        } catch (UnsupportedEncodingException e) {

            e.printStackTrace();

        }

        if(bt != null){

            newMsg = new BASE64Encoder().encode(bt);

        }

        if(newMsg.contains("\\r\\n")){

            newMsg = newMsg.replace("\\r\\n","");

        }else if(newMsg.contains("\\r")){

            newMsg = newMsg.replace("\\r","");

        }else if(newMsg.contains("\\n")){

            newMsg = newMsg.replace("\\n","");

        }

        return newMsg;

    }

}

base64 解密函数

package com.wedoctor;

import org.apache.commons.lang.StringUtils;

import org.apache.hadoop.hive.ql.exec.UDF;

import sun.misc.BASE64Decoder;

/\*\*

 \* Created by Liuzuochang on 2020/10/07.

 \* base64 解密 UDF

 \*/

public class Base64Decrypt extends UDF {

    public  String evaluate(String msg) throws Exception {

        // 判断传进来的参数是否为空

        if(StringUtils.isBlank(msg)){

            return "";

        }

        //base64 解密

        byte\[] bt = null;

        String result = null;

        if(msg != null){

            BASE64Decoder decoder = new BASE64Decoder();

            try {

                bt = decoder.decodeBuffer(msg);

                result = new String(bt, "utf-8");

            } catch (Exception e) {

                e.printStackTrace();

            }

        }

        return result;

    }

}

2 打包上传到运行 hive 所在的服务器

hive> add jar /usr/local/udf_demo.jar;

3 创建临时函数

hive> create temporary function test_en as 'com.wedoctor.Base64Encrypt';hive> create temporary function test_de as 'com.wedoctor.Base64Decrypt';

4 加解密临时函数测试

hive> select test_en('1234');

MTIzNA==

hive> select test_de('MTIzNA==');

1234

**8**

**HIVE 优化**

1

前言

    Hive 作为大数据领域常用的组件，我们一定要充分的利用其特点，对其进行调优，提高运行效率，节省资源并保证任务及时产出

毫不夸张的说，有没有掌握 hive 调优，是判断一个数据工程师是否合格的重要指标 

       hive 调优涉及到压缩和存储调优，参数调优，sql 的调优，数据倾斜调优，小文件问题的调优等

2

## **数据的压缩与存储格式**

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quA5RpzJD3iaEicHXtNyg8qFDNia5u8cc3YGErFUNvRv5JadWnf0hbJbElpA/640?wx_fmt=png)

1. map 阶段输出数据压缩 ，在这个阶段，优先选择一个低 CPU 开销的算法。

set hive.exec.compress.intermediate=true

set mapred.map.output.compression.codec= org.apache.hadoop.io.compress.SnappyCodec

set mapred.map.output.compression.codec=com.hadoop.compression.lzo.LzoCodec;

2. 对最终输出结果压缩

set hive.exec.compress.output=true

set mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec

\## 当然，也可以在 hive 建表时指定表的文件格式和压缩编码

结论，一般选择 orcfile/parquet + snappy 方式

3

## **建表时 \*\***合理利用分区分桶 \*\*

      分区是将表的数据在物理上分成不同的文件夹，以便于在查询时可以精准指定所要读取的分区目录，从来降低读取的数据量

分桶是将表数据按指定列的 hash 散列后分在了不同的文件中，将来查询时，hive 可以根据分桶结构，快速定位到一行数据所在的分桶文件，从来提高读取效率

分区表：原来的一个大表存储的时候分成不同的数据目录进行存储。如果说是单分区表，那么在表的目录下就只有一级子目录，如果说是多分区表，那么在表的目录下有多少分区就有多少级子目录。不管是单分区表，还是多分区表，在表的目录下，和非最终分区目录下是不能直接存储数据文件的 

分桶表：原理和 hashpartitioner 一样，将 hive 中的一张表的数据进行归纳分类的时候，归纳分类规则就是 hashpartitioner。（需要指定分桶字段，指定分成多少桶）

分区表和分桶的区别除了存储的格式不同外，最主要的是作用：

分区表：细化数据管理，缩小 mapreduce 程序 需要扫描的数据量。

分桶表：提高 join 查询的效率，在一份数据会被经常用来做连接查询的时候建立分桶，分桶字段就是连接字段；提高采样的效率。

有了分区为什么还要分桶?

(1) 获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。

(2)使取样 ( sampling) 更高效。在处理大规模数据集时，在开发和修改査询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

分桶是相对分区进行更细粒度的划分。分桶将表或者分区的某列值进行 hash 值进行区分，如要按照 name 属性分为 3 个桶，就是对 name 属性值的 hash 值对 3 取摸，按照取模结果对数据分桶。

与分区不同的是，分区依据的不是真实数据表文件中的列，而是我们指定的伪列，但是分桶是依据数据表中真实的列而不是伪列

4

## **hive 参数优化**

// 让可以不走 mapreduce 任务的，就不走 mapreduce 任务

hive> set hive.fetch.task.conversion=more;

// 开启任务并行执行

 set hive.exec.parallel=true;

// 解释：当一个 sql 中有多个 job 时候，且这多个 job 之间没有依赖，则可以让顺序执行变为并行执行（一般为用到 union all 的时候）

 // 同一个 sql 允许并行任务的最大线程数

set hive.exec.parallel.thread.number=8;

// 设置 jvm 重用

// JVM 重用对 hive 的性能具有非常大的 影响，特别是对于很难避免小文件的场景或者 task 特别多的场景，这类场景大多数执行时间都很短。jvm 的启动过程可能会造成相当大的开销，尤其是执行的 job 包含有成千上万个 task 任务的情况。

set mapred.job.reuse.jvm.num.tasks=10;

// 合理设置 reduce 的数目

// 方法 1：调整每个 reduce 所接受的数据量大小

set hive.exec.reducers.bytes.per.reducer=500000000; （500M）

// 方法 2：直接设置 reduce 数量

set mapred.reduce.tasks = 20

// map 端聚合，降低传给 reduce 的数据量

set hive.map.aggr=true  

 // 开启 hive 内置的数倾优化机制（负载均衡）

set hive.groupby.skewindata=true

5

## **sql 优化**

### **谓词下推**

对于这个调优，有待再次验证

目前我验证了 4 千万 join  8 千万

发现不管是 left join 还是 inner join，执行计划差不多，job 数，maptask，reducetask 数都一样，执行时长也差不多

可能都已经自动优化了 (谓词下推)

select

count(1)

from test1 t1

inner join test2 t2

on t1.uuid = t2.uuid and t2.date_id = '2020-10-23'

where t1.date_id = '2020-10-23'

;

select

count(1)

from (select \* from test1 where date_id = '2020-10-23') t1

inner join test2 t2

on t1.uuid = t2.uuid and t2.date_id = '2020-10-23'

;

### **union 优化**

尽量不要使用 union （union 去掉重复的记录）而是使用 union all 然后再用 group by 去重

### **count distinct 优化**

数据量小的时候无所谓，数据量大的情况下，由于 COUNT DISTINCT 操作需要用一个 Reduce Task 来完成，这一个 Reduce 需要处理的数据量太大，就会导致整个 Job 很难完成，一般 COUNT DISTINCT 使用先 GROUP BY 再 COUNT 的方式替换：

不要使用 count (distinct  cloumn) , 使用子查询

select count(1) from (select id from tablename group by id) tmp;

虽然会多用一个 Job 来完成，但在数据量大的情况下，这个绝对是值得的。

### **用 in 来代替 join**

如果需要根据一个表的字段来约束另为一个表，尽量用 in 来代替 join . in 要比 join 快

select

Id

,name

from tb1  a

join tb2 b

on(a.id = b.id);

 select

Id

,name

 from tb1

where id in(

select

id

from

tb2

);

### **Left semi join 优化 in/exists**

(a 表和 b 表通过 user_id 关联)

 a 表数据

select \* from wedw_dw.t_user;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAeHeQ5w9tBjvFUKutSuk5TOMUUgPllrOvkNxqVYyPm0LdGTS7icHNSwA/640?wx_fmt=png)

 b 表数据

select \* from wedw_dw.t_order;

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAFYOOnTMHiaib7Tu8icsOa4Vk3ia1AW8mmicnggt5nPvNQh9oiawjUcZbmGAA/640?wx_fmt=png)

 left semi join

Select

-

from

wedw_dw.t_user t1

left semi join wedw_dw.t_order t2

on t1.user_id = t2.user_id;

如图所示：只能展示 a 表的字段，因为 left semi join 只传递表的 join key 给 map 阶段

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAUO1Ek7c4LgwNJ4Ee1RV1lyWme5HrZic2PlkYsRibD06EBicwFJPGpQkYQ/640?wx_fmt=png)

总结

LEFT SEMI JOIN 是 IN/EXISTS 子查询的一种更高效的实现。

LEFT SEMI JOIN 的限制是， JOIN 子句中右边的表只能在 ON 子句中设置过滤条件，在 WHERE 子句、SELECT 子句或其他地方都不行。

因为 left semi join 是 in(keySet) 的关系，遇到右表重复记录，左表会跳过，而 join 则会一直遍历。这就导致右表有重复值得情况下 left semi join 只产生一条，join 会产生多条，也会导致 left semi join 的性能更高。

left semi join 是只传递表的 join key 给 map 阶段，因此 left semi join 中最后 select 的结果只许出现左表。因为右表只有 join key 参与关联计算了，而 left  join on 默认是整个关系模型都参与计算了

### **大表 join 小表优化**

 Shuffle 阶段代价非常昂贵，因为它需要排序和合并。减少 Shuffle 和 Reduce 阶段的代价可以提高任务性能。

       MapJoin 通常用于一个很小的表和一个大表进行 join 的场景，具体小表有多小，由参数 hive.mapjoin.smalltable.filesize 来决定，该参数表示小表的总大小，默认值为 25000000 字节，即 25M。  
     Hive0.7 之前，需要使用 hint 提示 /_+ mapjoin(table) _/ 才会执行 MapJoin, 否则执行 Common Join，但在 0.7 版本之后，默认自动会转换 Map Join，由参数 hive.auto.convert.join 来控制，默认为 true.  
       假设 a 表为一张大表，b 为小表，并且 hive.auto.convert.join=true, 那么 Hive 在执行时候会自动转化为 MapJoin。

       MapJoin 简单说就是在 Map 阶段将小表数据从 HDFS 上读取到内存中的哈希表中，读完后将内存中的哈希表序列化为哈希表文件，在下一阶段，当 MapReduce 任务启动时，会将这个哈希表文件上传到 Hadoop 分布式缓存中，该缓存会将这些文件发送到每个 Mapper 的本地磁盘上。因此，所有 Mapper 都可以将此持久化的哈希表文件加载回内存，并像之前一样进行 Join。顺序扫描大表完成 Join。减少昂贵的 shuffle 操作及 reduce 操作

MapJoin 分为两个阶段：

通过 MapReduce Local Task，将小表读入内存，生成 HashTableFiles 上传至 Distributed Cache 中，这里会 HashTableFiles 进行压缩。

MapReduce Job 在 Map 阶段，每个 Mapper 从 Distributed Cache 读取 HashTableFiles 到内存中，顺序扫描大表，在 Map 阶段直接进行 Join，将数据传递给下一个 MapReduce 任务

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAMxlC9ywRGAJdiaW361VgERlDDEZNgSrVaoqvT4baS72b8oib1udsRUMg/640?wx_fmt=png)

### **大表 join 大表优化**

**空 key 过滤**

有时 join 超时是因为某些 key 对应的数据太多，而相同 key 对应的数据都会发送到相同的 reducer 上，从而导致内存不够。此时我们应该仔细分析这些异常的 key，很多情况下，这些 key 对应的数据是异常数据，我们需要在 SQL 语句中进行过滤。例如 key 对应的字段为空

**空 key 转换**

有时虽然某个 key 为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在 join 的结果中，此时我们可以表 a 中 key 为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的 reducer 上。例如：

insert overwrite table test

select

n.\*

from test1 n

full join test2 o

on case when n.id is null then concat('hive', rand()) else n.id end = o.id;

### **列裁剪**

只读取需要查询的当，列很多或者数据量很大时，select \* 效率很低

### **避免笛卡尔积**

尽量避免笛卡尔积，join 的时候不加 on 条件，或者无效的 on 条件，Hive 只能使用 1 个 reducer 来完成笛卡尔积。

### **查看 sql 的执行计划**

基本语法：EXPLAIN \[EXTENDED | DEPENDENCY | AUTHORIZATION] query

查看执行计划：

explain select \* from wedw_tmp.t_sum_over;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quARgriaBh1zcX9SeJCXicOqtGVrImbcfzDSUL14WYtkxaN95912cK7VhTQ/640?wx_fmt=png)

学会查看 sql 的执行计划，优化业务逻辑 ，减少 job 的数据量。对调优也非常重要

查看详细执行计划：

explain  extended select \* from wedw_tmp.t_sum_over;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAnEDIXDySSLrA62pCFstfGB7oEQwNnSomoNCvFmKKcFOsZw1tuowRNg/640?wx_fmt=png)

6

## **数据倾斜**

表现：任务进度长时间维持在 99%（或 100%），查看任务监控页面，发现只有少量（1 个或几个）reduce 子任务未完成。因为其处理的数据量和其他 reduce 差异过大。

原因：某个 reduce 的数据输入量远远大于其他 reduce 数据的输入量

### **sql 导致的倾斜**

1）group by

如果是在 group by 中产生了数据倾斜，是否可以讲 group by 的维度变得更细，如果没法变得更细，就可以在原分组 key 上添加随机数后分组聚合一次，然后对结果去掉随机数后再分组聚合

在 join 时，有大量为 null 的 join key，则可以将 null 转成随机值，避免聚集

默认情况下，Map 阶段同一 Key 数据分发给一个 reduce，当一个 key 数据过大时就倾斜了。

并不是所有的聚合操作都需要在 Reduce 端完成，很多聚合操作都可以先在 Map 端进行部分聚合，最后在 Reduce 端得出最终结果。

（1）是否在 Map 端进行聚合，默认为 True

hive.map.aggr = true

（2）在 Map 端进行聚合操作的条目数目

hive.groupby.mapaggr.checkinterval = 100000

（3）有数据倾斜的时候进行负载均衡（默认是 false）

hive.groupby.skewindata = true

当选项设定为 true，生成的查询计划会有两个 MR Job。

第一个 MR Job 中，Map 的输出结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；

第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

count(distinct)

情形：某特殊值过多

后果：处理此特殊值的 reduce 耗时；只有一个 reduce 任务

解决方式：count distinct 时，将值为空的情况单独处理，比如可以直接过滤空值的行，

在最后结果中加 1。如果还有其他计算，需要进行 group by，可以先将值为空的记录单独处理，再和其他计算结果进行 union。

Mapjoin

Shuffle 阶段代价非常昂贵，因为它需要排序和合并。减少 Shuffle 和 Reduce 阶段的代价可以提高任务性能。

       MapJoin 通常用于一个很小的表和一个大表进行 join 的场景，具体小表有多小，由参数 hive.mapjoin.smalltable.filesize 来决定，该参数表示小表的总大小，默认值为 25000000 字节，即 25M。  
     Hive0.7 之前，需要使用 hint 提示 /_+ mapjoin(table) _/ 才会执行 MapJoin, 否则执行 Common Join，但在 0.7 版本之后，默认自动会转换 Map Join，由参数 hive.auto.convert.join 来控制，默认为 true.  
       假设 a 表为一张大表，b 为小表，并且 hive.auto.convert.join=true, 那么 Hive 在执行时候会自动转化为 MapJoin。

       MapJoin 简单说就是在 Map 阶段将小表数据从 HDFS 上读取到内存中的哈希表中，读完后将内存中的哈希表序列化为哈希表文件，在下一阶段，当 MapReduce 任务启动时，会将这个哈希表文件上传到 Hadoop 分布式缓存中，该缓存会将这些文件发送到每个 Mapper 的本地磁盘上。因此，所有 Mapper 都可以将此持久化的哈希表文件加载回内存，并像之前一样进行 Join。顺序扫描大表完成 Join。减少昂贵的 shuffle 操作及 reduce 操作

MapJoin 分为两个阶段：

通过 MapReduce Local Task，将小表读入内存，生成 HashTableFiles 上传至 Distributed Cache 中，这里会 HashTableFiles 进行压缩。

MapReduce Job 在 Map 阶段，每个 Mapper 从 Distributed Cache 读取 HashTableFiles 到内存中，顺序扫描大表，在 Map 阶段直接进行 Join，将数据传递给下一个 MapReduce 任务

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAMxlC9ywRGAJdiaW361VgERlDDEZNgSrVaoqvT4baS72b8oib1udsRUMg/640?wx_fmt=png)

### **业务数据本身的特性 (存在热点 key)**

join 的每路输入都比较大，且长尾是热点值导致的，可以对热点值和非热点值分别进行处理，再合并数据

### **key 本身分布不均**

可以在 key 上加随机数，或者增加 reduceTask 数量

开启数据倾斜时负载均衡

set hive.groupby.skewindata=true;

思想：就是先随机分发并处理，再按照 key group by 来分发处理。

操作：当选项设定为 true，生成的查询计划会有两个 MRJob。

第一个 MRJob 中，Map 的输出结果集合会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 GroupBy Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；

第二个 MRJob 再根据预处理的数据结果按照 GroupBy Key 分布到 Reduce 中（这个过程可以保证相同的原始 GroupBy Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

### **控制空值分布**

将为空的 key 转变为字符串加随机数或纯随机数，将因空值而造成倾斜的数据分不到多个 Reducer。

注：对于异常值如果不需要的话，最好是提前在 where 条件里过滤掉，这样可以使计算量大大减少

### **合理设置 Reduce 数**

1．调整 reduce 个数方法一

（1）每个 Reduce 处理的数据量默认是 256MB

hive.exec.reducers.bytes.per.reducer=256000000

（2）每个任务最大的 reduce 数，默认为 1009

hive.exec.reducers.max=1009

（3）计算 reducer 数的公式

N=min(参数 2，总输入数据量 / 参数 1)

2．调整 reduce 个数方法二

在 hadoop 的 mapred-default.xml 文件中修改

设置每个 job 的 Reduce 个数

set mapreduce.job.reduces = 15;

3．reduce 个数并不是越多越好

1）过多的启动和初始化 reduce 也会消耗时间和资源；

2）另外，有多少个 reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

在设置 reduce 个数的时候也需要考虑这两个原则：处理大数据量利用合适的 reduce 数；使单个 reduce 任务处理数据量大小要合适；

7

## **合并小文件**

默认情况下一个小文件会启动一个 maptask 来处理数据，这样的话会启动大量的 maptask，但是每个 maptask 处理的数据量又非常少，task 启动和销毁也是非常消耗时间和资源，所以这个时候我们就需要想办法来合并这些小文件

小文件的产生有三个地方，map 输入，map 输出，reduce 输出，小文件过多也会影响 hive 的分析效率：

设置 map 输入的小文件合并

set mapred.max.split.size=256000000;  // 一个节点上 split 的至少的大小 (这个值决定了多个 DataNode 上的文件是否需要合并)

set mapred.min.split.size.per.node=100000000;// 一个交换机下 split 的至少的大小 (这个值决定了多个交换机上的文件是否需要合并)

set mapred.min.split.size.per.rack=100000000;

// 执行 Map 前进行小文件合并

set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

设置 map 输出和 reduce 输出进行合并的相关参数：

// 设置 map 端输出进行合并，默认为 true 

set hive.merge.mapfiles = true

// 设置 reduce 端输出进行合并，默认为 false 

set hive.merge.mapredfiles = true

// 设置合并文件的大小

set hive.merge.size.per.task = 256000000

// 当输出文件的平均大小小于该值时，启动一个独立的 MapReduce 任务进行文件 merge。

set hive.merge.smallfiles.avgsize=16000000

8

## **全局排序的优化**

Order By 只能是在一个 reduce 进程中进行，所以如果对一个大数据集进行全局排序，会非常慢

如果是取排序后的 N 条数据，可以使用 distribute by 和 sort by 在多个 reduce 上进行排序然后取每个分区的前 N 条数据进行合并后最后再一个 reduce 中排序取前 N 条，效率会有很大的提升

-   order by 会对输入做全局排序，因此只有一个 reducer(多个 reducer 无法保证全局有序)，然而只有一个 Reducer 会导致当输入规模较大时，消耗较长的计算时间。
-   sort by 不是全局排序，其在数据进入 reducer 前完成排序，因此，如果用 sort by 进行排序并且设置 mapped. reduce. tasks >1，则 sort by 只会保证每个 reducer 的输出有序，并不保证全局有序。(全排序实现: 先用 sortby 保证每个 reducer 输出有序，然后在进行 order by 归并下前面所有的 reducer 输出进行单个 reducer 排序，实现全局有序。)
-   distribute by 是控制在 map 端如何拆分数据给 reduce 端的。hive 会根据 distribute by 后面列，对应 reduce 的个数进行分发，默认是采用 hash 算法。sort by 为每个 reduce 产生一个排序文件。在有些情况下，你需要控制某个特定行应该到哪个 reducer，这通常是为了进行后续的聚集操作。distribute by 刚好可以做这件事。因此， distribute by 经常和 sort by 配合使用。并且 hive 规定 distribute by 语句要写在 sort by 语句之前
-   当 distribute by 和 sort by 所指定的字段相同时，即可以使用 cluster by。注意: cluster by 指定的列只能是升序，不能指定 asc 和 desc。

9

## **Fetch 抓取**

Fetch 抓取是指，Hive 中对某些情况的查询可以不必使用 MapReduce 计算。例如：SELECT \* FROM test; 在这种情况下，Hive 可以简单地读取 test 对应的存储目录下的文件，然后输出查询结果到控制台。

在 hive-default.xml.template 文件中 hive.fetch.task.conversion 默认是 more，老版本 hive 默认是 minimal，该属性修改为 more 以后，在全局查找、字段查找、limit 查找等都不走 mapreduce。

比如

select \* from test;

select id from test;

select \* from test limit 10;

10

## **本地模式**

大多数的 Hadoop Job 是需要 Hadoop 提供的完整的可扩展性来处理大数据集的。不过，有时 Hive 的输入数据量是非常小的。在这种情况下，为查询触发执行任务消耗的时间可能会比实际 job 的执行时间要多的多。对于大多数这种情况，Hive 可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。

用户可以通过设置 hive.exec.mode.local.auto 的值为 true，来让 Hive 在适当时候自动启动这个优化。

set hive.exec.mode.local.auto=true;  // 开启本地 mr

// 设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local  mr 的方式，默认为 134217728，即 128M

set hive.exec.mode.local.auto.inputbytes.max=50000000;

// 设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式，默认为 4

set hive.exec.mode.local.auto.input.files.max=10;

修改前：

select \* from wedw_tmp.t_sum_over;

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quA4gSs3G4Jc2N3Cvpiaj2GRRBebjAveuePZvic4HyzoJLhKxuCX34PNePA/640?wx_fmt=png)

修改后：

select \* from wedw_tmp.t_sum_over;

 ![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAUBaSsSqMNIY1WHX3yG2JAiamlFnEYP6XIfiaSSOQpgrzGX8KQDy3Ec7Q/640?wx_fmt=png)

11

## **JVM 重用**

JVM 重用是 Hadoop 调优参数的内容，其对 Hive 的性能具有非常大的影响，特别是对于很难避免小文件的场景或 task 特别多的场景，这类场景大多数执行时间都很短。

Hadoop 的默认配置通常是使用派生 JVM 来执行 map 和 Reduce 任务的。这时 JVM 的启动过程可能会造成相当大的开销，尤其是执行的 job 包含有成百上千 task 任务的情况。JVM 重用可以使得 JVM 实例在同一个 job 中重新使用 N 次。N 的值可以在 Hadoop 的 mapred-site.xml 文件中进行配置。通常在 10-20 之间，具体多少需要根据具体业务场景测试得出。

<property>

  <name>mapreduce.job.jvm.numtasks</name>

  <value>10</value>

  <description>How many tasks to run per jvm. If set to -1, there is

  no limit.

  </description>

</property>

这个功能的缺点是，开启 JVM 重用将一直占用使用到的 task 插槽，以便进行重用，直到任务完成后才能释放。如果某个 “不平衡的”job 中有某几个 reduce task 执行的时间要比其他 Reduce task 消耗的时间多的多的话，那么保留的插槽就会一直空闲着却无法被其他的 job 使用，直到所有的 task 都结束了才会释放。

12

## **严格模式**

Hive 提供了一个严格模式，可以防止用户执行那些可能意想不到的不好的影响的查询。

通过设置属性 hive.mapred.mode 值为默认是非严格模式 nonstrict 。开启严格模式需要修改 hive.mapred.mode 值为 strict，开启严格模式可以禁止 3 种类型的查询。

<property>

    <name>hive.mapred.mode</name>

    <value>strict</value>

    <description>

      The mode in which the Hive operations are being performed.

      In strict mode, some risky queries are not allowed to run. They include:

        Cartesian Product.

        No partition being picked up for a query.

        Comparing bigints and strings.

        Comparing bigints and doubles.

        Orderby without limit.

</description>

</property>

-   对于分区表，除非 where 语句中含有分区字段过滤条件来限制范围，否则不允许执行。换句话说，就是用户不允许扫描所有分区。进行这个限制的原因是，通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表。
-   对于使用了 order by 语句的查询，要求必须使用 limit 语句。因为 order by 为了执行排序过程会将所有的结果数据分发到同一个 Reducer 中进行处理，强制要求用户增加这个 LIMIT 语句可以防止 Reducer 额外执行很长一段时间。
-   限制笛卡尔积的查询。对关系型数据库非常了解的用户可能期望在执行 JOIN 查询的时候不使用 ON 语句而是使用 where 语句，这样关系数据库的执行优化器就可以高效地将 WHERE 语句转化成那个 ON 语句。不幸的是，Hive 并不会执行这种优化，因此，如果表足够大，那么这个查询就会出现不可控的情况。

12

## **并行执行**

Hive 会将一个查询转化成一个或者多个阶段。这样的阶段可以是 MapReduce 阶段、抽样阶段、合并阶段、limit 阶段。或者 Hive 执行过程中可能需要的其他阶段。默认情况下，Hive 一次只会执行一个阶段。不过，某个特定的 job 可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个 job 的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么 job 可能就越快完成。

通过设置参数 hive.exec.parallel 值为 true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果 job 中并行阶段增多，那么集群利用率就会增加。

set hive.exec.parallel=true;              // 打开任务并行执行

set hive.exec.parallel.thread.number=16;  // 同一个 sql 允许最大并行度，默认为 8。

当然，得是在系统资源比较空闲的时候才有优势，否则，没资源，并行也起不来。

目录：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfhbnRcr1sL7RpNE1uB6quAFjpfgQNibLX5dWVdQwQzpIN5kviatxwiclU1kgBNldkFQSv6pRqszzgNA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_gif/dXCnejTRMLfaPKxaibsZ7cCiaozWvibvo25R8yoqsvvTmiaG2PAMap0daZ8F31icEtXsianE5bA67SQ5Lh1fAbT9yLvA/640?wx_fmt=gif)

[2020 大数据面试题真题总结 (附答案)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485055&idx=1&sn=4e74131939bb203865fd6219df82516c&chksm=ea68ecb3dd1f65a5c0f55066118e1ea1de73b40b428ec52ae5589b72cca612c5e825a242b696&scene=21#wechat_redirect)  

[一文探究数据仓库体系 (2.7 万字建议收藏)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485691&idx=1&sn=d6cb1353031e07e4b02cd903d8b57911&chksm=ea68e237dd1f6b210f65f25ef42dabf4453d3bfa36fe8f33b149c0ff5329f77b9b792eef7882&scene=21#wechat_redirect)

[数据建模知多少？](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485443&idx=1&sn=8307b0922a3ee002446936f83153bcc6&chksm=ea68e2cfdd1f6bd96cd1de5fa46a3191c0a70e4e3ca570ed57c1c647abd624fcf8a0901f08cc&scene=21#wechat_redirect)  

如何写好一篇数据部门规范文档  

[你要悄悄学会 HBase，然后惊艳所有人 (1.7 万字建议收藏)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485858&idx=1&sn=75a3a3dd71a612b5ac4412c423554787&chksm=ea68e36edd1f6a785b739bd2ec2f0cc94f5d3679ac40f631fbd61abad96e9652bea14cc705f5&scene=21#wechat_redirect)  

[Hive 调优，数据工程师成神之路](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485048&idx=1&sn=5fc1219f4947bea9743cd938cec510c7&chksm=ea68ecb4dd1f65a2df364d79272e0e472a394c5b13b5d55d848c89d9498ccb7ea78a933fbdea&scene=21#wechat_redirect)  

[数据质量那点事](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485039&idx=1&sn=140c3bc720da51765292fe3f5082fe38&chksm=ea68eca3dd1f65b5aef4d6f7ab0c33d3d3033bcc0eead1650be079687e0b4e898562bfe4d25b&scene=21#wechat_redirect)  

[简述元数据管理](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485186&idx=1&sn=85fbe5703c56aa2dcfd2980fccbab4f6&chksm=ea68edcedd1f64d8e2d8c3da6b456fcaa4b105f2216a2bddb2393a7380498166225de5e855b4&scene=21#wechat_redirect)

[Sqoop or Datax](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484752&idx=1&sn=567442111447f2a7cac5379b694205f7&chksm=ea68ef9cdd1f668ad81e435e8c42a622f0ccfda773ff03485fad3cabeb1bd9aeea2e8cb40428&scene=21#wechat_redirect)

[大厂高频面试题 - 连续登录问题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484649&idx=1&sn=3070c0d82b45157b86885c2f6a9b4dd8&chksm=ea68ee25dd1f6733df5f3e734f06fd3bb4e07ce91fa129763994c67b4572f60e8afb274be7ce&scene=21#wechat_redirect)  

[朋友面试数据研发岗遇到的面试题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484533&idx=1&sn=e817ea44f364f00d44fbcd3b321cc2c3&chksm=ea68eeb9dd1f67af148952b86c7d4911f788f9943992f339d5804f261f8c55908f4237b86d64&scene=21#wechat_redirect)

[简单聊一聊大数据学习之路](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484074&idx=1&sn=f1b74ec65a9d0101cafb7741046e29f9&chksm=ea68e866dd1f61703304e4f01fb97d3b1f98aba6ea3fc5a9fb258b4c5e4e317d79edd7f15267&scene=21#wechat_redirect)

[朋友面试数据专家岗遇到的面试题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484277&idx=1&sn=12568aaf7f50f259ff1fd1764dd17ad0&chksm=ea68e9b9dd1f60af84a49c9653849fb215b92566fc6ded9ce0d056d481cea2855654bc72fbd7&scene=21#wechat_redirect)

[HADOOP 快速入门](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484163&idx=1&sn=0741c076904f81133cb2524033c5993f&chksm=ea68e9cfdd1f60d9254ef6bd1128dacf0b379d7fb1409f356177e2b8a9577ea8319b7619c141&scene=21#wechat_redirect)

[数仓工程师的利器 - HIVE 详解](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484172&idx=1&sn=8bdb336229dc8aed89edbe1ab9dce771&chksm=ea68e9c0dd1f60d6f1d864bef3c531d0f42163beb764d9d6d1a7febf15f6f541f9709a2fa80c&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLeibmoxSpRXUevnVIz4RyV0AUrglXsIgLOkFMNuffPAh5lZbwOnSoRHbZqT12VyKibm0EmnTicXBSRTg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/ynrmMBi2TUbTZBVwjIoUBA](https://mp.weixin.qq.com/s/ynrmMBi2TUbTZBVwjIoUBA)
