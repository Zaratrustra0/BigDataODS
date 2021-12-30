# 数仓|如何使用RFM模型进行用户分层
在适当、有效的商务智能环境中，数据分析的质量必须得到保障。而确保数据分析质量的第一步就是根据问题需求从海量数据中提炼出真正所需的数据，因为这是发挥数据价值很重要的一个方面。通过数据的分析与可视化呈现可以更加直观的提供数据背后的秘密，从而辅助业务决策，实现真正的数据赋能业务。本文主要介绍在用户分层和用户标签中常常使用的一个模型——RFM 模型。

### 基本概念

RFM 模型是在客户关系管理 (CRM) 中常用到的一个模型, RFM 模型是衡量客户价值和客户创利能力的重要工具和手段。该模型通过一个客户的近期购买行为、购买的总体频率以及花了多少钱三项指标来描述该客户的价值状况。

RFM 模型较为动态地层示了一个客户的全部轮廓，这对个性化的沟通和服务提供了依据，同时，如果与该客户打交道的时间足够长，也能够较为精确地判断该客户的长期价值 (甚至是终身价值)，通过改善三项指标的状况，从而为更多的营销决策提供支持。

RFM 由三要素构成，即 R – Recency 最近一次活动，F – Frequency 用户活动频率，M – Monetary 消费金额，每个要素都代表着用户某种重要的行为特征。RFM 衡量数据是分析用户行为的重要指标，用户活动频率 F 和消费金额 M 代表了用户终生价值，最近一次活动 R 则代表了用户留存率以及用户参与度。

在 RFM 模式中，包括三个关键的因素，分别为：

-   R(Recency)：表示客户最近一次购买的时间有多远，即最近的一次消费，消费时间越近的客户价值越大
-   F(Frequency)：表示客户在最近一段时间内购买的次数，即消费频率，经常购买的用户也就是熟客，价值肯定比偶尔来一次的客户价值大
-   M (Monetary)：表示客户在最近一段时间内购买的金额，即客户的消费能力，通常以客户单次的平均消费金额作为衡量指标，消费越多的用户价值越大。

![](https://mmbiz.qpic.cn/mmbiz_jpg/PL10rfzHicsj7F6d8ODIGibZFfE9G6JnkZFX1eSJOodlg6m8SnBpnribphWyDO3C8VBl0S1WqhAyQ4AVdZMOZNIfw/640?wx_fmt=jpeg)

最近一次消费、消费频率、消费金额是测算消费者价值最重要也是最容易的方法，这充分的表现了这三个指标对营销活动的指导意义。而其中，最近一次消费是最有力的预测指标。

通过上面分析可以对客户群体进行分类：

| 客户类型与等级 | R   | F   | M   | 客户特征 |
| ------- | --- | --- | --- | ---- |

| 重要价值客户  
(A 级 / 111) | 高 (1) | 高 (1) | 高 (1) | 最近消费时间近、消费频次和消费金额都很高 |
| 重要发展客户  
(A 级 / 101) | 高 (1) | 低 (0) | 高 (1) | 最近消费时间较近、消费金额高，但频次不高，忠诚度不高，很有潜力的用户，必须重点发展 |
| 重要保持客户  
(B 级 / 011) | 低 (0) | 高 (1) | 高 (1) | 最近消费时间交远，消费金额和频次都很高。 |
| 重要挽留客户  
(B 级 / 001) | 低 (0) | 低 (0) | 高 (1) | 最近消费时间较远、消费频次不高，但消费金额高的用户，可能是将要流失或者已经要流失的用户，应当基于挽留措施。 |
| 一般价值客户  
(B 级 / 110) | 高 (1) | 高 (1) | 低 (0) | 最近消费时间近，频率高，但消费金额低，需要提高其客单价。 |
| 一般发展客户  
(B 级 / 100) | 高 (1) | 低 (0) | 低 (0) | 最近消费时间较近、消费金额，频次都不高。 |
| 一般保持客户  
(C 级 / 010) | 低 (0) | 高 (1) | 低 (0) | 最近消费时间较远、消费频次高，但金额不高。 |
| 一般挽留客户  
(C 级 / 000) | 低 (0) | 低 (0) | 低 (0) | 都很低 |

### 通过 RFM 模型能得到什么信息

-   谁是最佳用户？
-   哪些用户即将流失？
-   谁有潜力成为有价值用户？
-   哪些用户可以留存？

### 数据分析案例

-   效果图

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsjlv46XvXmiaDYmHx2zDY8fVmwxx66RADaJ9gokZvPddZ5byllADWFDpKn6IHWfpLSvKnE8PAN8MNA/640?wx_fmt=png)

-   实现步骤

假设有如下的样例数据：

| 客户名称             | 日期         | 消费金额  | 消费数量 |
| ---------------- | ---------- | ----- | ---- |
| 上海 \*\*\*\* 有限公司 | 2020-05-20 | 76802 | 2630 |

需要将数据集加工成如下格式：

![](https://mmbiz.qpic.cn/mmbiz_png/PL10rfzHicsjlv46XvXmiaDYmHx2zDY8fVkxQ7FuF0lKFz4DUBGn8d39V3QFTLbZ9MeiajDXibw0kficDSG9SA1d7uA/640?wx_fmt=png)

具体 SQL 实现

```sql
SELECT 
     customer_name,
     customer_avg_money,
     customer_frequency, 
     total_frequency,
     total_avg_frequency, 
     customer_recency_diff, 
     total_recency, 
     monetary,
     frequency, 
     recency, 
     rfm, 
     CASE
        WHEN rfm = "111" THEN "重要价值客户"
        WHEN rfm = "101" THEN "重要发展客户"
        WHEN rfm = "011" THEN "重要保持客户"
        WHEN rfm = "001" THEN "重要挽留客户"
        WHEN rfm = "110" THEN "一般价值客户"
        WHEN rfm = "100" THEN "一般发展客户"
        WHEN rfm = "010" THEN "一般保持客户"
        WHEN rfm = "000" THEN "一般挽留客户"
    END AS rfm_text
FROM
  (SELECT 
       customer_name,
       customer_avg_money,
       customer_frequency, 
       total_avg_money ,
       total_frequency,
       total_frequency / count(*) over() AS total_avg_frequency, 
       customer_recency_diff, 
       avg(customer_recency_diff) over() AS total_recency, 
       if(customer_avg_money > total_avg_money,1,0) AS monetary, 
       if(customer_frequency > total_frequency / count(*) over(),1,0) AS frequency, 
       if(customer_recency_diff > avg(customer_recency_diff) over(),0,1) AS recency, 
       concat(if(customer_recency_diff > avg(customer_recency_diff) over(),0,1),if(customer_frequency > total_frequency / count(*) over(),1,0),if(customer_avg_money > total_avg_money,1,0)) AS rfm
   FROM
     (SELECT 
           customer_name, 
           max(customer_avg_money) AS customer_avg_money , 
           max(customer_frequency) AS customer_frequency, 
           max(total_avg_money) AS total_avg_money ,
           max(total_frequency) AS total_frequency,
           datediff(CURRENT_DATE,max(customer_recency)) AS customer_recency_diff 
      FROM
        (SELECT 
               customer_name, 
               avg(money) over(partition BY customer_name) AS                customer_avg_money, 
               count(amount) over(partition BY customer_name) AS customer_frequency, 
               avg(money) over() AS total_avg_money,
               count(amount) over() AS total_frequency, 
               max(sale_date) over(partition BY customer_name) AS customer_recency 

       FROM customer_sales) t1
       GROUP BY customer_name)t2) t3
```

通过上面的分析，可以为相对应的客户打上客户特征标签，这样就可以针对某类客户指定不同的营销策略。  

结论  

-   如果一家公司「重要价值」的客户不多，其他都是价值很低的「一般保持」客户，表示客户结构很不健康，无法承受客户流失的风险。
-   「重要保持客户」是指最近一次的消费时间离现在较久，但消费频率和金额都很高的客户，企业要主动和他们保持联系。
-   「重要发展客户」是最近一次消费时间较近、消费金额高，但频率不高、忠诚度不高的潜力客户。企业必须严格检视每一次服务体验，是否让客户非常满意。
-   「重要挽留客户」则是最近一次消费时间较远、消费频率不高，但消费金额高的用户。企业要主动厘清久未光顾消费的原因，避免失去这群客户。

要减少重要挽留客户，促活重要保持客户，挖掘重要发展客户，才能产生源源不绝的重要价值客户。

总结

营销人员利用 RFM 分析能够快速地将用户细分成同类群组，并针对这些用户采取不同的个性化营销策略，从而提高用户的参与度和留存率。值得注意的是，不同的行业的数据特点和用户行为特点是不尽相同的，所以在实际的操作过过程中，会制定符合自己公司业务特点的 RFM 规则，但是基本的思路都是一致的。 
 [https://mp.weixin.qq.com/s/Dj-SjOHGmtgnU3V63OkzsQ](https://mp.weixin.qq.com/s/Dj-SjOHGmtgnU3V63OkzsQ)
