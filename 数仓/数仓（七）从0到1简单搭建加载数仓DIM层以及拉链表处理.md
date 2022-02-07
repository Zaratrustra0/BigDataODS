# 数仓（七）从0到1简单搭建加载数仓DIM层以及拉链表处理
[数仓（一）简介数仓，OLTP 和 OLAP](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503438&idx=1&sn=3d249b38099cdb4f3d17a7d4676f93de&chksm=f9ed3766ce9abe70d7ae724c1a83ad7669f19c4148b16bfc16f03dbc23f77b23465ca845a47d&scene=21#wechat_redirect)

[数仓（二）关系建模和维度建模](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503465&idx=1&sn=fee932bd60e96b2814f7a257399d8170&chksm=f9ed3741ce9abe57203c7cee8a0207d9c48b53cf261aa77ff6dd9b2d65ec85543f985f01acb7&scene=21#wechat_redirect)  

[数仓（三）简析阿里、美团、网易、恒丰银行、马蜂窝 5 家数仓分层架构](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503466&idx=2&sn=aa9ab1dd24fccdb41d9e805a937c12c4&chksm=f9ed3742ce9abe54f73b86a4db0125fe36dbc8c7889cd7452456b1ed058e2c13a9f4e26eb6d7&scene=21#wechat_redirect)  

[数仓（四）数据仓库分层](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503496&idx=2&sn=3f0c6f34fe307932efeaf4c3f19b5503&chksm=f9ed37a0ce9abeb60a5417821d94f7d1bfb9ef26844b0cf44aecf17aa5e901a91f85cda7ab91&scene=21#wechat_redirect)  

[数仓 (五) 元数据管理系统解析](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503532&idx=1&sn=c996c431e25199496e464270cf9aa8eb&chksm=f9ed3784ce9abe92a7b09b535b08e8fabacdc51807e801dc4bb7806de36288c39d7ac4111e03&scene=21#wechat_redirect)  

[数仓（六）从 0 到 1 简单搭建数仓 ODS 层 (埋点日志 + 业务数据)](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503555&idx=1&sn=9718725ec437cc7e9fe78167193bb958&chksm=f9ed37ebce9abefd9c68cdf5905be86e9718bab479e1a83868d17f6159b512b64052ecff1818&scene=21#wechat_redirect)  

回到数仓项目中，我们上一篇已经搭建了 ODS 层，并且把 HDFS 上的埋点数据和业务交易数据，load 到数仓的 ODS 层。本节我们在 ODS 层的基础上搭建 DIM 层即维度层，会根据不同的加载策略处理维度表并且讲解非常重要的拉链表的概念和使用，本节涉及很多 HQL 语句，不懂的童靴小白可以学一下。

**一、DIM 层表结构**

我们在 “[_数仓（四）数据仓库分层_](http://mp.weixin.qq.com/s?__biz=Mzg3ODU4ODMyOQ==&mid=2247484056&idx=1&sn=8bc3417fb27d8608c8cfcc2d91e7dc14&chksm=cf103a5ef867b34884f64d726824e2d4ee1f434576ae5aff68f6d516db1eaa12510ffca13147&scene=21#wechat_redirect)” 中讲解了什么是 DIM 层。这里在复述一下：

**1、DIM 层概念**

以维度作为建模驱动，基于每个维度的业务含义，通过添加维度属性、关联维度等定义计算逻辑，完成属性定义的过程并建立一致的数据分析维表。为了避免在维度模型中冗余关联维度的属性，基于雪花模型构建维度表。维度层的表通常也被称为维度逻辑表。

-   **高基数维度数据**

    一般是用户资料表、商品资料表等类似的资料表。数据量可能是千万级或者上亿级别。
-   **低基数维度数据**

    一般是配置表，比如枚举值对应的中文含义，比如国家、城市、县市、街道等维度表。数据量可能是个位数或者几千几万。

**2、DIM 表结构**

这里我们一共列出了有 6 个维度表，表名和加载策略方式如下：

\| **维度表** \| **表名** \| 

**加载方式**

 \|
| 商品维度表 | dim_sku_info  
 \| 

全量加载  

 \|
| 优惠券维度表 | dim_coupon_info | 

全量加载  

 \|
| 活动维度表 | dim_activity_rule_info | 

全量加载  

 \|
| 地区维度表 | dim_base_province | 

特殊加载  

 \|
| 时间维度表 | dim_date_info | 

特殊加载

 \|
| 用户维度表 | dim_user_info | 

拉链表  

 \|

**根据加载的类型有：全量加载、特殊加载、拉链表；下面一一说明**

**注意：** 这里有个加载策略方式是：拉链表，不懂拉链表的暂时可以先略过，下面讲到的时候在详细介绍。  

**二、商品维度表（全量加载）**

我们先来 DIM 层创建维度表的表结构。

**商品维度表 dim_sku_info（全量加载）**

创建表结构如下:

```sql
DROP TABLE IF EXISTS dim_sku_info;
CREATE EXTERNAL TABLE dim_sku_info (
    `id` STRING COMMENT '商品id',
    `price` DECIMAL(16,2) COMMENT '商品价格',
    `sku_name` STRING COMMENT '商品名称',
    `sku_desc` STRING COMMENT '商品描述',
    `weight` DECIMAL(16,2) COMMENT '重量',
    `is_sale` BOOLEAN COMMENT '是否在售',
    `spu_id` STRING COMMENT 'spu编号',
    `spu_name` STRING COMMENT 'spu名称',
    `category3_id` STRING COMMENT '三级分类id',
    `category3_name` STRING COMMENT '三级分类名称',
    `category2_id` STRING COMMENT '二级分类id',
    `category2_name` STRING COMMENT '二级分类名称',
    `category1_id` STRING COMMENT '一级分类id',
    `category1_name` STRING COMMENT '一级分类名称',
    `tm_id` STRING COMMENT '品牌id',
    `tm_name` STRING COMMENT '品牌名称',
    `sku_attr_values` ARRAY<STRUCT<attr_id:STRING,value_id:STRING,attr_name:STRING,value_name:STRING>> COMMENT '平台属性',
    `sku_sale_attr_values` ARRAY<STRUCT<sale_attr_id:STRING,sale_attr_value_id:STRING,sale_attr_name:STRING,sale_attr_value_name:STRING>> COMMENT '销售属性',
    `create_time` STRING COMMENT '创建时间'
) COMMENT '商品维度表'
PARTITIONED BY (`dt` STRING)
STORED AS PARQUET
LOCATION '/warehouse/gmall/dim/dim_sku_info/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

\***\*1、了解字段含义 \*\***

这里字段都很简单，重点关注 sku_attr_values 和 sku_sale_attr_values 这两个字段，由于平台、销售的属性个数和描述不一样，比如有些产品有 4 个平台属性，有些产品有 3 个完全和这 4 个不同的属性，这个时候我们怎么存储？使用数组结构体类型来封装平台、销售属性。

```ruby
`sku_attr_values` ARRAY<STRUCT<attr_id:STRING,value_id:STRING,attr_name:STRING,value_name:STRING>> COMMENT '平台属性',
`sku_sale_attr_values` ARRAY<STRUCT<sale_attr_id:STRING,sale_attr_value_id:STRING,sale_attr_name:STRING,sale_attr_value_name:STRING>> COMMENT '销售属性',
```

我们看一下 SKU_info、SKU_attr_value，base_attr_info、base_attr_value 这 4 张表结构：  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiac4Mt2papttjVufGw5KqUicwBQibQicDV4nibYaqW0AfZKUc1v4NlsWv052M1VfroIpBp0mZH5wHYd0A/640?wx_fmt=png)

**通过 sku_attr_values 表可以直接拿到属性：** 

attr_id:STRING,attr_name:STRING

value_id:STRING,,value_name:STRING

**同理，sku_sale_attr_values 表可拿销售属性：** 

sale_attr_id:STRING,sale_attr_value_id:STRING,

sale_attr_name:STRING,sale_attr_value_name:STRING

\***\*2、分区 \*\***

按照 dt 每天分区

```javascript
PARTITIONED BY (`dt` STRING)
```

\***\*3、存储格式 \*\***

PARQUET 列式存储，压缩格式采用 lzo；

```nginx
STORED AS PARQUET
LOCATION '/warehouse/gmall/dim/dim_sku_info/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

\***\*4、装载方式 \*\***

采用全量装载，ODS 层业务表首日（比如 5 月 1 日）数据全量加载到 DIM 层，第二日（5 月 2 日）数据也是全量加载到 DIM 层，以后每日 ODS 层数据都是全量加载到 DIM 层。

**5、实现的 SQL 语句**  

我们先来看分析一下字段的获取方法，通过 join 这 6 张表可以获取除结构体以外的字段。然后进行拼接。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUGVt1A2rD8yfIiaBF8NEQ5wMlIl7EmtvUUgNy4Plc6DMt3pkNv2azdyA/640?wx_fmt=png)

**5.1、join 共 6 张表获取字段（除结构体以外的字段）**

    1、ods_sku_info

```sql
select
    id,
    price,
    sku_name,
    sku_desc,
    weight,
    is_sale,
    spu_id,
    category3_id,
    tm_id,
    create_time
from ods_sku_info
where dt='2021-05-01'
```

    2、ods_spu_info

```sql
select
    id,
    spu_name
from ods_spu_info
where dt='2021-05-01'
```

    3、ods_base_category3  

```sql
select
    id,
    name,
    category2_id
from ods_base_category3
where dt='2021-05-01'
```

    4、ods_base_category2

```sql
select
    id,
    name,
    category1_id
from ods_base_category2
where dt='2021-05-01'
```

    5、ods_base_category1  

```sql
select
    id,
    name
from ods_base_category1
where dt='2021-05-01'
```

    6、ods_base_trademark

```sql
select
    id,
    tm_name
  from ods_base_trademark
  where dt='2021-05-01'
```

**5.2、获取结构体的字段**

 **1、先获取需要的字段**

```cs
select
    sku_id,
    attr_id,
    value_id,
    attr_name,
    value_name
from ods_sku_attr_value
where dt='2021-05-01'
```

 查看获取到的结果：  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUA4gIY9DaZkZ0STdds1n01wWWS26vAialDPAlxcvVIkdJ4jpwb4WGO3Q/640?wx_fmt=png)

**2、attr_id 属性通过转为结构体形式即一个属性对应一个值封装到一个结构体中**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUicMNgqYgapBJYZzkicicibtZMjKuvG3tdHpeVcQ96AEzVvZz1n9eNjoxJQ/640?wx_fmt=png)

```cs
select
    sku_id,
    named_struct('attr_id',attr_id,'value_id',value_id,'attr_name',attr_name,'value_name',value_name)
from ods_sku_attr_value
where dt='2021-05-01'
```

查看结果：我们已经把每个属性封装到了结构体。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUO1beSz6YoHOCBo2FEXClCjQ1dscoUicw5rma9fkXofGpmLTjWrK3xuA/640?wx_fmt=png)

**3、根据 sku_id 通过聚合函数封装到数组当中**

**其中 collect_set 函数做聚合操作，然后根据 sku_id 做 group by。** 

```cs
select
    sku_id,
    collect_set(named_struct('attr_id',attr_id,'value_id',value_id,'attr_name',attr_name,'value_name',value_name))
from ods_sku_attr_value
where dt='2021-05-01'
group by sku_id
```

查看结果：我们已经把每个属性封装到了结构体。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUbrv9eLupE95qnlhmkUrDHbt9AN9zuHyBkCmJxeNK71txNuQnr1GYvg/640?wx_fmt=png)

**6、装载数据**

把 ODS 层查询和拼接的数据加载到 DIM 层 dim_sku_info 表。

```sql
insert overwrite table dim_sku_info partition(dt='2021-05-01')
select
    sku.id,
    sku.price,
    sku.sku_name,
    sku.sku_desc,
    sku.weight,
    sku.is_sale,
    sku.spu_id,
    spu.spu_name,
    sku.category3_id,
    c3.name,
    c3.category2_id,
    c2.name,
    c2.category1_id,
    c1.name,
    sku.tm_id,
    tm.tm_name,
    attr.attrs,
    sale_attr.sale_attrs,
    sku.create_time
from sku
left join spu on sku.spu_id=spu.id
left join c3 on sku.category3_id=c3.id
left join c2 on c3.category2_id=c2.id
left join c1 on c2.category1_id=c1.id
left join tm on sku.tm_id=tm.id
left join attr on sku.id=attr.sku_id
left join sale_attr on sku.id=sale_attr.sku_id;
```

**注：** 由于 dim_sku_info 表是全量加载，每日加载全量数据所以首日加载好第二天加载一样，都是这个 insert 语句。

这样我们就完成了从 ODS 层到 DIM 层商品维度表的创建、装载。优惠券维度表、活动维度表和商品维度表的创建、装载一样。

\***\*DIM 层中优惠券维度表的创建和装载大同小异，**这里留给读者，你是否能完成呢？\*\*

**三、地区维度表（特殊加载）**

地区维度表是一张特殊表，没有分区。即每天不会装载数据。也是所有表中最容易处理的。  

**1、在 DIM 层创建表 dim_base_province**

表结构和字段含义，存储也是 PARQUET；

```sql
DROP TABLE IF EXISTS dim_base_province;
CREATE EXTERNAL TABLE dim_base_province (
    `id` STRING COMMENT 'id',
    `province_name` STRING COMMENT '省市名称',
    `area_code` STRING COMMENT '地区编码',
    `iso_code` STRING COMMENT 'ISO-3166编码，供可视化使用',
    `iso_3166_2` STRING COMMENT 'IOS-3166-2编码，供可视化使用',
    `region_id` STRING COMMENT '地区id',
    `region_name` STRING COMMENT '地区名称'
) COMMENT '地区维度表'
STORED AS PARQUET
LOCATION '/warehouse/gmall/dim/dim_base_province/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

**2、装载表**

地区表没有分区，数据变化不大，无需每日装载数据。

```sql
insert overwrite table dim_base_province
select
    bp.id,
    bp.name,
    bp.area_code,
    bp.iso_code,
    bp.iso_3166_2,
    bp.region_id,
    br.region_name
from ods_base_province bp
join ods_base_region br on bp.region_id = br.id;
```

**四、时间维度表（特殊加载）**

时间维度表也是一张特殊表。  

**1、创建表 dim_date_info**

表结构和字段含义，存储也是 PARQUET；

```sql
DROP TABLE IF EXISTS dim_date_info;
CREATE EXTERNAL TABLE dim_date_info(
    `date_id` STRING COMMENT '日',
    `week_id` STRING COMMENT '周ID',
    `week_day` STRING COMMENT '周几',
    `day` STRING COMMENT '每月的第几天',
    `month` STRING COMMENT '第几月',
    `quarter` STRING COMMENT '第几季度',
    `year` STRING COMMENT '年',
    `is_workday` STRING COMMENT '是否是工作日',
    `holiday_id` STRING COMMENT '节假日'
) COMMENT '时间维度表'
STORED AS PARQUET
LOCATION '/warehouse/gmall/dim/dim_date_info/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

**3、装载表**

时间维度表的数据其实并不是来自于业务系统，是手动写入的。并且由于时间维度表数据的可以预见性即日期年份都是固定的，无须每日导入，一般可一次性导入一年的数据（因为节假日这种数字都是前一年确定下一年的节假日）。

那么问题来了，节假日数据（一年）通过 text 文本格式，而 DIM 层存储是 PARQUET 格式，它无法识别文本格式，怎么处理？

**3.1、上传文本文件到 HDFS 上**

把 dim_date_info.txt 文本文件（描述一年节假日信息）上传到 HDFS 上临时表指定的路径：

/warehouse/gmall/tmp/tmp_dim_date_info/

**3.2、创建临时表 tmp_dim_date_info**

定义的字段和 dim_date_info 表一样；

```sql
DROP TABLE IF EXISTS tmp_dim_date_info;
CREATE EXTERNAL TABLE tmp_dim_date_info (
    `date_id` STRING COMMENT '日',
    `week_id` STRING COMMENT '周ID',
    `week_day` STRING COMMENT '周几',
    `day` STRING COMMENT '每月的第几天',
    `month` STRING COMMENT '第几月',
    `quarter` STRING COMMENT '第几季度',
    `year` STRING COMMENT '年',
    `is_workday` STRING COMMENT '是否是工作日',
    `holiday_id` STRING COMMENT '节假日'
) COMMENT '时间维度表'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/tmp/tmp_dim_date_info/';
```

**3、装载数据到 dim_date_info 表**

```sql
insert overwrite table dim_date_info select *from tmp_dim_date_info;
```

通过临时表数据作为中间表来完成装载。

**五、拉链表**

用户维度表涉及到拉链表的概念，这里先简单说一下数仓的拉链表。  

**1、拉链表概述**

拉链表的每条信息，它记录每条信息的生命周期，一旦一条记录的生命周期结束，就重新开始一条新的记录；并且把当前日期放入到生效开始日期。如果当前信息至今是有效的，则在生效结束日期中填充一个极大值（如年份日期：9999-99-99）。  

如下表：含有开始日志、结束日期

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUQ1GrnbmW0pL5ATBe7HadMLaglTKaicqM0uPSoNJ2I5maJo4FpklMYRg/640?wx_fmt=png)

**2、拉链表作用**

它能够高效的存储历史状态。适用于数据在每日发生变化，但是变化频繁又不是特别高的那种场景即数据缓慢变化。

场景：一个 APP 应用，用户信息每日发生变化，但是变化的频率并不高，并且用户数据具有一定量的规模 2000 万用户，需要保留每一个用户的历史状态。思考表结构的加载策略方式？

-   **方法一：每日全量加载，这是最简单的方式。** 

首日按照用户量 2000 万 + 做一次全量加载，然后每天新增用户后，在做一次全量加载！这样做完全可以实现需求，但是消耗硬盘资源大，保存效率会非常低。可以想象到了年底共 365 天，每天加载像滚雪球一样跑批。还有一个问题是如果用户信息很长一段时间没都没有发生变化，每天全量加载重复信息，存储效率低下。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUEzfnaanvuY5TgfdEUsBibhCkpocziaLo3L4MiaQtia1PZf0tuLynWr6tWA/640?wx_fmt=png)

-   **方法二：使用拉链表模式，记录变化数据。** 

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUBUfrm4yfzqc7vptRDK42AiaQRM6xdzjmNPNecxGIuNGejdHRnqBlaew/640?wx_fmt=png)

**2、拉链表使用**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUl1UyRoSwOdKiar0o5KxmzAvu4RwIPwsKaFSSf3kxBhk3TEFiayrjDq7g/640?wx_fmt=png)

**2.1、需要获取截止今天为止的数据（获取当前全量数据）**

这个简单，过滤条件是：end_date 为 9999-99-99；

```sql
select * from user_info where end_date = '9999-99-99'
```

**2.2、需要获取在 2019 年 1 月 1 日的数据（获取全量历史数据）**

分析：李四在 1 月 3 日姓名字段发生变化，而赵六 1 月 1 日还没有注册用户。1 月 1 日的数据只有张三、李四、王五；

SQL 语句进行过滤如下：

```cs
select * from user_info 
where start_date <= '2019-01-01' and end_date >= '2019-01-01'
```

执行结果：

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYU95zD8l3q07nJr27wrN3P85iayW1dHTfCz3miaMGKNN2wXVrOcKuib1NCA/640?wx_fmt=png)

**归纳：** 

**(1) 获取当前日期的全量数据**

**结束日期 = 9999-99-99**

**(2) 获取某个时间点的全量切片数据 (某个时间点的历史数据)**

**开始日期 >= 某个时间 and 结束日期 &lt;= 某个时间**

\***\*3、拉链表过程图 \*\***

![](https://mmbiz.qpic.cn/mmbiz_jpg/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUVry6LKyO7NCX6BubOnjyZfPfMUKyl1lfeibJiaiaBj4Sh6PcI9LqsBRNw/640?wx_fmt=jpeg)

1.  2019 年 1 月 1 日，注册三个用户分别是张三、李四、王五；
2.  2019 年 1 月 1 日的拉链表记录的 3 条数据是张三、李四、王五开始时间 1 月 1 日结束时间是 9999-99-99
3.  2019 年 1 月 2 日李四改名李小四，新增用户赵六和田七
4.  变化表是 3 条记录李小四、赵六、田七。开始时间 1 月 2 日、结束时间 9999-99-99；历史全量表是记录的张三、李四、王五开始时间 1 月 1 日结束时间是 9999-99-99 的 3 条记录。这样做合并，至今的数据是 6 条记录。

**六、用户维度表（**拉链表**）**

用户维度表是一张典型的拉链表。  

**1、用户维度表 (拉链表) 创建**

```sql
DROP TABLE IF EXISTS dim_user_info;
CREATE EXTERNAL TABLE dim_user_info(
    `id` STRING COMMENT '用户id',
    `login_name` STRING COMMENT '用户名称',
    `nick_name` STRING COMMENT '用户昵称',
    `name` STRING COMMENT '用户姓名',
    `phone_num` STRING COMMENT '手机号码',
    `email` STRING COMMENT '邮箱',
    `user_level` STRING COMMENT '用户等级',
    `birthday` STRING COMMENT '生日',
    `gender` STRING COMMENT '性别',
    `create_time` STRING COMMENT '创建时间',
    `operate_time` STRING COMMENT '操作时间',
    `start_date` STRING COMMENT '开始日期',
    `end_date` STRING COMMENT '结束日期'
) COMMENT '用户表'
PARTITIONED BY (`dt` STRING)
STORED AS PARQUET
LOCATION '/warehouse/gmall/dim/dim_user_info/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

**2、获取数据的思路**

-   假如首日 5 月 1 日，把全部数据从 ODS 层加载到 DIM 层，分区结束日期是 9999 分区；

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhB8sYGRU6z8penmiabpkFYUFBOvyKuJdu3j9CMLiawMslJBEaEQlhMtibYEP8oLLYJuaOh6nlu3Fpgw/640?wx_fmt=png)

-   第二日 5 月 2 日，一部分用户新增变化了；需要把 ODS 层新增以及变化数据，装载到 DIM 层，分区结束日期是 9999 分区它需要保证全量最新数据；另外注意的是 9999 分区一部分过期的数据（过期理解为数据发生了变化后，变化前的数据是过期数据）需要装载到变化前一日即 5 月 1 日分区 (过期的用户数据分区)。  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhVYOWpQekjYchPSZWiah7QmGWfKZibzTUnoCOmv0GcZyyb2TcribjAcBXtz8G1VBefCZBTmcJyOYNbQ/640?wx_fmt=png)

-   第三日 5 月 3 日，和 5 月 2 日类似。需要把 ODS 层新增以及变化数据，装载到 DIM 层 9999 分区。9999 分区一部分过期的数据装载到 5 月 2 日分区 (过期的用户数据分区)。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhVYOWpQekjYchPSZWiah7QmOIWN62Bj9ecHsLVRLgr9n5USc172I85aa6taksN8myHH7q0oiaeoy6Q/640?wx_fmt=png)

**3、首日装载**

**实现思路**

首日装载，比较简单。具体工作为将 ODS 层当日的全部历史用户数据一次性导入到 DIM 层拉链表 9999 分区中。比如：ODS 层，ods_user_info 表，如果初始时间是 2020 年 6 月 14 日，则 2020-06-14 的分区数据就是全部的历史用户数据，故将该分区数据导入 DIM 层拉链表的 9999 分区即可。

**SQL 语句实现**

```sql
insert overwrite table dim_user_info partition(dt='9999-99-99')
select
    id,
    login_name,
    nick_name,
    md5(name),
    md5(phone_num),
    md5(email),
    user_level,
    birthday,
    gender,
    create_time,
    operate_time,
    '2020-06-14',
    '9999-99-99'
from ods_user_info
where dt='2020-06-14';
```

**4、每日装载**

**实现思路**

再看一下每日装载的思路，上面我们已经知道了每日装载包含两部分内容即 9999 分区数据，和当日前一天 (2020 年 6 月 14 日) 过期数据。我们需要把这两部数据做 union。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhVYOWpQekjYchPSZWiah7QmMSia64EG4iaF9LLDIVb4Pga01vDRXDOO99iaoSCr15j4KEkXOkKXFFbwQ/640?wx_fmt=png)

**4.1 获取全量最新 9999 分区数据**

前一日 (2020-06-14)9999 分区和当日(2020-06-15) 的 ODS 层新增变化数据做 full join, 拿到当日 (2020-06-15) 的 999 分区最新全量数据。

**SQL 实现语句**

**(1)、先拿 2020-06-14 的 9999 分区，设置该表别名为 old**

```sql
select
    id,
    login_name,
    nick_name,
    name,
    phone_num,
    email,
    user_level,
    birthday,
    gender,
    create_time,
    operate_time,
    start_date,
    end_date
  from dim_user_info
  where dt='9999-99-99'
```

**(2)、再拿 2020-06-15 的最新变化数据，设置别名是 new**

这里 start_date 赋值是 2020-06-15，结束日期是 9999-99-99

```sql
select
    id,
    login_name,
    nick_name,
    md5(name) name,
    md5(phone_num) phone_num,
    md5(email) email,
    user_level,
    birthday,
    gender,
    create_time,
    operate_time,
    '2020-06-15' start_date,
    '9999-99-99' end_date
from ods_user_info
where dt='2020-06-15'

```

**(3)、做全外联 full outer join**

**对 old 和 new 表做**full outer join，并且把结果表的别名设置为 tmp\*\*\*\*

```sql
with
tmp as
(
  select
    old.id old_id,
    old.login_name old_login_name,
    old.nick_name old_nick_name,
    old.name old_name,
    old.phone_num old_phone_num,
    old.email old_email,
    old.user_level old_user_level,
    old.birthday old_birthday,
    old.gender old_gender,
    old.create_time old_create_time,
    old.operate_time old_operate_time,
    old.start_date old_start_date,
    old.end_date old_end_date,
    new.id new_id,
    new.login_name new_login_name,
    new.nick_name new_nick_name,
    new.name new_name,
    new.phone_num new_phone_num,
    new.email new_email,
    new.user_level new_user_level,
    new.birthday new_birthday,
    new.gender new_gender,
    new.create_time new_create_time,
    new.operate_time new_operate_time,
    new.start_date new_start_date,
    new.end_date new_end_date
  from
  (
    select
      id,
      login_name,
      nick_name,
      name,
      phone_num,
      email,
      user_level,
      birthday,
      gender,
      create_time,
      operate_time,
      start_date,
      end_date
    from dim_user_info
    where dt='9999-99-99'
  )old
  full outer join
  (
    select
      id,
      login_name,
      nick_name,
      md5(name) name,
      md5(phone_num) phone_num,
      md5(email) email,
      user_level,
      birthday,
      gender,
      create_time,
      operate_time,
      '2020-06-15' start_date,
      '9999-99-99' end_date
    from ods_user_info
    where dt='2020-06-15'
  )new
  on old.id=new.id
)
```

**(4)、从 tmp 获取全量最新数据**

**思路：tmp 表里面是 full outer join，过滤条件是：** 

**如果 new_id 为 null，则获取 old_id 的值；**\*\*****等价于**\*\*\*\***nvl\***\*(new_id,old_id)**

```cs
select
    nvl(new_id,old_id),
    nvl(new_login_name,old_login_name),
    nvl(new_nick_name,old_nick_name),
    nvl(new_name,old_name),
    nvl(new_phone_num,old_phone_num),
    nvl(new_email,old_email),
    nvl(new_user_level,old_user_level),
    nvl(new_birthday,old_birthday),
    nvl(new_gender,old_gender),
    nvl(new_create_time,old_create_time),
    nvl(new_operate_time,old_operate_time),
    nvl(new_start_date,old_start_date),
    nvl(new_end_date,old_end_date),
    nvl(new_end_date,old_end_date) dt
from tmp
```

**4.2 获取前一日过期分区数据**

前一日过期数据怎么获取呢？即表中绿色边框内容。  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhVYOWpQekjYchPSZWiah7QmUWZy4OjKiaZViamvRiagxWf1iafsnzAw9iaCAG1W8XM35vzniarV7ia3ib4btg/640?wx_fmt=png)

还是从 tmp 表里面取值，过滤条件是：  

new_id is not null and old_id is not null;

注意：

1、date_add('2020-06-15',-1) 获取前一天日志

2、cast(date_add('2020-06-15',-1) as string) 对日期做 string 类型转换，为什么要做强制转换呢？因为要做 union 操作，类型必须一致。

```sql
select
    old_id,
    old_login_name,
    old_nick_name,
    old_name,
    old_phone_num,
    old_email,
    old_user_level,
    old_birthday,
    old_gender,
    old_create_time,
    old_operate_time,
    old_start_date,
    cast(date_add('2020-06-15',-1) as string),
from tmp
where new_id is not null and old_id is not null;
```

**4.3、把两部分数据做 union 写入到拉链表中**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhVYOWpQekjYchPSZWiah7QmbXe7bsm1J2AibDRzeL8jmsdKjzroqpaVjsETtI5SbLPUrwa0IxCWTKQ/640?wx_fmt=png)

因为要写入两个分区，一个是 9999 分区，一个是 2020-6-14 分区。所以需要处理动态分区问题，怎么处理？

传入分区的时候，不能写死。这里写 dt

```sql
insert overwrite table dim_user_info partition(dt)
```

这里使用了一个方法是：最后一列数据 copy 前面一列数据，作文分区字段，但是又因为列字段不能一样，修改第二个列的别名为 dt, 分区是按照 dt 分区。

    `nvl(new_end_date,old_end_date),``nvl(new_end_date,old_end_date) dt`

```cs
 cast(date_add('2020-06-15',-1) as string),
 cast(date_add('2020-06-15',-1) as string) dt
```

**所以完整的 SQL 装载语句为**  

```sql
insert overwrite table dim_user_info partition(dt)
select
    nvl(new_id,old_id),
    nvl(new_login_name,old_login_name),
    nvl(new_nick_name,old_nick_name),
    nvl(new_name,old_name),
    nvl(new_phone_num,old_phone_num),
    nvl(new_email,old_email),
    nvl(new_user_level,old_user_level),
    nvl(new_birthday,old_birthday),
    nvl(new_gender,old_gender),
    nvl(new_create_time,old_create_time),
    nvl(new_operate_time,old_operate_time),
    nvl(new_start_date,old_start_date),
    nvl(new_end_date,old_end_date),
    nvl(new_end_date,old_end_date) dt
from tmp
union all
select
    old_id,
    old_login_name,
    old_nick_name,
    old_name,
    old_phone_num,
    old_email,
    old_user_level,
    old_birthday,
    old_gender,
    old_create_time,
    old_operate_time,
    old_start_date,
    cast(date_add('2020-06-15',-1) as string),
    cast(date_add('2020-06-15',-1) as string) dt
from tmp
where new_id is not null and old_id is not null;
```

到此我们就完成了 DIM 层用户维度表的创建和数据的加载。

DIM 层我们通过 4 种不同的加载策略完成表的创建和装载：商品维度表全量加载、地区表特殊加载、时间特殊加载、用户拉链表。

下一次我们会进入数仓当中最精彩的 DWD 层的讲解。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXd78W49nBaME6TkGc8gv8DBzMJvytIYy9Dibfsl7qq5ibATfYh9BN1xQO5qU1OejK3Gic6dfl8iafXwGg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/Kzj4wWoX_CTOG3aDFe1Y6A](https://mp.weixin.qq.com/s/Kzj4wWoX_CTOG3aDFe1Y6A)
