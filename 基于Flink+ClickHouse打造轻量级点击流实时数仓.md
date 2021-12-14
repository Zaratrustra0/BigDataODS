# 基于Flink+ClickHouse打造轻量级点击流实时数仓
点击上方卡片进入**五分钟学大数据**主页  

然后点击右上角 “设为星标”  

比别人更快接收好文章

## 一、前言

Flink 和 ClickHouse 分别是实时计算和 OLAP 领域的翘楚，也是近些年非常火爆的开源框架，很多大厂都在将两者结合使用来构建各种用途的实时平台，效果很好。关于两者的优点就不再赘述，本文来简单介绍笔者团队在点击流实时数仓方面的一点实践经验。

### 1. 点击流及其维度建模

所谓点击流（click stream），就是指用户访问网站、App 等 Web 前端时在后端留下的轨迹数据，也是流量分析（traffic analysis）和用户行为分析（user behavior analysis）的基础。点击流数据一般以访问日志和埋点日志的形式存储，其特点是量大、维度丰富。以我们一个中等体量的普通电商平台为例，每天产生 200+GB、十亿条左右的原始日志，埋点事件 100 + 个，涉及 50 + 个维度。按照 Kimball 的维度建模理论，点击流数仓遵循典型的星形模型，简图如下。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PYfNskNjoprP2w71lbAiaKvepFOA4mBiaKjGGuPNIg7Dcbic8A9fQ26qa2NNORsZSkb60mUp0Unp2WQ/640?wx_fmt=png)

### 2. 点击流数仓分层设计

点击流实时数仓的分层设计仍然可以借鉴传统数仓的方案，以扁平为上策，尽量减少数据传输中途的延迟。简图如下。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PYfNskNjoprP2w71lbAiaKvHJlLibQFCTRrmm1oWwPfCYntrGv7xoTqGwj4jibaoSZ32mupuR1pqooA/640?wx_fmt=png)

-   **DIM 层**：维度层，MySQL 镜像库，存储所有维度数据。
-   **ODS 层**：贴源层，原始数据由 Flume 直接进入 Kafka 的对应 topic。
-   **DWD 层**：明细层，通过 Flink 将 Kafka 中数据进行必要的 ETL 与实时维度 join 操作，形成规范的明细数据，并写回 Kafka 以便下游与其他业务使用。再通过 Flink 将明细数据分别写入 ClickHouse 和 Hive 打成大宽表，前者作为查询与分析的核心，后者作为备份和数据质量保证（对数、补数等）。
-   **DWS 层**：服务层，部分指标通过 Flink 实时汇总至 Redis，供大屏类业务使用。更多的指标则通过 ClickHouse 物化视图等机制周期性汇总，形成报表与页面热力图。特别地，部分明细数据也在此层开放，方便高级 BI 人员进行漏斗、留存、用户路径等灵活的 ad-hoc 查询，这些也是 ClickHouse 远超过其他 OLAP 引擎的强大之处。

## 二、要点与注意事项

### 1. Flink 实时维度关联

Flink 框架的异步 I/O 机制为用户在流式作业中访问外部存储提供了很大的便利。针对我们的情况，有以下三点需要注意：

-   使用异步 MySQL 客户端，如 Vert.x MySQL Client。
-   AsyncFunction 内添加内存缓存（如 Guava Cache、Caffeine 等），并设定合理的缓存驱逐机制，避免频繁请求 MySQL 库。
-   实时维度关联仅适用于缓慢变化维度，如地理位置信息、商品及分类信息等。快速变化维度（如用户信息）则不太适合打进宽表，我们采用 MySQL 表引擎将快变维度表直接映射到 ClickHouse 中，而 ClickHouse 支持异构查询，也能够支撑规模较小的维表 join 场景。未来则考虑使用 MaterializedMySQL 引擎将部分维度表通过 binlog 镜像到 ClickHouse。

### 2. Flink-ClickHouse Sink 设计

可以通过 JDBC（flink-connector-jdbc）方式来直接写入 ClickHouse，但灵活性欠佳。好在 clickhouse-jdbc 项目提供了适配 ClickHouse 集群的 BalancedClickhouseDataSource 组件，我们基于它设计了 Flink-ClickHouse Sink，要点有三：

-   写入本地表，而非分布式表，老生常谈了。
-   按数据批次大小以及批次间隔两个条件控制写入频率，在 part merge 压力和数据实时性两方面取得平衡。目前我们采用 10000 条的批次大小与 15 秒的间隔，只要满足其一则触发写入。
-   BalancedClickhouseDataSource 通过随机路由保证了各 ClickHouse 实例的负载均衡，但是只是通过周期性 ping 来探活，并屏蔽掉当前不能访问的实例，而没有故障转移——亦即一旦试图写入已经失败的节点，就会丢失数据。为此我们设计了重试机制，重试次数和间隔均可配置，如果当重试机会耗尽后仍然无法成功写入，就将该批次数据转存至配置好的路径下，并报警要求及时检查与回填。

当前我们仅实现了 DataStream API 风格的 Flink-ClickHouse Sink，随着 Flink 作业 SQL 化的大潮，在未来还计划实现 SQL 风格的 ClickHouse Sink，打磨健壮后会适时回馈给社区。另外，除了随机路由，我们也计划加入轮询和 sharding key hash 等更灵活的路由方式。

还有一点就是，ClickHouse 并不支持事务，所以也不必费心考虑 2PC Sink 等保证 exactly once 语义的操作。如果 Flink 到 ClickHouse 的链路出现问题导致作业重启，作业会直接从最新的位点（即 Kafka 的 latest offset）开始消费，丢失的数据再经由 Hive 进行回填即可。

### 3. ClickHouse 数据重平衡

ClickHouse 集群扩容之后，数据的重平衡（reshard）是一件麻烦事，因为不存在类似 HDFS Balancer 这种开箱即用的工具。一种比较简单粗暴的思路是修改 ClickHouse 配置文件中的 shard weight，使新加入的 shard 多写入数据，直到所有节点近似平衡之后再调整回来。但是这会造成明显的热点问题，并且仅对直接写入分布式表才有效，并不可取。

因此，我们采用了一种比较曲折的方法：将原表重命名，在所有节点上建立与原表 schema 相同的新表，将实时数据写入新表，同时用 clickhouse-copier 工具将历史数据整体迁移到新表上来，再删除原表。当然在迁移期间，被重平衡的表是无法提供服务的，仍然不那么优雅。

本文作者：LittleMagic

往期推荐

\[

大数据面试吹牛草稿 V2.0

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg2MzU2MDYzOA==&mid=2247490108&idx=1&sn=13245563d579f5cd2dcb5d0d96ce4d2e&chksm=ce77ecedf90065fbfca43fcdea481b17e7aef3dba6507a27d3459652a7d0f76f3f888d4d050c&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247490108&idx=1&sn=13245563d579f5cd2dcb5d0d96ce4d2e&chksm=ce77ecedf90065fbfca43fcdea481b17e7aef3dba6507a27d3459652a7d0f76f3f888d4d050c&scene=21#wechat_redirect))

\[

最强最全面的数仓建设规范指南（纯干货建议收藏）

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg2MzU2MDYzOA==&mid=2247489588&idx=1&sn=08c3b9443aba9e4757ec0d078bce3476&chksm=ce77eee5f90067f36004a10355d088b4adab133fda45ef33c9826406a1e8deaafe13993ce9ba&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247489588&idx=1&sn=08c3b9443aba9e4757ec0d078bce3476&chksm=ce77eee5f90067f36004a10355d088b4adab133fda45ef33c9826406a1e8deaafe13993ce9ba&scene=21#wechat_redirect))

\[

最强最全面的 Hive SQL 开发指南，超四万字全面解析！

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg2MzU2MDYzOA==&mid=2247491319&idx=1&sn=e52044e613102608fe5a3cd762102bd6&chksm=ce77e826f90061308ef43377b9604f01a74a550e7dc596d86b6ee543dc58574c0009bc3ae1a6&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247491319&idx=1&sn=e52044e613102608fe5a3cd762102bd6&chksm=ce77e826f90061308ef43377b9604f01a74a550e7dc596d86b6ee543dc58574c0009bc3ae1a6&scene=21#wechat_redirect))

\[

五万字 | 耗时一个月，整理出这份 Hadoop 吐血宝典

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg2MzU2MDYzOA==&mid=2247488984&idx=1&sn=5607a6bc24db7ae9f040a37b54e417bd&chksm=ce77e309f9006a1f4f78f5ce5682ab3ac18c2688e2d7de10d77256548f41dc08ea927ccd971c&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247488984&idx=1&sn=5607a6bc24db7ae9f040a37b54e417bd&chksm=ce77e309f9006a1f4f78f5ce5682ab3ac18c2688e2d7de10d77256548f41dc08ea927ccd971c&scene=21#wechat_redirect))

\[

美团数据平台及数仓建设实践，超十万字总结

]([http://mp.weixin.qq.com/s?\_\_biz=Mzg2MzU2MDYzOA==&mid=2247488024&idx=1&sn=d069bf73967f49f40e1c7dfb05180dd3&chksm=ce77e4c9f9006ddf232e57bd7b113ccf90211853fce9f227655b7d7d1e35c56abdef1e5b301b&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247488024&idx=1&sn=d069bf73967f49f40e1c7dfb05180dd3&chksm=ce77e4c9f9006ddf232e57bd7b113ccf90211853fce9f227655b7d7d1e35c56abdef1e5b301b&scene=21#wechat_redirect))

[数据仓库之数据质量建设（深度好文）](http://mp.weixin.qq.com/s?__biz=Mzg2MzU2MDYzOA==&mid=2247487775&idx=1&sn=438350dffe6b1cac346f292cc42bc3e1&chksm=ce77e7cef9006ed8c124adf74fe9270f2894bdb56962668bea945adf2167f49a214e2f5267cf&scene=21#wechat_redirect)

**--END--**

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ZubDbBye0zG2nW9msUBGp3lmbrXyyQPD4keYgsgVhnvmT3zBc5J9qQIxJNMxrzg6Laso7PoPGuQaMStSglsnibA/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/QVDCSuNQr272kOgDxA6yRg](https://mp.weixin.qq.com/s/QVDCSuNQr272kOgDxA6yRg)
