# Hive窗口函数/分析函数详解
在 sql 中有一类函数叫做聚合函数, 例如 sum()、avg()、max() 等等, 这类函数可以将多行数据按照规则聚集为一行, 一般来讲聚集后的行数是要少于聚集前的行数的。但是有时我们想要既显示聚集前的数据, 又要显示聚集后的数据, 这时我们便引入了窗口函数。窗口函数又叫 OLAP 函数 / 分析函数，窗口函数兼具分组和排序功能。

窗口函数最重要的关键字是 **partition by** 和 **order by。** 

具体语法如下：**over (partition by xxx order by xxx)**

## sum,avg,min,max 函数

准备数据

     1建表语句: 2create table bigdata_t1( 3cookieid string, 4createtime string,   --day  5pv int 6) row format delimited  7fields terminated by ','; 8 9加载数据：10load data local inpath '/root/hivedata/bigdata_t1.dat' into table bigdata_t1;1112cookie1,2018-04-10,113cookie1,2018-04-11,514cookie1,2018-04-12,715cookie1,2018-04-13,316cookie1,2018-04-14,217cookie1,2018-04-15,418cookie1,2018-04-16,41920开启智能本地模式21SET hive.exec.mode.local.auto=true;

SUM 函数和窗口函数的配合使用：结果和 ORDER BY 相关, 默认为升序。

     1#pv1 2select cookieid,createtime,pv, 3sum(pv) over(partition by cookieid order by createtime) as pv1  4from bigdata_t1; 5 6#pv2 7select cookieid,createtime,pv, 8sum(pv) over(partition by cookieid order by createtime rows between unbounded preceding and current row) as pv2 9from bigdata_t1;1011#pv312select cookieid,createtime,pv,13sum(pv) over(partition by cookieid) as pv314from bigdata_t1;1516#pv417select cookieid,createtime,pv,18sum(pv) over(partition by cookieid order by createtime rows between 3 preceding and current row) as pv419from bigdata_t1;2021#pv522select cookieid,createtime,pv,23sum(pv) over(partition by cookieid order by createtime rows between 3 preceding and 1 following) as pv524from bigdata_t1;2526#pv627select cookieid,createtime,pv,28sum(pv) over(partition by cookieid order by createtime rows between current row and unbounded following) as pv629from bigdata_t1;303132pv1: 分组内从起点到当前行的pv累积，如，11号的pv1=10号的pv+11号的pv, 12号=10号+11号+12号33pv2: 同pv134pv3: 分组内(cookie1)所有的pv累加35pv4: 分组内当前行+往前3行，如，11号=10号+11号， 12号=10号+11号+12号，36                           13号=10号+11号+12号+13号， 14号=11号+12号+13号+14号37pv5: 分组内当前行+往前3行+往后1行，如，14号=11号+12号+13号+14号+15号=5+7+3+2+4=2138pv6: 分组内当前行+往后所有行，如，13号=13号+14号+15号+16号=3+2+4+4=13，39                             14号=14号+15号+16号=2+4+4=10

如果不指定 rows between, 默认为从起点到当前行;

如果不指定 order by，则将分组内所有值累加;

关键是理解 rows between 含义, 也叫做**window 子句**：

preceding：往前

following：往后

current row：当前行

unbounded：起点

unbounded preceding 表示从前面的起点

unbounded following：表示到后面的终点

AVG，MIN，MAX，和 SUM 用法一样。

## row_number,rank,dense_rank,ntile 函数

准备数据

     1cookie1,2018-04-10,1 2cookie1,2018-04-11,5 3cookie1,2018-04-12,7 4cookie1,2018-04-13,3 5cookie1,2018-04-14,2 6cookie1,2018-04-15,4 7cookie1,2018-04-16,4 8cookie2,2018-04-10,2 9cookie2,2018-04-11,310cookie2,2018-04-12,511cookie2,2018-04-13,612cookie2,2018-04-14,313cookie2,2018-04-15,914cookie2,2018-04-16,71516CREATE TABLE bigdata_t2 (17cookieid string,18createtime string,   --day 19pv INT20) ROW FORMAT DELIMITED 21FIELDS TERMINATED BY ',' 22stored as textfile;2324加载数据：25load data local inpath '/root/hivedata/bigdata_t2.dat' into table bigdata_t2;

-   ROW_NUMBER() 使用

    ROW_NUMBER() 从 1 开始，按照顺序，生成分组内记录的序列。


    1SELECT 2cookieid,3createtime,4pv,5ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY pv desc) AS rn 6FROM bigdata_t2;

-   RANK 和 DENSE_RANK 使用

    RANK() 生成数据项在分组中的排名，排名相等会在名次中留下空位 。

    DENSE_RANK() 生成数据项在分组中的排名，排名相等会在名次中不会留下空位。


    1SELECT 2cookieid,3createtime,4pv,5RANK() OVER(PARTITION BY cookieid ORDER BY pv desc) AS rn1,6DENSE_RANK() OVER(PARTITION BY cookieid ORDER BY pv desc) AS rn2,7ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY pv DESC) AS rn3 8FROM bigdata_t2 9WHERE cookieid = 'cookie1';

-   NTILE

    有时会有这样的需求: 如果数据排序后分为三部分，业务人员只关心其中的一部分，如何将这中间的三分之一数据拿出来呢? NTILE 函数即可以满足。

    ntile 可以看成是：把有序的数据集合平均分配到指定的数量（num）个桶中, 将桶号分配给每一行。如果不能平均分配，则优先分配较小编号的桶，并且各个桶中能放的行数最多相差 1。

    然后可以根据桶号，选取前或后 n 分之几的数据。数据会完整展示出来，只是给相应的数据打标签；具体要取几分之几的数据，需要再嵌套一层根据标签取出。


    1SELECT 2cookieid,3createtime,4pv,5NTILE(2) OVER(PARTITION BY cookieid ORDER BY createtime) AS rn1,6NTILE(3) OVER(PARTITION BY cookieid ORDER BY createtime) AS rn2,7NTILE(4) OVER(ORDER BY createtime) AS rn38FROM bigdata_t2 9ORDER BY cookieid,createtime;

## lag,lead,first_value,last_value 函数

-   LAG    
    **LAG(col,n,DEFAULT) 用于统计窗口内往上第 n 行值**第一个参数为列名，第二个参数为往上第 n 行（可选，默认为 1），第三个参数为默认值（当往上第 n 行为 NULL 时候，取默认值，如不指定，则为 NULL）


     1  SELECT cookieid, 2  createtime, 3  url, 4  ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY createtime) AS rn, 5  LAG(createtime,1,'1970-01-01 00:00:00') OVER(PARTITION BY cookieid ORDER BY createtime) AS last_1_time, 6  LAG(createtime,2) OVER(PARTITION BY cookieid ORDER BY createtime) AS last_2_time  7  FROM bigdata_t4; 8 910  last_1_time: 指定了往上第1行的值，default为'1970-01-01 00:00:00'  11                            cookie1第一行，往上1行为NULL,因此取默认值 1970-01-01 00:00:0012                            cookie1第三行，往上1行值为第二行值，2015-04-10 10:00:0213                            cookie1第六行，往上1行值为第五行值，2015-04-10 10:50:0114  last_2_time: 指定了往上第2行的值，为指定默认值15                           cookie1第一行，往上2行为NULL16                           cookie1第二行，往上2行为NULL17                           cookie1第四行，往上2行为第二行值，2015-04-10 10:00:0218                           cookie1第七行，往上2行为第五行值，2015-04-10 10:50:01

-   LEAD

    与 LAG 相反  
    **LEAD(col,n,DEFAULT) 用于统计窗口内往下第 n 行值**  
    第一个参数为列名，第二个参数为往下第 n 行（可选，默认为 1），第三个参数为默认值（当往下第 n 行为 NULL 时候，取默认值，如不指定，则为 NULL）


    1  SELECT cookieid,2  createtime,3  url,4  ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY createtime) AS rn,5  LEAD(createtime,1,'1970-01-01 00:00:00') OVER(PARTITION BY cookieid ORDER BY createtime) AS next_1_time,6  LEAD(createtime,2) OVER(PARTITION BY cookieid ORDER BY createtime) AS next_2_time 7  FROM bigdata_t4;

-   FIRST_VALUE

    取分组内排序后，截止到当前行，第一个值


    1  SELECT cookieid,2  createtime,3  url,4  ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY createtime) AS rn,5  FIRST_VALUE(url) OVER(PARTITION BY cookieid ORDER BY createtime) AS first1 6  FROM bigdata_t4;

-   LAST_VALUE

    取分组内排序后，截止到当前行，最后一个值


    1  SELECT cookieid,2  createtime,3  url,4  ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY createtime) AS rn,5  LAST_VALUE(url) OVER(PARTITION BY cookieid ORDER BY createtime) AS last1 6  FROM bigdata_t4;

如果想要取分组内排序后最后一个值，则需要变通一下：

    1  SELECT cookieid,2  createtime,3  url,4  ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY createtime) AS rn,5  LAST_VALUE(url) OVER(PARTITION BY cookieid ORDER BY createtime) AS last1,6  FIRST_VALUE(url) OVER(PARTITION BY cookieid ORDER BY createtime DESC) AS last2 7  FROM bigdata_t4 8  ORDER BY cookieid,createtime;

**特别注意 order  by**

如果不指定 ORDER BY，则进行排序混乱，会出现错误的结果

    1  SELECT cookieid,2  createtime,3  url,4  FIRST_VALUE(url) OVER(PARTITION BY cookieid) AS first2  5  FROM bigdata_t4;

## cume_dist,percent_rank 函数

这两个序列分析函数不是很常用，**注意： 序列函数不支持 WINDOW 子句**

-   数据准备


     1  d1,user1,1000 2  d1,user2,2000 3  d1,user3,3000 4  d2,user4,4000 5  d2,user5,5000 6 7  CREATE EXTERNAL TABLE bigdata_t3 ( 8  dept STRING, 9  userid string,10  sal INT11  ) ROW FORMAT DELIMITED 12  FIELDS TERMINATED BY ',' 13  stored as textfile;1415  加载数据：16  load data local inpath '/root/hivedata/bigdata_t3.dat' into table bigdata_t3;

-   CUME_DIST  和 order by 的排序顺序有关系

    CUME_DIST 小于等于当前值的行数 / 分组内总行数  order 默认顺序 正序 升序  
    比如，统计小于等于当前薪水的人数，所占总人数的比例


     1  SELECT  2  dept, 3  userid, 4  sal, 5  CUME_DIST() OVER(ORDER BY sal) AS rn1, 6  CUME_DIST() OVER(PARTITION BY dept ORDER BY sal) AS rn2  7  FROM bigdata_t3; 8 9  rn1: 没有partition,所有数据均为1组，总行数为5，10       第一行：小于等于1000的行数为1，因此，1/5=0.211       第三行：小于等于3000的行数为3，因此，3/5=0.612  rn2: 按照部门分组，dpet=d1的行数为3,13       第二行：小于等于2000的行数为2，因此，2/3=0.6666666666666666

-   PERCENT_RANK

    PERCENT_RANK 分组内当前行的 RANK 值 - 1 / 分组内总行数 - 1


     1  SELECT  2  dept, 3  userid, 4  sal, 5  PERCENT_RANK() OVER(ORDER BY sal) AS rn1,   --分组内 6  RANK() OVER(ORDER BY sal) AS rn11,          --分组内RANK值 7  SUM(1) OVER(PARTITION BY NULL) AS rn12,     --分组内总行数 8  PERCENT_RANK() OVER(PARTITION BY dept ORDER BY sal) AS rn2  9  FROM bigdata_t3;1011  rn1: rn1 = (rn11-1) / (rn12-1) 12         第一行,(1-1)/(5-1)=0/4=013         第二行,(2-1)/(5-1)=1/4=0.2514         第四行,(4-1)/(5-1)=3/4=0.7515  rn2: 按照dept分组，16       dept=d1的总行数为317       第一行，(1-1)/(3-1)=018       第三行，(3-1)/(3-1)=1

## grouping sets,grouping\_\_id,cube,rollup 函数

这几个分析函数通常用于 OLAP 中，不能累加，而且需要根据不同维度上钻和下钻的指标统计，比如，分小时、天、月的 UV 数。

-   数据准备


     1  2018-03,2018-03-10,cookie1 2  2018-03,2018-03-10,cookie5 3  2018-03,2018-03-12,cookie7 4  2018-04,2018-04-12,cookie3 5  2018-04,2018-04-13,cookie2 6  2018-04,2018-04-13,cookie4 7  2018-04,2018-04-16,cookie4 8  2018-03,2018-03-10,cookie2 9  2018-03,2018-03-10,cookie310  2018-04,2018-04-12,cookie511  2018-04,2018-04-13,cookie612  2018-04,2018-04-15,cookie313  2018-04,2018-04-15,cookie214  2018-04,2018-04-16,cookie11516  CREATE TABLE bigdata_t5 (17  month STRING,18  day STRING, 19  cookieid STRING 20  ) ROW FORMAT DELIMITED 21  FIELDS TERMINATED BY ',' 22  stored as textfile;2324  加载数据：25  load data local inpath '/root/hivedata/bigdata_t5.dat' into table bigdata_t5;

-   GROUPING SETS

    grouping sets 是一种将多个 group by 逻辑写在一个 sql 语句中的便利写法。

    等价于将不同维度的 GROUP BY 结果集进行 UNION ALL。

    **GROUPING\_\_ID**，表示结果属于哪一个分组集合。


     1  SELECT  2  month, 3  day, 4  COUNT(DISTINCT cookieid) AS uv, 5  GROUPING__ID  6  FROM bigdata_t5  7  GROUP BY month,day  8  GROUPING SETS (month,day)  9  ORDER BY GROUPING__ID;1011  grouping_id表示这一组结果属于哪个分组集合，12  根据grouping sets中的分组条件month，day，1是代表month，2是代表day1314  等价于 15  SELECT month,NULL,COUNT(DISTINCT cookieid) AS uv,1 AS GROUPING__ID FROM bigdata_t5 GROUP BY month UNION ALL 16  SELECT NULL as month,day,COUNT(DISTINCT cookieid) AS uv,2 AS GROUPING__ID FROM bigdata_t5 GROUP BY day;

再如：

     1  SELECT  2  month, 3  day, 4  COUNT(DISTINCT cookieid) AS uv, 5  GROUPING__ID  6  FROM bigdata_t5  7  GROUP BY month,day  8  GROUPING SETS (month,day,(month,day))  9  ORDER BY GROUPING__ID;1011  等价于12  SELECT month,NULL,COUNT(DISTINCT cookieid) AS uv,1 AS GROUPING__ID FROM bigdata_t5 GROUP BY month 13  UNION ALL 14  SELECT NULL,day,COUNT(DISTINCT cookieid) AS uv,2 AS GROUPING__ID FROM bigdata_t5 GROUP BY day15  UNION ALL 16  SELECT month,day,COUNT(DISTINCT cookieid) AS uv,3 AS GROUPING__ID FROM bigdata_t5 GROUP BY month,day;

-   CUBE

    根据 GROUP BY 的维度的所有组合进行聚合。


     1  SELECT  2  month, 3  day, 4  COUNT(DISTINCT cookieid) AS uv, 5  GROUPING__ID  6  FROM bigdata_t5  7  GROUP BY month,day  8  WITH CUBE  9  ORDER BY GROUPING__ID;1011  等价于12  SELECT NULL,NULL,COUNT(DISTINCT cookieid) AS uv,0 AS GROUPING__ID FROM bigdata_t513  UNION ALL 14  SELECT month,NULL,COUNT(DISTINCT cookieid) AS uv,1 AS GROUPING__ID FROM bigdata_t5 GROUP BY month 15  UNION ALL 16  SELECT NULL,day,COUNT(DISTINCT cookieid) AS uv,2 AS GROUPING__ID FROM bigdata_t5 GROUP BY day17  UNION ALL 18  SELECT month,day,COUNT(DISTINCT cookieid) AS uv,3 AS GROUPING__ID FROM bigdata_t5 GROUP BY month,day;

-   ROLLUP

    是 CUBE 的子集，以最左侧的维度为主，从该维度进行层级聚合。


     1  比如，以month维度进行层级聚合： 2  SELECT  3  month, 4  day, 5  COUNT(DISTINCT cookieid) AS uv, 6  GROUPING__ID   7  FROM bigdata_t5  8  GROUP BY month,day 9  WITH ROLLUP 10  ORDER BY GROUPING__ID;1112  --把month和day调换顺序，则以day维度进行层级聚合：1314  SELECT 15  day,16  month,17  COUNT(DISTINCT cookieid) AS uv,18  GROUPING__ID  19  FROM bigdata_t5 20  GROUP BY day,month 21  WITH ROLLUP 22  ORDER BY GROUPING__ID;23  （这里，根据天和月进行聚合，和根据天聚合结果一样，因为有父子关系，如果是其他维度组合的话，就会不一样）

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

五分钟学大数据

文: 园陌

长按右侧二维码

收获更多技术

![](https://mmbiz.qpic.cn/mmbiz_gif/c6gqmhWiafyobvdkCN1GNh88geK5jbWxJ16lNcEJicicNgoRJDre75yTz7LicJBqaZMmNrojibsmOxWcia5Dk7qKC0eg/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zFXFP8Nqo5j6gk7icneiciaibL6PKich0rZzoNRPrlEJGZCqUyC8Uia5GLY9CJj97iaSphibTruU79jxE9uLQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_gif/TN05MmJLxMrNiboff6rhmPUYUNXdWTXZfCTse4EDYOEoRVrlpgoZF6t00NhnrdePKAibI1zaap7wibR0iclbJZFS1A/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/8LC2yjjGo8J81fZJt6CxkQ](https://mp.weixin.qq.com/s/8LC2yjjGo8J81fZJt6CxkQ)
