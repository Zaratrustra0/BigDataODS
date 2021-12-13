# 我们在学习Flink的时候，到底在学习什么？
> 这是一篇指南和大纲性质的文章。
>
> Flink 经过 2 年左右的官方和社区的大规模推广，现在国内的一众大小企业基本都在使用。

后台很多小伙伴都在问 Flink 的学习路径，那么我们在学习 Flink 的时候，到底重点学习哪些东西呢？

从我个人的学习经历来看，在学习任何一个新出现的框架或者技术点的时候，核心方法就是：【先看背景，整理大纲，逐个击破】。

#### 核心背景和论文

知其然且知其所以然。

Flink 框架自提出到实现，是有深厚的理论作为背书的，其中又以**《Lightweight Asynchronous Snapshots for Distributed Dataflows》**最为核心，本文提出了一种轻量级的异步分布式快照 (Asynchronous Barrier Snapshot, 简称 ABS) 方法，既支持无环图，又支持有环图，而且可以做到线性扩展。

传统的流式计算由算子节点和连接算子的数据管道组成，传统的分布式快照方案就像拍照片一样，把每个算子内的 state 状态和彼此相连管道中的数据都保存下来。ABS 方案的提出对传统的流式计算引擎的设计方案可以说是颠覆性的。

另外一篇是**《Apache FlinkTM: Stream and Batch Processing in a Single Engine》**，严格来说这篇论文是一篇 Flink 的概要设计文档。可以当成一个技术文档来看，对于很多我们难以理解的设计都有很大的帮助。

此外，**《The world beyond batch: streaming 101/102》**是由 Google 的大神 Tyler Akidau 写的两篇文章。第 1 篇文章，在深入了解时间，对批处理和流式数据常见处理方式进行高阶阐述之前，介绍一些基本的背景知识和术语。第 2 篇文章主要介绍包括 Google Dataflow 大数据平台使用的统一批量 + 流式传输模式。这两篇文章对于我们理解 Flink 中的时间、窗口、触发器等等的实现有十分重要的指导意义。并且作者还在 YouTube 上传了动画，可谓用心良苦，各位各凭本事翻墙去找吧。

#### 基础概念

如果你读过了上面的核心论文，那么你就会对 Flink 中一些概念的提出有更为深刻的理解，在 Flink 这个框架中，延用了很多之前 Hadoop 体系或者 Spark 中的设计概念。

这里面最核心的概念包括：

-   流 (无界流、有界流) 和转换
-   State 和 checkpoint
-   并行度
-   Workers,Slots,Resources
-   时间和窗口
-   ...

此外还有一些例如：分布式缓存、重启策略等。当然在 Flink SQL 中还有一些特定的概念，例如：Window Aggregate、Non-window Aggregate、Group Aggregate、Append Stream 等等。

上面这些核心概念是我们学习 Flink 框架的基础。

#### 核心模块

我们直接把官网的图拿过来，这张图上基本就是 Flink 框架整体的设计思想和模块要拆分：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icEQl4tLYWPuVFpdn9UVkmkVT7lErxMjfzOOeOUA1rib5zaRLu5ibhBIyA/640?wx_fmt=png)

那么对于我们数据开发来说，里面最核心的包分配我们看一下 Github：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icVmtHowbm3PxItjQGhzSevXFHT2ShrwhRX6A9BEbv7BKZET8Y1wNdeQ/640?wx_fmt=png)

我把其中核心的实现用红框圈出来了。

#### 源码阅读

根据上面我们选择的核心模块，我们大概可以得知，整个 Flink 的核心实现包含了：

-   Flink 基本组件和逻辑计划：介绍了 Flink 的基本组件、集群构建的过程、以及客户端逻辑计划的生成过程
-   Flink 物理计划生成：介绍了 Flink JobManager 对逻辑计划的运行时抽象，运行时物理计划的生成和管理等
-   Jobmanager 基本组件和 TaskManager 的基本组件
-   Flink 算子的生命周期：介绍了 Flink 的算子从构建、生成、运行、及销毁的过程
-   Flink 网络栈：介绍了 Flink 网络层的抽象，包括中间结果抽象、输入输出管理、BackPressure 技术、Netty 连接等
-   Flink 的水印和 Checkpoint
-   Flink-scheduler：介绍 Flink 的任务调度算法及负载均衡
-   Flink 对用户代码异常处理：介绍作业的代码异常后 Flink 的处理逻辑，从而更好的理解 Flink 是如何保证了 exactly-once 的计算语义
-   Flink Table/SQL 执行流程、Flink 和 Hive 的集成等

#### 行业应用

Flink 的行业应用，我们在之前的文章中反复提及。主要的应用：

-   实时数据计算

如果你对大数据技术有所接触，那么下面的这些需求场景你应该并不陌生：

> 各大电商每年双十一都会直播，实时监控大屏是如何做到的？
>
> 公司想看一下大促中销量最好的商品 TOP5？
>
> 我是公司的运维，希望能实时接收到服务器的负载情况？

我们可以看到，数据计算场景需要从原始数据中提取有价值的信息和指标，比如上面提到的实时销售额、销量的 TOP5，以及服务器的负载情况等。

传统的分析方式通常是利用批查询，或将事件（生产上一般是消息）记录下来并基于此形成有限数据集（表）构建应用来完成。为了得到最新数据的计算结果，必须先将它们写入表中并重新执行 SQL 查询，然后将结果写入存储系统比如 MySQL 中，再生成报告。

Apache Flink 同时支持流式及批量分析应用，这就是我们所说的批流一体。Flink 在上述的需求场景中承担了数据的实时采集、实时计算和下游发送。

-   **实时数据仓库和 ETL**

ETL（Extract-Transform-Load）的目的是将业务系统的数据经过抽取、清洗转换之后加载到数据仓库的过程。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6iciaPUEOotgELtCx6SWPjEHEQEZkJRYuZ9kqfz5kiaNxFC1fXpic8ZUj4cg/640?wx_fmt=png)

传统的离线数据仓库将业务数据集中进行存储后，以固定的计算逻辑定时进行 ETL 和其他建模后产出报表等应用。离线数据仓库主要是构建 T+1 的离线数据，通过定时任务每天拉取增量数据，然后创建各个业务相关的主题维度数据，对外提供 T+1 的数据查询接口。

上图展示了离线数据仓库 ETL 和实时数据仓库的差异，可以看到离线数据仓库的计算和数据的实时性均较差。数据本身的价值随着时间的流逝会逐步减弱，因此数据发生后必须尽快的达到用户的手中，实时数仓的构建需求也应运而生。

实时数据仓库的建设是 “数据智能 BI” 必不可少的一环，也是大规模数据应用中必然面临的挑战。

Flink 在实时数仓和实时 ETL 中有天然的优势：

-   状态管理，实时数仓里面会进行很多的聚合计算，这些都需要对于状态进行访问和管理，Flink 支持强大的状态管理
-   丰富的 API，Flink 提供极为丰富的多层次 API，包括 Stream API、Table API 及 Flink SQL
-   生态完善，实时数仓的用途广泛，Flink 支持多种存储（HDFS、ES 等）
-   批流一体，Flink 已经在将流计算和批计算的 API 进行统一。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icfOINtNeUYDHg8tNkKg41ZKXDXyAKPClfmTz52ppvWlMTdaDrWdVNibw/640?wx_fmt=png)

-   **事件驱动型应用**

你是否有这样的需求：

```sql
我们公司有几万台服务器，希望能从服务器上报的消息中将 CPU、MEM、LOAD 信息分离出来做分析，然后触发自定义的规则进行报警？
我是公司的安全运维人员，希望能从每天的访问日志中识别爬虫程序，并且进行 IP 限制？
```

事件驱动型应用是一类具有状态的应用，它从一个或多个事件流提取数据，并根据到来的事件触发计算、状态更新或其他外部动作。

在传统架构中，我们需要读写远程事务型数据库，比如 MySQL。在事件驱动应用中数据和计算不会分离，应用只需访问本地（内存或磁盘）即可获取数据，所以具有更高的吞吐和更低的延迟。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icibYlSoQxMC1oPC89Z6JyS2BAzhJAdaCYPJFvySpTQibjcFjhEnUYibyeQ/640?wx_fmt=png)

Flink 的以下特性完美的支持了事件驱动型应用：

-   高效的状态管理，Flink 自带的 State Backend 可以很好的存储中间状态信息
-   丰富的窗口支持，Flink 支持包含滚动窗口、滑动窗口及其他窗口
-   多种时间语义，Flink 支持 Event Time、Processing Time 和 Ingestion Time
-   不同级别的容错，Flink 支持 At Least Once 或 Exactly Once 容错级别

我们也看到了，Flink 目前在各大公司的应用基本集中在以上 3 个方面。此外，未来的数据领域我们看到了一些例如：

-   Flink 和 IceBerg 等框架结合打造未来的数据湖

    ![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icoPkkwrDSmk3vvtI3tJGUQt8fctGdxvKRqQmKibnQJ3G6oujkoicCKLQA/640?wx_fmt=png)
-   基于 Flink 的 IOT 解决方案![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2PuhHbXjLtrTYXQEfy4OQ6icx6SevTJT7ONwtFbn51GXCIRMYm0JhTP3LsKSQl6OL6EYvX2NyJmsFg/640?wx_fmt=jpeg)

上述一些新兴的领域，Flink 也会有一些应用。此外，如果你把 Flink 当成了算法执行引擎，那么 FlinkML 可能就是你研究的重点。 
 [https://mp.weixin.qq.com/s/xh4SEX9t-fRVdoiAl0KKSQ](https://mp.weixin.qq.com/s/xh4SEX9t-fRVdoiAl0KKSQ)
