# 从0到1搭建数仓DWD层案例实践
**一、DWD 层结构**

DWD 层是对用户的日志行为事实进行解析，以及对交易业务数据采用维度模型的方式重新建模（即维度退化）。

**1、回顾 DWD 层概念**

我们在来回顾一下对 DWD 层（Data Warehouse Detail）的定义：“明细粒度事实层：是以业务过程来作为建模驱动，基于每个具体的业务过程特点，构建最细粒度的明细层事实表（注意是最细粒度）。需要结合企业的数据使用特点，将明细事实表的某些重要维度属性字段做适当冗余，即宽表化处理。明细粒度事实层的表通常也被称为**逻辑事实表**。”

**2、DWD 层建模 4 步骤**  

DWD 层是事实建模层，这层建模主要做的 4 个步骤：

![](https://mmbiz.qpic.cn/mmbiz_jpg/EGZ1ibxgFrbgqHUicWfnn3YYKq0lptnUsHqHR2Qzo08d4Os5ykz0s4IV5U696obur0BDgibBkPFhsg6ZyQaCD7m3Q/640?wx_fmt=jpeg)

我们目前已经完成了：  

**2.1、选择业务过程**

选择了事实表，比如：订单事实表、支付事实表等；  

**2.2、声明粒度**

即确认每一行数据是什么，要保证事实表的最小粒度。

**2.3、确认维度**

在前面两节中我们确定了 6 个维度；比如时间、用户、地点、商品、优惠券、活动这 6 个维度。思路是其他 ODS 层表的维度需要向这 6 个维度进行退化到 DIM 层，这样做的母的是减少后期的大量表之间的 join 操作。

**![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiadSnicNe1icicia80EXibrxWMMbu7OAsMEI6M3FmrkFj7LVQsl1v75vKQANVlXYtXNssmvTYlHnibGGicWQ/640?wx_fmt=png)**

6 个维度表的退化操作其实我们在前面的第十二章节已经做了即 DIM 层。除了第 3 张表即商品维度表是 5 个表退化到 1 张表上，其他都是 1-2 张表退化到 1 张表上，相对比较简单。

**2.4、确认事实**

就是确认事实表的每张事实表的度量值。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiadSnicNe1icicia80EXibrxWMMbDeaO9Hnu8yhq9LmHqMxuibYKCeGy5roHvgq0anbNWtez7XGkzOdicU7w/640?wx_fmt=png)

下面我们根据事实表的加载方式来选择几个实战操作一下。

**二、DWD 层 - 事务型事实表**

关于事实表分类，我们在数仓关系建模和维度建模，里面说过，分为 6 类事实表。

**1、事务型事实表的概念**

适用于不会发生变化的业务。业务表的同步策略是增量同步。以每个事务或事件为单位，例如一个销售订单记录，一笔支付记录等，作为事实表里的一行数据。一旦事务被提交，事实表数据被插入，数据就不再进行更改，其更新方式为增量更新。

8 张表里面包含：支付事实表、评价事实表、退款事实表、订单明细 (详情) 事实表

**2、解析思路**

根据事实表（行），选择不同的维度（列）来建表。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbjmwR8j4s6DLtSqObkrPYKDhaNfXBRJC8fcVBwxfGNFR7fb3xibAal1V9SzyVZNEKzFFgFnvRhibX0g/640?wx_fmt=png)

**3、支付事实表（事务型事实表）**

需要时间、用户、地区三个维度，查看 ODS 层表 ods_payment_info，发现没有地区维度字段。所以通过 ods_order_info 表关联做 join 获取该字段。

**3.1、建表语句**

```sql
drop table if exists dwd_fact_payment_info;
create external table dwd_fact_payment_info (
    `id` string COMMENT 'id',
    `out_trade_no` string COMMENT '对外业务编号',
    `order_id` string COMMENT '订单编号',
    `user_id` string COMMENT '用户编号',
    `alipay_trade_no` string COMMENT '支付宝交易流水编号',
    `payment_amount`    decimal(16,2) COMMENT '支付金额',
    `subject`         string COMMENT '交易内容',
    `payment_type` string COMMENT '支付类型',
    `payment_time` string COMMENT '支付时间',
    `province_id` string COMMENT '省份ID'
) COMMENT '支付事实表表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_payment_info/'
tblproperties ("parquet.compression"="lzo");
```

**3.2、装载语句**  

province_id 省份 ID 这个字段通过 ods_order_info 表做 join 获取

```sql
SET hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
insert overwrite table dwd_fact_payment_info partition(dt='2021-05-03')
select
    pi.id,
    pi.out_trade_no,
    pi.order_id,
    pi.user_id,
    pi.alipay_trade_no,
    pi.total_amount,
    pi.subject,
    pi.payment_type,
    pi.payment_time,
    oi.province_id
from
(
    select * from ods_payment_info where dt='2021-05-03'
)pi
join
(
    select id, province_id from ods_order_info where dt='2021-05-03'
)oi
on pi.order_id = oi.id;
```

**4、退款事实表（事务型事实表）**

需要时间、用户、商品三个维度，查看 ODS 层表 ods_order_refund_info，所有字段都有，那么直接取数装载。

**4.1、创建表**

```sql
drop table if exists dwd_fact_order_refund_info;
create external table dwd_fact_order_refund_info(
    `id` string COMMENT '编号',
    `user_id` string COMMENT '用户ID',
    `order_id` string COMMENT '订单ID',
    `sku_id` string COMMENT '商品ID',
    `refund_type` string COMMENT '退款类型',
    `refund_num` bigint COMMENT '退款件数',
    `refund_amount` decimal(16,2) COMMENT '退款金额',
    `refund_reason_type` string COMMENT '退款原因类型',
    `create_time` string COMMENT '退款时间'
) COMMENT '退款事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_order_refund_info/'
tblproperties ("parquet.compression"="lzo");
```

**4.2、装载时间**

直接从 ODS 层查到数据后装载。

```sql
insert overwrite table dwd_fact_order_refund_info partition(dt='2021-05-03')
select
    id,
    user_id,
    order_id,
    sku_id,
    refund_type,
    refund_num,
    refund_amount,
    refund_reason_type,
    create_time
from ods_order_refund_info
where dt='2021-05-03';
```

**5、评价事实表、订单明细事实表（事务型事实表）**

都和上面 “退款事实表” 处理方法一样，并且所有字段均从 ODS 层 ods_comment_info 直接获取。你是否可以自己创建呢？

**三、DW 层 - 周期型快照事实表**

**1、周期型快照事实表的概念**

周期型快照事实表，表中不会保留所有数据，只保留固定时间间隔的数据，例如每天或者每月的销售额或每月的账户余额等。例如购物车，有加减商品，随时都有可能变化，但是我们更关心每天结束时这里面有多少商品，方便我们后期统计分析。相当于每天一个全量快照，业务表的同步策略是全量同步。

**2、解析思路**

每天做一次快照，导入的数据是全量，区别于事务型事实表是每天导入新增。

存储的数据比较讲究时效性，时间太久了的意义不大，可以删除以前的数据。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgugEqew0X0aHCwSLhXUBAp0ez5AZ1dEXRzq1RribZ9zq1ncp2FE4lDlzKw8pEdmKtomLpC7yThvug/640?wx_fmt=png)

**3、加购事实表（周期型快照事实表）**

**3.1、创建表结构**  

所有字段 ODS 层，fact_cart_info 表都有。  

```sql
drop table if exists dwd_fact_cart_info;
create external table dwd_fact_cart_info(
    `id` string COMMENT '编号',
    `user_id` string  COMMENT '用户id',
    `sku_id` string  COMMENT 'skuid',
    `cart_price` string  COMMENT '放入购物车时价格',
    `sku_num` string  COMMENT '数量',
    `sku_name` string  COMMENT 'sku名称 (冗余)',
    `create_time` string  COMMENT '创建时间',
    `operate_time` string COMMENT '修改时间',
    `is_ordered` string COMMENT '是否已经下单。1为已下单;0为未下单',
    `order_time` string  COMMENT '下单时间',
    `source_type` string COMMENT '来源类型',
    `srouce_id` string COMMENT '来源编号'
) COMMENT '加购事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_cart_info/'
tblproperties ("parquet.compression"="lzo");
```

**3.2、装载数据**

```sql
insert overwrite table dwd_fact_cart_info partition(dt='2021-05-03')
select
    id,
    user_id,
    sku_id,
    cart_price,
    sku_num,
    sku_name,
    create_time,
    operate_time,
    is_ordered,
    order_time,
    source_type,
    source_id
from ods_cart_info
where dt='2020-06-14';
```

**4、收藏事实表**  

收藏事实表的操作和加购事实表一样，从时间、商品、用户三个维度来创建表。

**四、DWD 层 - 累积型快照事实表**

**1、累积型快照事实表的概念**

累积型快照事实表，用于周期性发生变化的业务，即需要周期性的跟踪业务事实的变化。例如：数据仓库中可能需要累积或者存储订单从下订单开始，到订单商品被打包、运输、和签收的各个业务阶段的时间点数据来跟踪订单声明周期的进展情况。当这个业务过程进行时，事实表的记录也要不断更新。

业务表的同步策略是新增以及变化同步。

**2、解析思路**

我们以优惠券领用事实表为例。首先要了解优惠卷的生命周期：领取优惠卷——> 用优惠卷下单——> 优惠卷参与支付

累积型快照事实表使用：统计优惠卷领取次数、优惠卷下单次数、优惠卷参与支付次数。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhzpmCCuwLSGmiaKv6JeUh0uPv9ia5v2oqwliafEib4JPw012YVeFM9D1KYIicjTyV6sEyem3N0DibAjTfw/640?wx_fmt=png)

**3、优惠券领用事实表（累积型快照事实表）**

**3.1、创建表结构**

```sql
drop table if exists dwd_fact_coupon_use;
create external table dwd_fact_coupon_use(
    `id` string COMMENT '编号',
    `coupon_id` string  COMMENT '优惠券ID',
    `user_id` string  COMMENT 'userid',
    `order_id` string  COMMENT '订单id',
    `coupon_status` string  COMMENT '优惠券状态',
    `get_time` string  COMMENT '领取时间',
    `using_time` string  COMMENT '使用时间(下单)',
    `used_time` string  COMMENT '使用时间(支付)'
) COMMENT '优惠券领用事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_coupon_use/'
tblproperties ("parquet.compression"="lzo");
```

注意：这里 dt 是按照优惠卷领用时间 get_time 做为分区  

```sql
`get_time` string  COMMENT '领取时间',
`using_time` string  COMMENT '使用时间(下单)',
`used_time` string  COMMENT '使用时间(支付)'
```

**3.2 装载数据**

**首日装载分析**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbhzpmCCuwLSGmiaKv6JeUh0uDHYiaVVwWSic8IdlW9RuBvRrVPbbxGwsqf4z837ufHbGv24XA3GkvFsA/640?wx_fmt=png)

首日装载 SQL 代码, 注意是动态分区。

```sql
insert overwrite table dwd_coupon_use partition(dt)
select
    id,
    coupon_id,
    user_id,
    order_id,
    coupon_status,
    get_time,
    using_time,
    used_time,
    expire_time,
    coalesce(date_format(used_time,'yyyy-MM-dd'),date_format(expire_time,'yyyy-MM-dd'),'9999-99-99')
from ods_coupon_use
where dt='2021-05-03';
```

每日装载思路分析

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbghwypnlx60h4mHx8zGbFQx4p8LHtoymSqKr8ibUBx5zdUUiciczO88zPxHZsPwh6DaSw3tavARnGhnA/640?wx_fmt=png)

SQL 代码  

```sql
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
insert overwrite table dwd_fact_coupon_use partition(dt)
select
    if(new.id is null,old.id,new.id),
    if(new.coupon_id is null,old.coupon_id,new.coupon_id),
    if(new.user_id is null,old.user_id,new.user_id),
    if(new.order_id is null,old.order_id,new.order_id),
    if(new.coupon_status is null,old.coupon_status,new.coupon_status),
    if(new.get_time is null,old.get_time,new.get_time),
    if(new.using_time is null,old.using_time,new.using_time),
    if(new.used_time is null,old.used_time,new.used_time),
    date_format(if(new.get_time is null,old.get_time,new.get_time),'yyyy-MM-dd')
from
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from dwd_fact_coupon_use
    where dt in
    (
        select
            date_format(get_time,'yyyy-MM-dd')
        from ods_coupon_use
        where dt='2021-05-04'
    )
)old
full outer join
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from ods_coupon_use
    where dt='2021-05-04'
)new
on old.id=new.id;
```

其他类似的累积型事实表也是这个操作思路。

这样我们就完成了 DWD 层业务数据的建模和设计、搭建和使用包括简要的 SQL 代码的编写。

**现在我们来总结一下：** 

DWD 层是对事实表的处理，代表的是业务的最小粒度层。任何数据的记录都可以从这一层获取，为后续的 DWS 和 DWT 层做准备。DWD 层是站在选择好事实表的基础上，对维度建模的视角，这层维度建模主要做的 4 个步骤：选择业务过程、声明粒度、确认维度、确认事实。

* * *

**参考书籍：** 

1.  数据仓库第 4 版
2.  数据仓库工具箱
3.  DAMA 数据管理知识体系指南
4.  华为数据之道

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXd78W49nBaME6TkGc8gv8DBzMJvytIYy9Dibfsl7qq5ibATfYh9BN1xQO5qU1OejK3Gic6dfl8iafXwGg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/CbdxWmx0ZVc6luyrsCGDVA](https://mp.weixin.qq.com/s/CbdxWmx0ZVc6luyrsCGDVA)
