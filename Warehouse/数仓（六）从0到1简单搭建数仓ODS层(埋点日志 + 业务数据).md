# 数仓（六）从0到1简单搭建数仓ODS层(埋点日志 + 业务数据)
[数仓（一）简介数仓，OLTP 和 OLAP](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503438&idx=1&sn=3d249b38099cdb4f3d17a7d4676f93de&chksm=f9ed3766ce9abe70d7ae724c1a83ad7669f19c4148b16bfc16f03dbc23f77b23465ca845a47d&scene=21#wechat_redirect)  

[数仓（二）关系建模和维度建模](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503465&idx=1&sn=fee932bd60e96b2814f7a257399d8170&chksm=f9ed3741ce9abe57203c7cee8a0207d9c48b53cf261aa77ff6dd9b2d65ec85543f985f01acb7&scene=21#wechat_redirect)  

[数仓（三）简析阿里、美团、网易、恒丰银行、马蜂窝 5 家数仓分层架构](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503466&idx=2&sn=aa9ab1dd24fccdb41d9e805a937c12c4&chksm=f9ed3742ce9abe54f73b86a4db0125fe36dbc8c7889cd7452456b1ed058e2c13a9f4e26eb6d7&scene=21#wechat_redirect)  

[数仓（四）数据仓库分层](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503496&idx=2&sn=3f0c6f34fe307932efeaf4c3f19b5503&chksm=f9ed37a0ce9abeb60a5417821d94f7d1bfb9ef26844b0cf44aecf17aa5e901a91f85cda7ab91&scene=21#wechat_redirect)  

[数仓 (五) 元数据管理系统解析](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503532&idx=1&sn=c996c431e25199496e464270cf9aa8eb&chksm=f9ed3784ce9abe92a7b09b535b08e8fabacdc51807e801dc4bb7806de36288c39d7ac4111e03&scene=21#wechat_redirect)  

最近工作一直忙着，报名参加了上海地区观安杯 CTF 的比赛。第一次参加比预期好，拿了银行行业分行二等奖（主要是团队给力！）。

此外还在搞 DAMA 中国 CDGA 考证的事情。9 月 5 日考试发挥正常，感觉应该是 PASS 可以拿到证书，数据治理证书我感觉最近几年会很火爆！就像十年前的项目管理证书 PMP。数据管理，数据治理方向必定火爆！这次 9 月成绩北京、广州、深圳早一天出成绩，一些大佬特别是彭友会的已经发喜报了！

可惜上海今天中午才能出结果！中午吃饭的时候邮件推送消息显示 81 分！成功 get 证书，也是预料之中吧！

* * *

开始继续数仓的内容。通过前面（七）（八）（九）（十）4 次内容分享，我们讲解了 hive 原理，搭建了 hive 数仓环境、配置了 hive on Spark 的引擎、并且讨论了 Yarn 资源调度器的概念，hive 元数据管理 metastore 的概念、配置方式、以及 hiveserver2 服务。

这节比较实战，把已经存储 HDFS 上的埋点日志信息，以及 mysql 数据源过来的业务数据，通过数仓的建表并且把两部分数据加载到数仓的 ODS 层。

**一、ODS 层数据搭建前提工作**

ODS 搭建的前提条件是业务系统（如：javaweb）前端的埋点日志信息、以及业务数据（存储在 mysql）以及采集到了 HDFS 平台上了。假设现在业务系统 (如：javaweb) 的埋点日志信息、以及业务交易数据（存储在 mysql）已经都采集到了 HDFS 平台。

[Flume NG：Flume 发展史上的第一次革命](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247487188&idx=1&sn=7062e3837532cafa73d379ff2f4c1fc3&chksm=f9eef7fcce997eeaf4d35437a4abe2ae3b8698033e2a589848e3a790cbac00b1f980296168b1&scene=21#wechat_redirect)  

[Flume+Kafka 双剑合璧玩转大数据平台日志采集](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247501975&idx=1&sn=951f8bc29616678f09d84d1c9ee97a13&chksm=f9ed31bfce9ab8a967bde752283dc23353f4cc53945b096d108d389dea72e16faf5018d4df64&scene=21#wechat_redirect)  

[埋点治理：如何把 App 埋点做到极致？](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247492315&idx=2&sn=0d55f52518cff3c5fd0c8d9caac6c76a&chksm=f9ed1bf3ce9a92e539ec556a2430dca569c830d04d3dac9861947fab43859b3f1a82745fd64f&scene=21#wechat_redirect)  

[知乎数据埋点方案](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247487158&idx=1&sn=d9981190ebed0ed9ff685fe81778a62c&chksm=f9eef79ece997e88dc2f4f4358f1c847b3c56175cf1c8e13d89449f7cdff5a33af1915318a69&scene=21#wechat_redirect)  

**二、DataGrip 利器使用**

我们在操作数仓表、数据等需要使用一些工具，这里推荐使用 JetBrains 的 DataGrip 工具。

**1、配置 Data Source 界面**

-   ##### 添加数据源，选择 Apache Hive

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2jEog5b8fKN5mPFaxKykViceSzwxicRL5rjkX9CI1Ht5V4zib3KPI0YWQA/640?wx_fmt=png)

-   配置相关信息

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav24cB4ZicBiceRqcU9qO2ELyFPr1IPMqwBZpVLfibviatmraicf4QOKrQBDAA/640?wx_fmt=png)

-   先启动 hiveServer2 服务

       在做测试连接 TestConnection 前，先启动 hiveServer2 服务。

**注意：** 并且有 4 个 hive session id 出现才点击 “TestConnection” 按钮。否则出现 connectied failure

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav22u3f6fZW2xDp9BG8guTTdGCtROjE2Oab8U1EJuJIjFEibibWFbdGbibSw/640?wx_fmt=png)

-   ##### 下载相应的驱动（自动下载）

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2PkImrNXK42VSYYJMegCFjrDQaxe0ofaiaBicjPhYYDia3zvcyvCuMtUjQ/640?wx_fmt=png)

**2、解决用户访问拒绝问题**

##### 这个报错界面，困扰了我一些时间，远程访问被拒绝，原因在于 hive 和 hdfs 以及 linux 之间的权限问题。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2daXxVXibfjUtXbCxTzYoDZo91020RGy67vspfNxUpg1HOP9vbibh7qrg/640?wx_fmt=png)

**解决办法如下：** 

-   **配置 hdfs 的 core-site.xml 文件，配置用户；**

 <property>  
 <name>hadoop.proxyuser.root.hosts</name>  
 <value>*</value>  
 </property>  
 <property>  
 <name>hadoop.proxyuser.root.groups</name>  
 <value>*</value>  
</property>

测试后发现还是不行，报错依然是访问拒绝！qiusheng 和 root 用户都不行

-   **思考配置 hive 的 hive-site.xml 文件**

<property>  
    <name>hive.server2.enable.doAs</name>  
    <value>FALSE</value>  
    <description>  
        Setting this property to true will have HiveServer2 execute  
        Hive operations as the user making the calls to it.  
    </description>  
</property> 

<property>  
 <name>dfs.permissions.enabled</name>  
 <value>true</value>  
 <description>  
 If "true", enable permission checking in HDFS.  
 If "false", permission checking is turned off,  
 but all other behavior is unchanged.  
 Switching from one parameter value to the other does not change the                mode,owner or group of files or directories.  
 </description>  
</property>

-   **重启 Hadoop 集群**

    修改配置后需要重启 hadoop。
-   **测试连接**

    显示测试成功！说明主要问题在于 linux 用户访问 HDFS 权限问题。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2WicqyFpamQz4DAVQIW32zHbRUFkbatlO6CbfBM5vp2NkDbTbTu3aAdg/640?wx_fmt=png)

这样 Datagrip 就正常连接 hive 了。  

**3、查看 hive 仓库里面的数据库和表以及内容**

发现 student 里面已经有 id 和 name 字段了

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2PSjngyQE279yic6EzcF2orzEgGx6bL4bTKIPtBzZz07icPHkOlibiav5Aw/640?wx_fmt=png)

当然，也可以通过 beeline 来通过 SQL 查询。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2ZzxUK62uFR4TajSWjsVmKVibZ4LxpPHy7Yt1E0C7VQ69c6NXXML9GKQ/640?wx_fmt=png)

**三、ODS 层（**埋点日志处理**）**

首先我们回顾一下 ODS 层的主要作用和特点：

**ODS(Operation Data Store)**

这层字面意思叫操作型数据存储，存储来自多个业务系统、前端埋点、爬虫获取等的一系列数据源的数据。

-   又叫 “**贴源层**”，这层保持数据原貌不做任何修改，保留历史数据，储存起到备份数据的作用。
-   数据一般采用 lzo、Snappy、parquet 等压缩格式，减少磁盘存储空间（例如：原始数据 10G，根据算法可以压缩到 1G 左 右）。
-   创建分区表，防止后续的全表扫描，减少集群资源访问数仓的压力，一般按天存储在数仓中。

有些公司还会把 ODS 层再细分两层：

**STG**：数据缓冲层，存原始数据；

**ODS**：对 STG 层简单清洗后的数据。

**1、创建前端埋点日志 ods_log 表**

前端埋点日志信息都是 JSON 格式形式主要包括两方面：

（1）启动日志；（2）事件日志：

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav28pm6eQQicAQHhF7IG0GGV3peHlSXEg9bVQOElTs5w4n8Ukhtn0Fex8A/640?wx_fmt=png)

我们把前端整个 1 条记录埋点日志，当一个字符串来处理，传入到 hive 数据库。

**创建 HQL 语句如下：**   

在 DataGrip 里面执行

```sql
drop table if exists ods_log;
create external table ods_log(line string)
partitioned by (dt string)
Stored as
    inputformat 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
    outputformat 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
Location '/warehouse/gmail/ods/ods_log';
```

**添加 lzo 索引**

还需要在 hive 文件上，添加 lzo 索引，要不然无法支持切片操作。

具体做法是通过 hadoop 自带的 jar 包在 hadoop 集群命令行里面执行：

```javascript
hadoop jar /opt/module/hadoop-3.1.4/share/hadoop/common/hadoop-lzo-0.4.20.jar 
  com.hadoop.compression.lzo.DistributedLzoIndexer 
  -Dmapreduce.job.queuename=hive 
  /warehouse/gmail/ods/ods_log/dt=2021-05-01
```

\***\*2、执行完，观察 Browser Director\*\***

发现已经多了 lzo.index 索引  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2WJV3pPY7I1pLgDzf5jvhZhwlgiaS8E79iavGZQIqJbddDzAYg7hoOKtA/640?wx_fmt=png)

**3、加载读取 HDFS 数据到 hive**

```sql
load data inpath '/origin_data/gmall/log/topic_log/2021-05-01'
    into table ods_log partition (dt='2021-05-01');
```

源路径在

HDFS 上 / origin_data/gmall/log/topic_log/2021-05-01

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2ONHiajtmwYiaCbKlHwBEPjibickeaxaBughvGGpdwvIHMCdlC8lyibf3bPg/640?wx_fmt=png)

加载到 hive，路径是

/warehouse/gmail/ods/ods_log/dt=2021-05-01

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2ELbhI9ppSgIECK0TuOHJIcgCxnNaCb6cJXtEFy3y60zsYJhPzQDGkQ/640?wx_fmt=png)

load 做的是剪切的操作！  

**4、查看 hive 库数据是否已经加载成功？**  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2gZz5bZHFhF1ibaia2uSTwqFzkRlf1uokgaPQfjoM5YhyqlBmeNXa5OoQ/640?wx_fmt=png)

这样我们就完成了 ODS 层前端埋点日志的处理。  

**四、ODS 层（业务数据**处理**）**

现在我们来处理 ODS 层业务数据，先需要回顾一下 ODS 层当时建表的表结构关系。

**1、业务表逻辑结构关系**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2cpsYgu4Esp5FFQBoicwgaVsAumeefaTeBicMpDa3EItTLdQD6af1enWA/640?wx_fmt=png)

其中有颜色填充的代表：事实表；  

其他：代表维度表；  

**2、HDFS 文件对应 hive 表结构关系**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2SsjZOFHYgjrdsI6eE6apRXlmv27vfENdm4LCtkzGMxnjueEMdMlh9g/640?wx_fmt=png)

源业务系统 javaweb 项目中的数据存储在 mysql 里面，通过 sqoop 采集到 HDFS 对应的文件，需要在 hive 数仓里面设计外部表与其一一对应；

（1）考虑到分区 partitioned by 时间

（2）考虑到 lzo 压缩，并且需要 lzo 压缩支持切片的话，必须要添加 lzo 索引

（3）mysql 数据库的表通过 sqoop 采集到 HDFS，用的是\\t 作为分割，那数仓里面 ODS 层也需要\\t 作为分割；

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav2SWMgEPsBu2fQCALJ7oO1fibQKUF2hsOmBficVgHnrxZtnHCfASWicaicCQ/640?wx_fmt=png)

**3、创建 ODS 层业务表**  

这里我们业务系统 javaweb 项目一共有 23 张表，数仓 hive 我们这里只演示创建三张代表性的表。因为 23 张创建表的结构大都一样，这里只有 3 种数据同步策略的方式。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav24XbTicBNgg43tChxy2aztdb1zfX9Kwk4LXglVPD0fQ1jVcDGacxQqdA/640?wx_fmt=png)

-   ##### 订单表 ods_order_info

    ##### 表数据同步更新策略：增量

```sql
drop table if exists ods_order_info;
create external table ods_order_info (
    `id` string COMMENT '订单号',
    `final_total_amount` decimal(16,2) COMMENT '订单金额',
    `order_status` string COMMENT '订单状态',
    `user_id` string COMMENT '用户id',
    `out_trade_no` string COMMENT '支付流水号',
    `create_time` string COMMENT '创建时间',
    `operate_time` string COMMENT '操作时间',
    `province_id` string COMMENT '省份ID',
    `benefit_reduce_amount` decimal(16,2) COMMENT '优惠金额',
    `original_total_amount` decimal(16,2)  COMMENT '原价金额',
    `feight_fee` decimal(16,2)  COMMENT '运费'
) COMMENT '订单表'
PARTITIONED BY (`dt` string) -- 按照时间创建分区
row format delimited fields terminated by '\t' -- 指定分割符为\t 
STORED AS -- 指定存储方式，读数据采用LzoTextInputFormat；输出数据采用TextOutputFormat
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmail/ods/ods_order_info/'; -- 指定数据在hdfs上的存储位置
```

-   ##### SKU 商品表 ods_sku_info 表

    ##### 数据同步更新策略：全量

```sql
drop table if exists ods_sku_info;
create external table ods_sku_info( 
    `id` string COMMENT 'skuId',
    `spu_id` string   COMMENT 'spuid', 
    `price` decimal(16,2) COMMENT '价格',
    `sku_name` string COMMENT '商品名称',
    `sku_desc` string COMMENT '商品描述',
    `weight` string COMMENT '重量',
    `tm_id` string COMMENT '品牌id',
    `category3_id` string COMMENT '品类id',
    `create_time` string COMMENT '创建时间'
) COMMENT 'SKU商品表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmail/ods/ods_sku_info/';
```

-   ##### **省份表 ods_base_province**

    ##### 数据同步更新策略：特殊一次性全部加载

    ##### 不做分区 partitioned BY

```sql
drop table if exists ods_base_province;
create external table ods_base_province (
    `id`   bigint COMMENT '编号',
    `name`        string COMMENT '省份名称',
    `region_id`    string COMMENT '地区ID',
    `area_code`    string COMMENT '地区编码',
    `iso_code` string COMMENT 'iso编码,superset可视化使用'
)  COMMENT '省份表'
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmail/ods/ods_base_province/';
```

**4、加载数据**

```sql
load data 
  inpath '/origin_data/gmall/db/order_info/XXXX-XX-XX' 
  OVERWRITE into table gmail.ods_order_info partition(dt='XXXX-XX-XX');
```

显示如下：  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgQdiafLbcgtRlZb7M6ickiav21ayOqRTge1nMMerTs9qn91pbzQJzuekh5JJzNibfdM9RwkgyYpta09Q/640?wx_fmt=png)

然后把其他 20 张表也类似这样操作，我们就完成了整个 ODS 层的数据处理。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXd78W49nBaME6TkGc8gv8DBzMJvytIYy9Dibfsl7qq5ibATfYh9BN1xQO5qU1OejK3Gic6dfl8iafXwGg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s?\_\_biz=MzUyMDA4OTY3MQ==&mid=2247503555&idx=1&sn=9718725ec437cc7e9fe78167193bb958&chksm=f9ed37ebce9abefd9c68cdf5905be86e9718bab479e1a83868d17f6159b512b64052ecff1818&scene=21#wechat_redirect](https://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503555&idx=1&sn=9718725ec437cc7e9fe78167193bb958&chksm=f9ed37ebce9abefd9c68cdf5905be86e9718bab479e1a83868d17f6159b512b64052ecff1818&scene=21#wechat_redirect)
