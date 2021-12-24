# 那些被问懵的Flink面试题
前言  

* * *

         有没有去面试的时候被问到 Flink 的面试题你答不上来，为什么呢？，菜吗？不是。原因是你接触的面试题太少了，那我今天就根据不同的群体来给大家你分享。![](https://mmbiz.qpic.cn/mmbiz_png/2UHIhrbfNBeST23dhL0Ukss6oyibYLufFRic2B6kBHBH0QMPRBicEtxCMax2MZP9TSh9pKMtib2ibC6ia6rQLicDibkhicQ/640?wx_fmt=png)
1 Flink 基础（适合初入职场）

1.  简单介绍一下 Flink
2.  Flink 相比传统的 Spark Streaming 区别?
3.  Flink 的组件栈有哪些？
4.  Flink 的运行必须依赖 Hadoop 组件吗？
5.  你们的 Flink 集群规模多大？
6.  Flink 的基础编程模型了解吗？
7.  Flink 集群有哪些角色？各自有什么作用？
8.  说说 Flink 资源管理中 Task Slot 的概念
9.  说说 Flink 的常用算子？
10. 说说你知道的 Flink 分区策略？
11. Flink 的并行度了解吗？Flink 的并行度设置是怎样的？
12. Flink 的 Slot 和 parallelism 有什么区别？
13. Flink 有没有重启策略？说说有哪几种？
14. 用过 Flink 中的分布式缓存吗？如何使用？
15. 说说 Flink 中的广播变量，使用时需要注意什么？
16. 说说 Flink 中的窗口？
17. 说说 Flink 中的状态存储？
18. Flink 中的时间有哪几类
19. Flink 中水印是什么概念，起到什么作用？
20. Flink Table & SQL 熟悉吗？TableEnvironment 这个类有什么作用
21. Flink SQL 的实现原理是什么？是如何实现 SQL 解析的呢？

## 2 Flink 中级 (适合 1~2 年开发经验的人）

1.  Flink 是如何支持批流一体的？
2.  Flink 是如何做到高效的数据交换的？
3.  Flink 是如何做容错的？
4.  Flink 分布式快照的原理是什么？
5.  Flink 是如何保证 Exactly-once 语义的？
6.  Flink 的 kafka 连接器有什么特别的地方？
7.  说说 Flink 的内存管理是如何做的?
8.  说说 Flink 的序列化如何做的?
9.  Flink 中的 Window 出现了数据倾斜，你有什么解决办法？
10. Flink 中在使用聚合函数 GroupBy、Distinct、KeyBy 等函数时出现数据热点该如何解决？
11. Flink 任务延迟高，想解决这个问题，你会如何入手？
12. Flink 是如何处理反压的？
13. Flink 的反压和 Strom 有哪些不同？
14. Operator Chains（算子链）这个概念你了解吗？
15. Flink 什么情况下才会把 Operator chain 在一起形成算子链？
16. 说说 Flink1.9 的新特性？
17. 消费 kafka 数据的时候，如何处理脏数据？

## 3 Flink 高级 (适合 3 年以上）

1.  Flink Job 的提交流程
2.  Flink 所谓 "三层图" 结构是哪几个 "图"？
3.  JobManger 在集群中扮演了什么角色？
4.  JobManger 在集群启动过程中起到什么作用？
5.  TaskManager 在集群中扮演了什么角色？
6.  TaskManager 在集群启动过程中起到什么作用？
7.  Flink 计算资源的调度是如何实现的？
8.  简述 Flink 的数据抽象及数据交换过程？
9.  Flink 中的分布式快照机制是如何实现的？
10. 简单说说 FlinkSQL 的是如何实现的？

## 4 企业面试题（重点）

1.  应用架构
2.  压测和监控
3.  有了 Spark 还为什么用 Flink
4.  checkpoint 的存储
5.  exactly-once 的保证
6.  状态机制
7.  海量 key 去重
8.  checkpoint 与 spark 比较
9.  watermark 机制
10. exactly-once 如何实现
11. CEP
12. 三种时间语义
13. 数据高峰的处理

## 小结

          好今天的 Flink 的题目就分享到这里，背过上面的那些题目害怕面试官提问？信自己，努力和汗水总会能得到回报的。我是大数据老哥，我们下期见\~\~~

> 答案获取：[https://github.com/lhh2002/Framework-Of-BigData](https://github.com/lhh2002/Framework-Of-BigData)

![](https://mmbiz.qpic.cn/mmbiz_png/h7whAEqyjWRKzvOm3Zjiazq0nOkYbXWTb16VLTQzwWagbMSOuIAyKD78ErPeanWm9XtPnojM3NOicYAs4vkX1RDA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/zgDxf7oOMEv9XQW77ApR6aShzMIDhXkf42pS4PppsM5mx5wmtFtPZV85xKWYfjJ7mfmJu3hQrJhDZCsRcsl60A/640?wx_fmt=png)

点个 在看 你最好看 
 [https://mp.weixin.qq.com/s/E2pv1sA_QdE4A-HNCrlMdw](https://mp.weixin.qq.com/s/E2pv1sA_QdE4A-HNCrlMdw)
