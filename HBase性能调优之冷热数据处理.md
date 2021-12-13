# HBase性能调优之冷热数据处理
![](https://mmbiz.qpic.cn/mmbiz_jpg/xqJ76ZLyAwBJqics7JOHibribibIVOr90dZmK4bGuA3O8r56TnJPereApbA73JOCNFOfouePBQrzr0YQShc1jTQFeQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_png/7QRTvkK2qC6jNJS1xw99aQZkRfiaXtVtDzxao4mIvdhHKL53TVBmhAx5OylXVXCBQm1RGRhjjYQYsWdelI1gr1Q/640?wx_fmt=png)

**长按二维码关注**

大数据领域必关注的公众号

![](https://mmbiz.qpic.cn/mmbiz_jpg/xqJ76ZLyAwBJqics7JOHibribibIVOr90dZm5FdKd7658Roicd4JibRwgU5xp64v17kBlcRSBubspF2IlxqeyE0rETAw/640?wx_fmt=jpeg)

**By 大数据研习社**

**概要：** 在 HBase 集群资源紧张的情况下，可以通过下面的方式进行优化，从而保证最近写入的数据先查询，提供查询效率。

**关键词：** HBase、性能调优、冷热数据处理

**HBase 性能调优之冷热数据处理**

在 HBase 集群资源紧张的情况下，可以通过下面的方式进行优化，从而保证最近写入的数据先查询，提供查询效率。目前由于硬件层面不可控因素太多，所以接下来我们从软件和设计角度考虑调优手段。

**1.1**

**优化方式 1**

**优化手段：**   

将 (Long.MAX_VALUE – timestamp) 加到 rowkey

**优化原理：** 

如果最近写入 HBase 表中的数据是最可能被访问的，可以考虑将时间戳作为 row key 的一部分，由于是字典序排序，所以可以使用 Long.MAX_VALUE – timestamp 作为 row key，这样能保证新写入的数据排在最前面，在读取时可以被快速命中。

**说明：** 

快速命中，较少 IO，查询也就快

**1.2**

**优化方式 2**

**优化手段：**   

适当减少 memstore 的 flush

**优化原理：** 

写数据先写入 memstore，超多大小才 flush 到磁盘，可以把阈值适当调大，使刚写入的热数据在内存里待一会，增大缓存命中率，同时也降低了写磁盘的次数

**说明：** 

阈值不能太大，否则 flush 比较慢，可以通过 hbase.regionserver.global.memstore.upperLimit 参考控制阈值大小。

**1.3**

**优化方式 3**

**优化手段：** 

使用堆外内存做二级缓存 (blockcache)

**优化原理：** 

热数据不容易被淘汰，命中率高，有一定的效果

**说明：** 

1）使用的 LRU 算法

2）不能以单次读写论英雄

3）云主机没有配 ssd(固态硬盘)，也无法使用二级缓存

**1.4**

**优化方式 4**

**优化手段：** 

按天建表 + 最近时间段常驻内存

(设置将冷数据表关闭 blockcache，留更多缓存空间给热数据)

通过 hfile.block.cache.size=0 来关闭

**优化原理：** 

按时间段建表，比如按天，当天的可以常驻内存，其余的动态设置为不常住内存；

**说明：** 

具体时间段怎么界定有待讨论，不到万不得已不建议时间段搞得很小

**1.5**

**优化方式 5**

**优化手段：** 

按天建表 + 当天适当增大副本

**优化原理：** 

副本数越多，数据本地性就越强，查询会变快

**说明：** 

副本数不能过多，占用大量存储；冷数据可以降副本以缓解整体存储的占用

欢迎点赞 + 收藏 + 在看  素质三连 

**完**

▼

往期精彩回顾

▼

[程序员，如何避免内卷](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247489160&idx=1&sn=ae48de203f212768735e387f45bf5591&chksm=ec1c7048db6bf95efd02cff03950ea092b883b691d697440b745ac59a24f599b334a5e9103e4&scene=21#wechat_redirect)  

[【全网首发】Hadoop 3.0 分布式集群安装](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247483999&idx=1&sn=016e4c4d0ba7bd96e9f2d2d5f8cbe0de&chksm=ec1c649fdb6bed89e74984c28859557f577cdfedcdcee3f67ad50a5097daaff0e67718c50121&scene=21#wechat_redirect)

[【2020 最新整理】大数据面试 130 题](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247485085&idx=1&sn=5ea5585ebe9a101e3287b934f11a9532&chksm=ec1c605ddb6be94b3d13c7f329e45037b2fbdcbe7e8cedb7ad79f8f275054d0f2435668d64e0&scene=21#wechat_redirect)

[某集团大数据平台整体架构及实施方案完整目录](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247488101&idx=1&sn=0327c492f0f30cbfe3043bd274ccaae5&chksm=ec1c74a5db6bfdb3d609ec19a70fec0415931aa860de02ec854f3a01f8dfe169c7d6080ed793&scene=21#wechat_redirect)  

[大数据平台基础架构指南](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247488735&idx=1&sn=6bfe430958d42a3952a6a95eaa769260&chksm=ec1c721fdb6bfb09ad2647fc7abf8ab8d2dbb98cf4834b764a06fc580255bed639f4293500ba&scene=21#wechat_redirect)  

[大数据凉凉了？Apache 将一众大数据开源项目束之高阁！](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247489171&idx=1&sn=4ae22d71dd2b390f80041378a36e1d68&chksm=ec1c7053db6bf9457c4bc087fb872e55c4d2db060c963f9fccebc5a0ba4e9f28964dc4b36a59&scene=21#wechat_redirect)  

[实战企业数据湖，抢先数仓新玩法](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247488568&idx=1&sn=7d37a9c703e2ff4cec5d93a342dd54d3&chksm=ec1c72f8db6bfbeee1da9a4174e22afa3846e0f66cc18dde13dd7ab13cac33d15e28266c7e30&scene=21#wechat_redirect)  

[Superset 制作智慧数据大屏，看它就够了](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247489329&idx=1&sn=d2b475f1236971db561cf9b526ccb7a0&chksm=ec1c71f1db6bf8e716211d53c935f466c73b4241f418dae5b79ee54489c570cc48a01473c5e6&scene=21#wechat_redirect)  

[Apache Flink 在快手的过去、现在和未来](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247487980&idx=1&sn=f91bae73e1b5372feffa25d98f07ba51&chksm=ec1c772cdb6bfe3add9728dda65556aa609e27d95ae7d2d9e29def9c364b525516282d614528&scene=21#wechat_redirect)  

[大数据基础运维：HDFS 参数调优](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247487732&idx=1&sn=4974dece1a40ac5782573e2c1f47cfd1&chksm=ec1c7634db6bff22ae52576cd5f540960125ebd8ca84fa06a69919256c2797d17f3e27e18418&scene=21#wechat_redirect)

[大数据无处不在，向左还是向右](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247488004&idx=1&sn=b54fdbbceeae8526d2e1c99bbbcfaf7b&chksm=ec1c74c4db6bfdd24b488232cd0df43dbdc7f76fa1fabad18b0be6b662c637d49c61da9e435e&scene=21#wechat_redirect)  

[【HBase 调优】Hbase 万亿级存储性能优化总结](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247483936&idx=1&sn=51948ae9478f8fbd0e16135b477fc030&chksm=ec1c64e0db6bedf6f70f4e90358513e376f9b56bb39c9b86bd929b2b931ce8ff80c10f647285&scene=21#wechat_redirect)  

[【Python 精华】100 个 Python 练手小程序](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247483800&idx=1&sn=c6d159dc2d90aa1da1631bf2e95c4e7e&chksm=ec1c6758db6bee4e3f10f442be9b16b63fd9a15ccdf849dd045a5ee4fadef6368689fb428b4b&scene=21#wechat_redirect)  

[【HBase 企业应用开发】工作中自己总结的 Hbase 笔记，非常全面！](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247483794&idx=1&sn=02c9d959bce88520d833f8b10d63c997&chksm=ec1c6752db6bee44c37842eca5cc6f7973e99675eedeedbfcec5a84bd508f776cecc72d4fbe2&scene=21#wechat_redirect)

[【剑指 Offer】近 50 个常见算法面试题的 Java 实现代码](http://mp.weixin.qq.com/s?__biz=MzI5MDYxNjIzOQ==&mid=2247483767&idx=1&sn=ac3bb74985c13b3a5e92089d8b0510ba&chksm=ec1c67b7db6beea168b5d10f8c8d2cbb8071254149c24869f58fe615e378109bf5ef3f27fe35&scene=21#wechat_redirect)  

![](https://mmbiz.qpic.cn/mmbiz_jpg/xqJ76ZLyAwCZa1fpodUQy3Ncs5WsagqvxHHUfHsMShpm77L6HVCoIVhibOEOdFefFLejMjQXyxOoDrWLzycr8pg/640?wx_fmt=jpeg)

     长按识别左侧二维码

    关注领福利    

**领 10 本经典大数据书** 
 [https://mp.weixin.qq.com/s/QQNfMTEdyKph3cXkJ_CU0A](https://mp.weixin.qq.com/s/QQNfMTEdyKph3cXkJ_CU0A)
