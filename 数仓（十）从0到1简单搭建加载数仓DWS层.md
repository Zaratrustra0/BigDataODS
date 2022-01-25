# 数仓（十）从0到1简单搭建加载数仓DWS层
[数仓（一）简介数仓，OLTP 和 OLAP](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503438&idx=1&sn=3d249b38099cdb4f3d17a7d4676f93de&chksm=f9ed3766ce9abe70d7ae724c1a83ad7669f19c4148b16bfc16f03dbc23f77b23465ca845a47d&scene=21#wechat_redirect)  

[数仓（二）关系建模和维度建模](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503465&idx=1&sn=fee932bd60e96b2814f7a257399d8170&chksm=f9ed3741ce9abe57203c7cee8a0207d9c48b53cf261aa77ff6dd9b2d65ec85543f985f01acb7&scene=21#wechat_redirect)  

[数仓（三）简析阿里、美团、网易、恒丰银行、马蜂窝 5 家数仓分层架构](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503466&idx=2&sn=aa9ab1dd24fccdb41d9e805a937c12c4&chksm=f9ed3742ce9abe54f73b86a4db0125fe36dbc8c7889cd7452456b1ed058e2c13a9f4e26eb6d7&scene=21#wechat_redirect)  

[数仓（四）数据仓库分层](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503496&idx=2&sn=3f0c6f34fe307932efeaf4c3f19b5503&chksm=f9ed37a0ce9abeb60a5417821d94f7d1bfb9ef26844b0cf44aecf17aa5e901a91f85cda7ab91&scene=21#wechat_redirect)  

[数仓 (五) 元数据管理系统解析](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503532&idx=1&sn=c996c431e25199496e464270cf9aa8eb&chksm=f9ed3784ce9abe92a7b09b535b08e8fabacdc51807e801dc4bb7806de36288c39d7ac4111e03&scene=21#wechat_redirect)  

[数仓（六）从 0 到 1 简单搭建数仓 ODS 层 (埋点日志 + 业务数据)](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503555&idx=1&sn=9718725ec437cc7e9fe78167193bb958&chksm=f9ed37ebce9abefd9c68cdf5905be86e9718bab479e1a83868d17f6159b512b64052ecff1818&scene=21#wechat_redirect)  

[数仓（七）从 0 到 1 简单搭建加载数仓 DIM 层以及拉链表处理](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503579&idx=2&sn=4eef68828d5b616631dff19319ec2b99&chksm=f9ed37f3ce9abee5023f1385c608ad67cd10b7a026e7009416c8bfa0f401e8a207a01a901395&scene=21#wechat_redirect)  

[数仓（八）从 0 到 1 简单搭建加载数仓 DWD 层（用户行为日志数据解析）](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503593&idx=1&sn=9a8444c5ed5ebc4f3d0f290a1a63879e&chksm=f9ed37c1ce9abed781e4507e8e6356fa8c4fc8fa38d33c6272c46e738832a2ffe1fc68fbe08d&scene=21#wechat_redirect)

上一次我们讲解了 DWD 层（关于用户交易等业务数据）的搭建、解析加载。这节我们讲解 DWS 层关于各个主题的加工和使用，这层是宽表聚合值，是各个事实表的聚合值。

**一、DWS 层概念回顾**

在第四章我们对 DWS 层做了概述，现在我们再来回顾一下。  

**DWS(Data Warehouse Service)**

-   使轻度汇总层，从 ODS 层中对用户的行为做一个初步的汇总，抽象出来一些通用的维度：时间、ip、id，并根据这些维度做一些统计值。
-   这里做轻度的汇总会让以后的计算更加的高效，如：统计各个主题对象计算 7 天、30 天、90 天的行为， 应对特殊需求（例如，购买行为，统计商品复购率）会快很多不必走 ODS 层反复拿数据做加工。
-   这层以分析的主题对象作为建模驱动，基于上层的应用和产品的指标需求，构建公共粒度的汇总指标事实表，以宽表化手段物理化模型。构建命名规范、口径一致的统计指标，为上层提供公共指标，建立汇总宽表、明细事实表。
-   服务于 DWT 层的主题宽表，以及一些业务明细数据。

涉及的主题包括：访客主题、用户主题、商品主题、优惠券主题、活动主题、地区主题等。  

**二、DWS 层用户主题加工**

用户主题记录某一个用户在某一天的汇总行为。

**1、首先我们看一下我们需要统计或者说加工的指标，一共是 25 个左右；**

```sql
login_count` BIGINT COMMENT '登录次数'，
`cart_count` BIGINT COMMENT '加入购物车次数',
`favor_count` BIGINT COMMENT '收藏次数',
`order_count` BIGINT COMMENT '下单次数',
`order_activity_count` BIGINT COMMENT '订单参与活动次数',
`order_activity_reduce_amount` DECIMAL(16,2) COMMENT '订单减免金额(活动)',
`order_coupon_count` BIGINT COMMENT '订单用券次数',
`order_coupon_reduce_amount` DECIMAL(16,2) COMMENT '订单减免金额(使用优惠券)',
`order_original_amount` DECIMAL(16,2)  COMMENT '订单单原始金额',
`order_final_amount` DECIMAL(16,2) COMMENT '订单总金额',
`payment_count` BIGINT COMMENT '支付次数',
`payment_amount` DECIMAL(16,2) COMMENT '支付金额',
`refund_order_count` BIGINT COMMENT '退单次数',
`refund_order_num` BIGINT COMMENT '退单件数',
`refund_order_amount` DECIMAL(16,2) COMMENT '退单金额',
`refund_payment_count` BIGINT COMMENT '退款次数',
`refund_payment_num` BIGINT COMMENT '退款件数',
`refund_payment_amount` DECIMAL(16,2) COMMENT '退款金额',
`coupon_get_count` BIGINT COMMENT '优惠券领取次数',
`coupon_using_count` BIGINT COMMENT '优惠券使用(下单)次数',
`coupon_used_count` BIGINT COMMENT '优惠券使用(支付)次数',
`appraise_good_count` BIGINT COMMENT '好评数'，
`appraise_mid_count` BIGINT COMMENT '中评数',
`appraise_bad_count` BIGINT COMMENT '差评数',
`appraise_default_count` BIGINT COMMENT '默认评价数',
`order_detail_stats` array<struct<sku_id:string,sku_num:bigint,order_count:bigint,activity_reduce_amount:decimal(16,2),coupon_reduce_amount:decimal(16,2),original_amount:decimal(16,2),final_amount:decimal(16,2)>> COMMENT '下单明细统计'
```

**2、根据指标建表**

这里我们搭建 DWS 层级表，XXXX 就是上面指标的字段。

```sql
DROP TABLE IF EXISTS dws_user_action_daycount;
CREATE EXTERNAL TABLE dws_user_action_daycount
(    XXXXX    
      ......
) 
COMMENT '每日用户行为'PARTITIONED BY (`dt` STRING)
STORED AS PARQUETLOCATION
'/warehouse/gmall/dws/dws_user_action_daycount/'
TBLPROPERTIES ("parquet.compression"="lzo");
```

**3、逐个计算指标**  

**3.1 login_count 登录次数**

这个指标简单，根据 dwd 层，dwd_page_log 表可以直接计算；

```sql
select    
  dt,    
  user_id,   
  count(*) login_count    
from dwd_page_log
where user_id is not null
and last_page_id is null
group by dt,user_id
```

**3.2 cart_count 加入购物车次数;**

**favor_count 收藏次数;**

这两个指标也是很简单，直接根据 dwd 层，dwd_action_log 表中获取

**SQL 实现**

```sql
select    
  dt,    
  user_id,    
  sum(if(action_id='cart_add',1,0)) cart_count,
  sum(if(action_id='favor_add',1,0)) favor_count
from dwd_action_log
where user_id is not null
and action_id in ('cart_add','favor_add')
group by dt,user_id
```

**3.3 order_count 下单次数；**

    **order_activity_count 订单参与活动次数；**

    **order_activity_reduce_amount 订单减免金额 (活动)；**

    **order_coupon_count 订单用券次数；**

    **order_coupon_reduce_amount 订单减免金额 (使用优惠券)；**

    **order_original_amount  订单单原始金额；**

    **order_final_amount  订单总金额；**

订单表的几个指标相对也很简单，直接根据 dwd 层，dwd_order_info 表中获取

**SQL 实现**

```sql
select    
  date_format(create_time,'yyyy-MM-dd') dt,   
  user_id,    
  count(*) order_count,    
  sum(if(activity_reduce_amount>0,1,0)) order_activity_count,    
  sum(if(coupon_reduce_amount>0,1,0)) order_coupon_count,    
  sum(activity_reduce_amount) order_activity_reduce_amount,    
  sum(coupon_reduce_amount) order_coupon_reduce_amount,    
  sum(original_amount) order_original_amount,    
  sum(final_amount) order_final_amount
from dwd_order_info
group by date_format(create_time,'yyyy-MM-dd'),user_id
```

**3.4 payment_count'支付次数',**

**payment_amount'支付金额',**

\***\*SQL 实现 \*\***

```sql
select
  date_format(callback_time,'yyyy-MM-dd') dt,
  user_id,
  count(*) payment_count,
  sum(payment_amount) payment_amount
from dwd_payment_info
group by date_format(callback_time,'yyyy-MM-dd'),user_id
```

**3.5 refund_order_count 退单次数',**

    **refund_order_num 退单件数',**

    **refund_order_amount '退单金额',**

\***\*SQL 实现 \*\***

```sql
select
    date_format(create_time,'yyyy-MM-dd') dt,
    user_id,
    count(*) refund_order_count,
    sum(refund_num) refund_order_num,
    sum(refund_amount) refund_order_amount
from dwd_order_refund_info
group by date_format(create_time,'yyyy-MM-dd'),user_id

```

**3.6refund_payment_count '退款次数',**

    **refund_payment_num '退款件数',**

    **refund_payment_amount '退款金额',**

**ri 表示上述退单中间表，rp 表示退款中间表；然后做左外链接取交集；**

\***\*SQL 实现 \*\***

```sql
select
      date_format(callback_time,'yyyy-MM-dd') dt,
      rp.user_id,
      count(*) refund_payment_count,
      sum(ri.refund_num) refund_payment_num,
      sum(rp.refund_amount) refund_payment_amount
  from
  (
      select
          user_id,
          order_id,
          sku_id,
          refund_amount,
          callback_time
      from dwd_refund_payment
  )rp
  left join
  (
      select
          user_id,
          order_id,
          sku_id,
          refund_num
      from dwd_order_refund_info
  )ri
  on rp.order_id=ri.order_id
  and rp.sku_id=rp.sku_id
  group by date_format(callback_time,'yyyy-MM-dd'),rp.user_id
  
```

**3.7coupon_get_count '优惠券领取次数',**

**coupon_using_count '优惠券使用 (下单) 次数',**

**coupon_used_count '优惠券使用 (支付) 次数',**

**SQL 实现**

```sql
select
    coalesce(coupon_get.dt,coupon_using.dt,coupon_used.dt) dt,
    coalesce(coupon_get.user_id,coupon_using.user_id,coupon_used.user_id) user_id,
    nvl(coupon_get_count,0) coupon_get_count,
    nvl(coupon_using_count,0) coupon_using_count,
    nvl(coupon_used_count,0) coupon_used_count
from
(
    select
        date_format(get_time,'yyyy-MM-dd') dt,
        user_id，
        count(*) coupon_get_count
    from dwd_coupon_use
    where get_time is not null
    group by user_id,date_format(get_time,'yyyy-MM-dd')
)coupon_get
full outer join
(
    select
        date_format(using_time,'yyyy-MM-dd') dt,
        user_id,
        count(*) coupon_using_count
    from dwd_coupon_use
    where using_time is not null
    group by user_id,date_format(using_time,'yyyy-MM-dd')
)coupon_using
on coupon_get.dt=coupon_using.dt
and coupon_get.user_id=coupon_using.user_id
full outer join
(
    select
        date_format(used_time,'yyyy-MM-dd') dt,
        user_id,
        count(*) coupon_used_count
    from dwd_coupon_use
    where used_time is not null
    group by user_id,date_format(used_time,'yyyy-MM-dd')
)coupon_used
on nvl(coupon_get.dt,coupon_using.dt)=coupon_used.dt
and nvl(coupon_get.user_id,coupon_using.user_id)=coupon_used.user_id
)
```

**3.8appraise_good_count '好评数',**

**appraise_mid_count  '中评数',**

**appraise_bad_count  '差评数',**

**这个数值直接从 dwd 层，dwd_comment_info 表里面取 sum 值**

**SQL 实现**

```sql
select
    date_format(create_time,'yyyy-MM-dd') dt,
    user_id
    sum(if(appraise='1201',1,0)) appraise_good_count,
    sum(if(appraise='1202',1,0)) appraise_mid_count,
    sum(if(appraise='1203',1,0)) appraise_bad_count,
    sum(if(appraise='1204',1,0)) appraise_default_count
from dwd_comment_info
group by date_format(create_time,'yyyy-MM-dd'),user_id
```

**4、完成首日和每日加载**

汇总 SQL 后做首日装载，分区是动态分区 dt  

```css
insert overwrite table dws_user_action_daycount partition(dt)
select
    coalesce(tmp_login.user_id,tmp_cf.user_id,tmp_order.user_id,tmp_pay.user_id,tmp_ri.user_id,tmp_rp.user_id,tmp_comment.user_id,tmp_coupon.user_id,tmp_od.user_id),
    nvl(login_count,0),
    nvl(cart_count,0),
    nvl(favor_count,0),
    XXXX,
    
```

每日装载, dt = 当前时间分区;

```sql
insert overwrite table dws_user_action_daycount partition(dt='2020-06-15')
select
    coalesce(tmp_login.user_id,tmp_cf.user_id,tmp_order.user_id,tmp_pay.user_id,tmp_ri.user_id,tmp_rp.user_id,tmp_comment.user_id,tmp_coupon.user_id,tmp_od.user_id),
    nvl(login_count,0),
    nvl(cart_count,0),
    nvl(favor_count,0),
    nvl(order_count,0),
```

这样我们就完成了关于用户主题的 DWS 层的汇聚值的计算。

**请读者参考后完成以下主题：** 

访客主题、商品主题、优惠券主题、活动主题、地区主题等，每日装载和首装载的 SQL。

* * *

**总结：**   

1.    数仓 DWS 层的概念、作用；  
2.  DWS 层的宽表聚合值是各个事实表的聚合值，怎么样加工和计算？

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXd78W49nBaME6TkGc8gv8DBzMJvytIYy9Dibfsl7qq5ibATfYh9BN1xQO5qU1OejK3Gic6dfl8iafXwGg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/tvlkh8vwOnk-TixH46-0_A](https://mp.weixin.qq.com/s/tvlkh8vwOnk-TixH46-0_A)
