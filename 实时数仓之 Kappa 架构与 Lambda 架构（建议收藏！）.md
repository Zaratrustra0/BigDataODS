# 实时数仓之 Kappa 架构与 Lambda 架构（建议收藏！）
点击上方公众号进入**3 分钟秒懂大数据**主页  

然后点击右上角**“设为标星”**   

比别人更快接收硬核文章

大家好，我是土哥.

2021 年 1 月份，给大家重点分享一下离线数仓与实时数仓的内容。今天，我们先了解一下数据仓库架构的演变过程，本文主要从五个方面进行介绍

1.  数据仓库概念
2.  离线大数据架构
3.  Lambda 架构
4.  Kappa 架构
5.  Lambda 架构与 Kappa 架构的对比

## 1 数据仓库概念

数据仓库是一个`面向主题的`（Subject Oriented）、`集成的`（Integrate）、`相对稳定的`（Non-Volatile）、`反映历史变化`（Time Variant）的数据集合，用于支持管理决策。

数据仓库概念是 Inmon 于 1990 年提出并给出了完整的建设方法。随着互联网时代来临，数据量暴增，开始使用 **大数据工具** 来替代经典数仓中的传统工具。此时仅仅是工具的取代，架构上并没有根本的区别，可以把这个架构叫做**离线大数据架构**

后来随着业务实时性要求的不断提高，人们开始在 **离线大数据架构** 基础上加了一个加速层，使用流处理技术直接完成那些实时性要求较高的指标计算，这便是 Lambda 架构。

再后来，实时的业务越来越多，事件化的数据源也越来越多，实时处理从次要部分变成了主要部分，架构也做了相应调整，出现了以**实时事件处理为核心的 Kappa 架构**。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqH6gTnxQggW7ia1GLJp5sibgicic65R5wgvcnFgPUTcovrxIBsqUl6R7iaMg/640?wx_fmt=png)

## 2 离线大数据架构

数据源通过离线的方式导入到离线数仓中。下游应用根据业务需求选择直接读取 DM 或加一层数据服务，比如 MySQL 或 Redis。**数据仓库从模型层面分为三层：** 

ODS，操作数据层，保存原始数据；

DWD，数据仓库明细层，根据主题定义好事实与维度表，保存最细粒度的事实数据；

DM，数据集市 / 轻度汇总层，在 DWD 层的基础之上根据不同的业务需求做轻度汇总；

**如果要细分，分为五层：** 

**ODS 层**

`ODS 层`: Operation Data Store，数据准备区，贴源层。直接接入源数据的：业务库、埋点日志、消息队列等。ODS 层数数据仓库的准备区

**DW 数仓**

`DWD 层`:Data Warehouse Details, 数据明细层，属于业务层和数据仓库层的隔离层，把持和 ODS 层相同颗粒度。进行数据清洗和规范化操作，去空值 / 脏数据、离群值等。

`DWM 层`:Data Warehouse middle, 数据中间层，在 DWD 的基础上进行轻微的聚合操作，算出相应的统计指标

`DWS 层`:Data warehouse service, 数据服务层，在 DWM 的基础上，整合汇总一个主题的数据服务层。汇总结果一般为**宽表**，用于 OLAP、数据分发等。

**ADS 层**

`ADS 层`:Application data service, 数据应用层，存放在 ES,Redis、PostgreSql 等系统中，供数据分析和挖掘使用。

典型的数仓存储是 HDFS/Hive，ETL 可以是 MapReduce 脚本或 HiveSQL。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqoUzkrnSsVj5bUfSSMrMLWzKAvF1VfyEfzLM7CaxzFmovYPu6OL4wxw/640?wx_fmt=png)

**数仓分层的优点：** 

1.  **划清层次结构**：每一个数据分层都有它的作用域，这样我们在使用表的时候能更方便地定位和理解。
2.  **数据血缘追踪**：简单来讲可以这样理解，我们最终给下游是直接能使用的业务表，但是它的来源有很多，如果有一张来源表出问题了，我们希望能够快速准确地定位到问题，并清楚它的危害范围。
3.  **减少重复开发**：规范数据分层，开发一些通用的中间层数据，能够减少极大的重复计算。
4.  **把复杂问题简单化**。将一个复杂的任务分解成多个步骤来完成，每一层只处理单一的步骤，比较简单和容易理解。而且便于维护数据的准确性，当数据出现问题之后，可以不用修复所有的数据，只需要从有问题的步骤开始修复。
5.  **屏蔽原始数据的异常**。屏蔽业务的影响，不必改一次业务就需要重新接入数据。

## 3  Lambda 架构

随着大数据应用的发展，人们逐渐对系统的实时性提出了要求，为了计算一些实时指标，**就在原来离线数仓的基础上增加了一个实时计算的链路**，并**对数据源做流式改造**（即把数据发送到消息队列），实时计算去订阅消息队列，直接完成指标增量的计算，推送到下游的数据服务中去，由数据服务层完成离线 & 实时结果的合并。

Lambda 架构（Lambda Architecture）是由 Twitter 工程师南森 · 马茨（Nathan Marz）提出的大数据处理架构。这一架构的提出基于马茨在 BackType 和 Twitter 上的分布式数据处理系统的经验。

Lambda 架构使开发人员能够构建大规模分布式数据处理系统。它具有很好的灵活性和可扩展性，也对硬件故障和人为失误有很好的容错性。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqTKramOKbP1yOa9ThibpMTjHutfnFdAwLmuWlqsn3e5mwOK07FHWcBSQ/640?wx_fmt=png)

Lambda 架构总共由三层系统组成：`批处理层（Batch Layer）`，`速度处理层（Speed Layer`），以及用于响应查询的`服务层（Serving Layer）`。

在 Lambda 架构中，每层都有自己所肩负的任务。

**批处理层** 存储管理主数据集（不可变的数据集）和预先批处理计算好的视图。

**批处理层** 使用可处理大量数据的分布式处理系统预先计算结果。它通过处理所有的已有历史数据来实现数据的准确性。这意味着它是基于完整的数据集来重新计算的，能够修复任何错误，然后更新现有的数据视图。输出通常存储在只读数据库中，更新则完全取代现有的预先计算好的视图。

**速度处理层** 会实时处理新来的大数据。

**速度层** 通过提供最新数据的实时视图来最小化延迟。速度层所生成的数据视图可能不如批处理层最终生成的视图那样准确或完整，但它们几乎在收到数据后立即可用。而当同样的数据在批处理层处理完成后，在速度层的数据就可以被替代掉了。

**Lambda 架构问题：** 

虽然 Lambda 架构使用起来十分灵活，并且可以适用于很多的应用场景，但在实际应用的时候，Lambda 架构也存在着一些不足，主要表现在它的维护很复杂。

（1）**同样的需求需要开发两套一样的代码：这是 Lambda 架构最大的问题**，两套代码不仅仅意味着开发困难（同样的需求，一个在批处理引擎上实现，一个在流处理引擎上实现，还要分别构造数据测试保证两者结果一致），后期维护更加困难，比如需求变更后需要分别更改两套代码，独立测试结果，且两个作业需要同步上线。

（2）**资源占用增多**：同样的逻辑计算两次，整体资源占用会增多（多出实时计算这部分）

## 4  Kappa 架构

Lambda 架构虽然满足了实时的需求，但带来了更多的开发与运维工作，其架构背景是流处理引擎还不完善，流处理的结果只作为临时的、近似的值提供参考。后来随着 Flink 等流处理引擎的出现，流处理技术很成熟了，这时为了解决两套代码的问题，**LickedIn 的 Jay Kreps 提出了 Kappa 架构**。

Kappa 架构可以认为是 Lambda 架构的简化版（只要移除 lambda 架构中的批处理部分即可）。

在 Kappa 架构中，需求修改或历史数据重新处理都通过上游重放完成。

Kappa 架构最大的问题是**流式重新处理历史的吞吐能力会低于批处理**，但这个可以通过增加计算资源来弥补。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqblTOpruRLLNF4kW4dRhO7a8qRic8fM9NIbiaR1xRFRiaiaGLkcVCj61wMQ/640?wx_fmt=png)

**Kappa 架构的重新处理过程：** 

重新处理是人们对 Kappa 架构最担心的点，但实际上并不复杂：

（1）选择一个具有重放功能的、能够保存历史数据并支持多消费者的消息队列，根据需求设置历史数据保存的时长，比如 Kafka，可以保存全部历史数据。

（2）当某个或某些指标有重新处理的需求时，按照新逻辑写一个新作业，然后从上游消息队列的最开始重新消费，把结果写到一个新的下游表中。

（3）当新作业赶上进度后，应用切换数据源，读取 2 中产生的新结果表。

（4）停止老的作业，删除老的结果表。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqoQ3BGAzpkOP2z1PjcM8225s8rV8c0NUwUCLI0FYc0vFRJSqEVoANnQ/640?wx_fmt=png)

## 5 Lambda 架构与 Kappa 架构的对比

如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqYHLGWytzoujevqahyRZ8Kuk0enEFG9l4RzSiajQooIwRcN36yreI3Ag/640?wx_fmt=png)

1.  在真实的场景中，很多时候并不是完全规范的 `Lambda` 架构或 `Kappa` 架构，`可以是两者的混合`，比如大部分实时指标使用 `Kappa` 架构完成计算，少量关键指标（比如金额相关）使用 `Lambda` 架构用批处理重新计算，增加一次校对过程。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKqVREA97fgYb52Rz1Yq4ANZPYI8O6M3BZUNkSmozwjCs15mIiaNKpwCwA/640?wx_fmt=png)

2.  `Kappa` 架构并不是中间结果完全不落地，现在很多大数据系统都需要支持机器学习（离线训练），所以`实时中间结果需要落地对应的存储引擎供机器学习使用`，另外有时候还需要对明细数据查询，这种场景也需要把实时明细层写出到对应的引擎中。

![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoUUpKYfkkPOj7WvgUqpgKq4ibeZ9lSwYqoR90kp9jy4ibp8pRc6KtalM3nfVYDg5ibz5vvHibSWRInUg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icKqSso1xFnCKswBibrQ5PdkszaVbjF6MQk9qwoyKs5vEfWnj6EZrQfDIj4zly4FSlwdrFwbP6c1ezsjzWWNajXg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icKqSso1xFnCKswBibrQ5PdkszaVbjF6MQ7ls7kkqBg5TxqzzibRG4Ukf0urq3nRWIQqZ62MxhpDvENtH04fdARtg/640?wx_fmt=png)

End

![](https://mmbiz.qpic.cn/mmbiz_png/ibOfZAXfkqIz0PYmkyNblWibzfnOaZy5DiaNknXDIW1lFQ3a86GwzDHHVEibzF1YhcgUiaN8WicxfqE12Jd3Ruutj6IQ/640?wx_fmt=png)

**本文原创作者：土哥、一名大数据算法工程师。** **文章首发平台：微信公众号：** **3 分钟秒懂大数据**

**添加土哥微信，拉你进大数据交流群，和** **4000+** **大数据好友一块交流技术**

**![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoQH0Huv22tibAr1naMvfmhiaonCxj24w1MqDxGt1VUZcibMRYvOE0MibvZZZqRPpbvjoddQW2EYX2XVw/640?wx_fmt=png)**

**往期精品文章：** 

[史上最全系列 | 大数据框架知识点汇总（资源分享、还不快拿去！）](http://mp.weixin.qq.com/s?__biz=Mzg5NDY3NzIwMA==&mid=2247502915&idx=1&sn=730abe74b8965a4148b648b7e36c66c9&chksm=c01975fcf76efcea8ad84eec0d868234dab01833786dd6049ad5e379ff7300b558a6759c073e&scene=21#wechat_redirect)  

[干货总结！Kafka 面试大全（万字长文，37 张图，28 个知识点）](http://mp.weixin.qq.com/s?__biz=Mzg5NDY3NzIwMA==&mid=2247502522&idx=1&sn=c40e90bc0b3795eb31f9ec39f2f64874&chksm=c0197305f76efa139d3beb1c911707fb07014d5ee56516f7e57d512f8f3383cabcdb87105f3c&scene=21#wechat_redirect)

[史上最全干货！Flink 面试大全总结（全文 6 万字、110 个知识点、160 张图）](http://mp.weixin.qq.com/s?__biz=Mzg5NDY3NzIwMA==&mid=2247497240&idx=1&sn=954c0702a2d842f9facb4e36c8c44563&chksm=c0194fa7f76ec6b1f8b41e96ca6347b0e0da7fea3077cbed02ed862a0f3e335289eda3153924&scene=21#wechat_redirect)  

[Spark 面试干货总结！（8 千字长文、27 个知识点、21 张图）](http://mp.weixin.qq.com/s?__biz=Mzg5NDY3NzIwMA==&mid=2247500540&idx=1&sn=1ff423566e052b2b509c28811e7ea2f0&chksm=c0197b43f76ef255b14c0d8225a42cd071312a5e6948c0f6890b68671b5f553fbb347ba5ec69&scene=21#wechat_redirect) 
 [https://mp.weixin.qq.com/s/yf4NGU8oXJaNVa003sLSWw](https://mp.weixin.qq.com/s/yf4NGU8oXJaNVa003sLSWw)
