# 五分钟系列 | Hive中的count(distinct)优化
来源:[http://suo.im/6pkTWU](http://suo.im/6pkTWU)

问题描述  

* * *

COUNT(DISTINCT xxx) 在 hive 中很容易造成数据倾斜。针对这一情况，网上已有很多优化方法，这里不再赘述。   
但有时，“数据倾斜” 又几乎是必然的。我们来举个例子：

假设表 detail_sdk_session 中记录了访问某网站 M 的客户端会话信息，即：如果用户 A 打开 app 客户端，则会产生一条会话信息记录在该表中，该表的粒度为 “一次” 会话，其中每次会话都记录了用户的唯一标示 uuid，uuid 是一个很长的字符串，假定其长度为 64 位。现在的需求是：每天统计当月的活用用户数——“月活跃用户数”（当月访问过 app 就为活跃用户）。我们以 2016 年 1 月为例进行说明，now 表示当前日期。   
最简单的方法

这个问题逻辑上很简单，SQL 也很容易写出来，例如：

```sql
SELECT
  COUNT(DISTINCT uuid)
FROM detail_sdk_session t
WHERE t.date >= '2016-01-01' AND t.date <= now
```

上述 SQL 代码中，now 表示当天的日期。很容易想到，越接近月末，上面的统计的数据量就会越大。更重要的是，在这种情况下，“数据倾斜” 是必然的，因为只有一个 reducer 在进行 COUNT(DISTINCT uuid) 的计算，所有的数据都流向唯一的一个 reducer，不倾斜才怪。

## 优化 1

其实，在 COUNT(DISTINCT xxx)的时候，我们可以采用 “分治” 的思想来解决。对于上面的例子，首先我们按照 uuid 的前 n 位进行 GROUP BY，并做 COUNT(DISTINCT )操作，然后再对所有的 COUNT(DISTINCT)结果进行求和。   
我们先把 SQL 写出来，然后再做分析。

```perl
-- 外层SELECT求和
SELECT
  SUM(mau_part) mau
FROM
(
  -- 内层SELECT分别进行COUNT(DISTINCT)计算
  SELECT
    substr(uuid, 1, 3) uuid_part,
    COUNT(DISTINCT substr(uuid, 4)) AS mau_part
  FROM detail_sdk_session
  WHERE partition_date >= '2016-01-01' AND partition_date <= now
  GROUP BY substr(uuid, 1, 3)
) t;
```

上述 SQL 中，内层 SELECT 根据 uuid 的前 3 位进行 GROUP BY，并计算相应的活跃用户数 COUNT(DISTINCT)，外层 SELECT 求和，得到最终的月活跃用户数。   
这种方法的好处在于，在不同的 reducer 各自进行 COUNT(DISTINCT) 计算，充分发挥 hadoop 的优势，然后进行求和。  

注意，上面 SQL 中，n 设为 3，不应过大。   
为什么 n 不应该太大呢？我们假定 uuid 是由字母和数字组成的：大写字母、小写字母和数字，字符总数为 26+26+10=62。理论上，内层 SELECT 进行 GROUP BY 时，会有 62^n 个分组，外层 SELECT 就会进行 62^n 次求和。所以 n 不宜过大。当然，如果数据量十分巨大，n 必须充分大，才能保证内层 SELECT 中的 COUNT(DISTINCT) 能够计算出来，此时可以再嵌套一层 SELECT，这里不再赘述。

## 优化 2

其实，很多博客中都记录了使用 GROUP BY 操作代替 COUNT(DISTINCT) 操作，但有时仅仅使用 GROUP BY 操作还不够，还需要加点小技巧。   
还是先来看一下代码：

```php
--  第三层SELECT
SELECT
  SUM(s.mau_part) mau
FROM
(
  -- 第二层SELECT
  SELECT
    tag,
    COUNT(*) mau_part
  FROM
  (
      -- 第一层SELECT
    SELECT
      uuid, 
      CAST(RAND() * 100 AS BIGINT) tag  -- 为去重后的uuid打上标记，标记为：0-100之间的整数。
    FROM detail_sdk_session
    WHERE partition_date >= '2016-01-01' AND partition_date <= now
    GROUP BY uuid   -- 通过GROUP BY，保证去重
   ) t
  GROUP BY tag
) s
;
```

-   第一层 SELECT：对 uuid 进行去重，并为去重后的 uuid 打上整数标记
-   第二层 SELECT：按照标记进行分组，统计每个分组下 uuid 的个数
-   第三层 SELECT：对所有分组进行求和   
    上面这个方法最关键的是为每个 uuid 进行标记，这样就可以对其进行分组，分别计数，最后去和。如果数据量确实很大，也可以增加分组的个数。例如：CAST(RAND() \* 1000 AS BIGINT) tag

![](https://mmbiz.qpic.cn/mmbiz_jpg/TwK74MzofXdvpPkWNR4ibqdjiaTT7Nbia86hV5KpWKiaO6jPHKIAOSsNc4Cgekz0u5cLpPAxJUMQiaN2ia10E3XLricZg/640?wx_fmt=jpeg) 
 [https://mp.weixin.qq.com/s/D5-RRt6Vv_y3ie1sxz86XA](https://mp.weixin.qq.com/s/D5-RRt6Vv_y3ie1sxz86XA)
