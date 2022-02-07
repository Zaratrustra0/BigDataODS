# 如何优雅的设计DWS层？
文末数仓干货文章合集推荐  

![](https://mmbiz.qpic.cn/mmbiz_gif/ZlqFTnJCYqyDBOqD1KIOwsEY8meFVQgj7ZbKw49XaibYeJ2yrAibbhicbiaSxGm9P8JFps5hkux716MOuEFpeyXbpg/640?wx_fmt=gif)

**导读**

对于数仓的分层，想必大家都不陌生。基于 OneData 方法论的三层数仓划分：**数据引入层**（ODS，Operational Data Store）、**数据公共层**（CDM，Common Dimenions Model）和**数据应用层**（ADS，Application Data Store）早就深入人心。  

当然啦，涉及到每一层具体该怎么开发、建模，可能大家都有自己的理解。

但好在大家对数据建模重要性的认识那都是一致的，如果我们把指标比作树上的果实，那么模型就好比是大树的躯干，想让果实结得好，必须让树干变得粗壮。

我们先来回想下，构建数据中台的初衷是什么：

-   缺少可以复用的数据
-   大家不得不使用原始数据进行清洗、加工和计算指标
-   大量重复代码的开发对资源的消耗  

问题的根源就在于数据模型的无法复用，以及数据开发都是烟囱式的。所以要解决这个问题，就要搞清楚健壮的数据模型该如何设计。

下图是数仓分层的逻辑架构图，大家不妨回忆一下数据模型的分层设计：

[![](https://mmbiz.qpic.cn/mmbiz_png/zWSuIP8rdu2gwBjUucvb5oWPQiaibicJbAqUDJP6tcf1pRRHuckTqRibMyBvtfxjWj78vjJ9ElJytiatHxLGyVUJE8w/640?wx_fmt=png)
](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247507972&idx=1&sn=e19cfa6cb4119831f3ab5ef26af03c98&chksm=cf379209f8401b1f449f43a0631aa46981fecc84be511da2ac91c0e552fdcd45ea6b9aac218e&scene=21#wechat_redirect)

-   **数据引入层（ODS，Operational Data Store，又称数据基础层）**：将原始数据几乎无处理地存放在数据仓库系统中，结构上与源系统基本保持一致，是数据仓库的数据准备区。这一层的主要职责是将基础数据同步、存储。
-   **数据公共层（CDM，Common Dimenions Model）**：存放明细事实数据、维表数据及公共指标汇总数据。其中，明细事实数据、维表数据一般根据 ODS 层数据加工生成。公共指标汇总数据一般根据维表数据和明细事实数据加工生成。CDM 层又细分为维度层（DIM）、明细数据层（DWD）和汇总数据层（DWS），采用维度模型方法作为理论基础， 可以定义维度模型主键与事实模型中外键关系，减少数据冗余，也提高明细数据表的易用性。在汇总数据层同样可以关联复用统计粒度中的维度，采取更多的宽表化手段构建公共指标数据层，提升公共指标的复用性，减少重复加工。


-   **维度层（DIM，Dimension）**：以维度作为建模驱动，基于每个维度的业务含义，通过添加维度属性、关联维度等定义计算逻辑，完成属性定义的过程并建立一致的数据分析维表。为了避免在维度模型中冗余关联维度的属性，基于雪花模型构建维度表。
-   **明细数据层（DWD，Data Warehouse Detail）**：以业务过程作为建模驱动，基于每个具体的业务过程特点，构建最细粒度的明细事实表。可将某些重要属性字段做适当冗余，也即宽表化处理。
-   **汇总数据层（DWS，Data Warehouse Summary）**：以分析的主题对象作为建模驱动，基于上层的应用和产品的指标需求，构建公共粒度的汇总指标表。以宽表化手段物理化模型，构建命名规范、口径一致的统计指标，为上层提供公共指标，建立汇总宽表、明细事实表。


-   **数据应用层（ADS，Application Data Store）**：存放数据产品个性化的统计指标数据，根据 CDM 层与 ODS 层加工生成。

通常，大家都会有这样的疑问：明明可以直接从 DWD 层取数，为什么要多此一举建立 DWS 的汇总逻辑表呢？

我想说的是：如果在业务场景不复杂的情况下，那样做是没有问题的。可一旦面对复杂的业务场景，那这种做法无疑是混乱的根源所在，前面提到的烟囱式开发、计算资源的浪费等等情况，正是这样产生的。

举个例子，我们需要的是从数据明细层中做一个初步的汇总，抽象出来一些通用的维度：时间、用户 ID、IP 等，并根据这些维度做一些统计，比如用户每个时间段在不同登录 IP 购买的商品数等。

这里做一层轻度的汇总会让计算更加的高效，在此基础上如果计算仅 7 天、30 天、90 天的行为的话会快很多。我们希望 80% 的业务都能通过我们的 DWS 层计算，而不是 ODS 或者 DWD 层。

聚集是指针对原始明细粒度的数据进行汇总。DWS 汇总数据层是面向分析对象的主题聚集建模，以零售的场景为例，我们最终的分析目标为：最近一天某个类目（例如，厨具）商品在各省的销售总额、该类目销售额 Top10 的商品名称、各省用户购买力分布。

因此，我们可以以最终交易成功的商品、类目、买家等角度对最近一天的数据进行汇总。数据聚集的注意事项如下：

-   **聚集是不跨越事实的**。聚集是针对原始星形模型进行的汇总。为获取和查询与原始模型一致的结果，聚集的维度和度量必须与原始模型保持一致，因此聚集是不跨越事实的，所以原子指标只能基于一张事实表定义，但是支持原子指标组合为衍生原子指标。
-   **聚集会带来查询性能的提升，但聚集也会增加 ETL 维护的难度**。当子类目对应的一级类目发生变更时，先前存在的、已经被汇总到聚集表中的数据需要被重新调整。

此外，进行 DWS 层设计时还需遵循**数据公用性原则**。数据公用性需要考虑汇总的聚集是否可以提供给第三方使用。我们可以思考基于某个维度的聚集是否经常用于数据分析中，如果答案是肯定的，就有必要把明细数据经过汇总沉淀到聚集表中。

简单的说就是：

-   主题
-   宽表
-   轻度汇总

以电商零售的场景为例，我们已经基于 ODS 层的订单表、用户表、商品表、优惠券表等，经过 ETL 完成了 DWD 层的建模，一般是采用星型模型。

这里严格按照：**业务过程→声明粒度→确认维度→确认事实** 完成建模，过程如下：

![](https://mmbiz.qpic.cn/mmbiz_png/zWSuIP8rdu2gwBjUucvb5oWPQiaibicJbAqzibY6DbzX4NH06m9ib5tICLcF5HLkbiaicl8dL7xnBff7kQibyrzgRP7flQ/640?wx_fmt=png)

接下来，便是到了 DWS 层设计的环节。按照我们上面的设计思路，通过从维度表去看事实表，便可得出每天的宽表。

这样即可统计各个主题对象的当天行为，服务于 ADS 层的主题宽表以及一些业务的明细数据，也可以以应对一些特殊的需求，例如：购买行为，统计商品复购率等。

[![](https://mmbiz.qpic.cn/mmbiz_png/zWSuIP8rdu2gwBjUucvb5oWPQiaibicJbAqFJiaEOlKHCbbuuxOT6864Lf2aIcGBUdgZ52cNn4TvGMfVylzecf1Waw/640?wx_fmt=png)
](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247507972&idx=2&sn=3f25b2a1527f453bcde3edc35721fb38&chksm=cf379209f8401b1fbd694838d7d029a3048261d545bb3f9394b0bce083913084a7ae714e6b67&scene=21#wechat_redirect)

通过外键获取相关的度量值，我们整合多个 DWD 的明细事实表度量值来构成新表。

在这里，我们还是要遵循上文提到的设计原则，在设计上尽量体现出公共性、使用简单并且用户很容易理解。

在我们数据中台实际实施落地的过程中，团队不但要建设公共数据层，形成数据中台，还要承担着新需求的压力。

往往我们要先满足需求（活下去），再研发公共数据层（构建美好未来），在满足业务需求的过程中，再根据需求不断对模型进行迭代和优化，随着时间的推移，越来越多的业务需求可以通过 DWS 层的数据完成。

这一过程中，完善度是很好的考核标准，主要看 DWS 层汇总的数据能满足多少的查询需求，如果汇总数据无法满足需求，使用数据的人就必须使用明细的数据，甚至是 ODS 层的原始数据。

DWS/ADS 层的完善度越高，说明数据的上层建设越完善，而从使用者的角度来说，查询快、易取数、用的爽，那才是硬道理。

end ~  

**![](https://mmbiz.qpic.cn/mmbiz_gif/XVbUB37bCmAE7DFc2slTaY80fZSWeicgEfGFjfdZOReaAHD4m0PwmuiazjEzytmy3KTKs6jDiavlmv9xQ4FfvRqoA/640?wx_fmt=gif)**

**实时数仓：**   

1.  [启蒙 | 如果你也想做实时数仓…](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484067&idx=1&sn=d424b90296f777d5bc6d7f8fd91a2e06&chksm=cf3430aef843b9b85501d7d71db9c0c10b527a07c41c1323c9be5e5cac6a8428cc5ba0dcfc33&scene=21#wechat_redirect)
2.  [启蒙 | Flink 0-1 知识点之全景图. xmind](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247498537&idx=1&sn=017768e3f05efb45c9b2a5222b2a6aa0&chksm=cf37c924f8404032b90159212755cb01b848bd221b41464c96cb120a732b7f9a3d521c24e6d3&scene=21#wechat_redirect)
3.  [启蒙](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247498537&idx=1&sn=017768e3f05efb45c9b2a5222b2a6aa0&chksm=cf37c924f8404032b90159212755cb01b848bd221b41464c96cb120a732b7f9a3d521c24e6d3&scene=21#wechat_redirect)[ | ClickHouse 全面学习指南. xmind](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247500026&idx=1&sn=421aea78d0e8015f0e7cac6f5181789d&chksm=cf37f2f7f8407be19769f2cd4f6c6af818a85d74d567b29ea571b6043da366a56a49491585d7&scene=21#wechat_redirect)  
4.  [系列 | 实时数仓实践第一篇 NO.1](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484331&idx=2&sn=7995b83c060494854678e521d8e45b2d&chksm=cf3431a6f843b8b0093f1d28eeb595185bd0c5db149302241049a5d64dd96f759887c8c2fb08&scene=21#wechat_redirect)
5.  [系列 | 实时数仓实践第二篇 NO.2](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484355&idx=1&sn=4e59be34662a74cec17101967bd4359b&chksm=cf3431cef843b8d82f998fa67b71d242d3263e2e0c995abef586869e9ce4b0eec9e9c80e1f70&scene=21#wechat_redirect)
6.  [系列 | 离线数仓实践第三篇 NO.3](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484463&idx=2&sn=4d963b251e1d7cdb7ee00ff856115cd4&chksm=cf343622f843bf345dc04d97eeee18d36804d6fbbe03e0a68fffcac61b614c50a1b121fb7f30&scene=21#wechat_redirect)
7.  [系列 | 实时数仓实践第四篇 NO.4](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247494880&idx=2&sn=6d8f3e8591150713aeacd4db05df84a2&chksm=cf37deedf84057fb66024ddd972f3e06851510b910d7d900fc38d538efb5e26157184bef18bf&scene=21#wechat_redirect)  
8.  [回顾 | 基于 Flink 的严选实时数仓实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484750&idx=1&sn=8409b44181060f7a033de5bf3c89ef6b&chksm=cf343743f843be55f039e877d40b30d321b2e6597e433e5045167a4a77a239649e7242fbef04&scene=21#wechat_redirect)
9.  [回顾 | 基于 Flink 的 58 实时数仓实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484589&idx=1&sn=a52011a059955bfc0780ae51f0a3c0f1&chksm=cf3436a0f843bfb602a115fc80b5d2d5e05bcc65821a87e0ce1ff5b90829270ea510d9f1434a&scene=21#wechat_redirect)
10. [回顾 | 基于 Flink 的美团实时数仓实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247484426&idx=1&sn=7a251830204daa9845a0cc6fc533a29d&chksm=cf343607f843bf1176c889bcf757412198f7b8d52c0d73f14998f91e7dadf9bc5940a8fc3790&scene=21#wechat_redirect)
11. [回顾 | 爱奇艺大数据生态实时数仓](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247503349&idx=1&sn=853afe4559943a33a2717396a733f089&chksm=cf37fff8f84076ee950d34f69e5c319c516e63fea71fefcfe0312fe20054e6b04f7b0a1ee634&scene=21#wechat_redirect)
12. [回顾 | 菜鸟实时数仓 2.0 进阶之路](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247495208&idx=1&sn=629633a4e6cc84eb066001340df591e6&chksm=cf37dc25f840553308421ed1010fe92fbb91190fe6104d7576b9102e89e27701e741427d9d33&scene=21#wechat_redirect)
13. [回顾 | 美团外卖实时数仓建设与实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247493817&idx=2&sn=19a48642118b4213feb6509f91d8c538&chksm=cf37dab4f84053a28cf88dea4e0d035c3eb1cabe891322d4a17bdb90d3dbd0879a1b874655b6&scene=21#wechat_redirect)
14. [架构 | 漫谈实时数仓架构](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247504011&idx=2&sn=407426d004bd3d5b95756800a3ca3b72&chksm=cf37e286f8406b90cf06e303eaf5d74785e2877bfb45950e963f8cf30fa21b92431073188372&scene=21#wechat_redirect)
15. [架构 | 实时数据仓库 1.0 2.0 3.0 架构](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247503651&idx=1&sn=e8d73438e642aafa220e0c73c44ea330&chksm=cf37fd2ef8407438a82565fb25096fd5d52d39a1c3569640c76045a200fdbbecba39f81ad980&scene=21#wechat_redirect)  
16. [架构 | 实时数仓架构设计与选型（附 ppt）](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247495260&idx=2&sn=5ddd1ab06312cd17001b7c1c1fea26eb&chksm=cf37dc51f8405547ae92fdb59410a40361c528c1e61dab2e4b31924e10ab43a58b828ba8c917&scene=21#wechat_redirect)  
17. [架构 | 爱奇艺大数据实时数仓：ClickHouse](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247504212&idx=1&sn=837646027dcea07cb57e8889a5ee6b67&chksm=cf37e359f8406a4f218d75f66e8000cca541c49471e6e46d71c313350f3ec9e6e50c6767bae0&scene=21#wechat_redirect)
18. [源码 | Flink SQL 实时数仓开源 UI 平台](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247498423&idx=2&sn=450d08d581fd15deec7bdf0010dad779&chksm=cf37c8baf84041acdd2cdf921274327c1122a09c5c5a86461fd59cb59ec3f79770b188e36ed4&scene=21#wechat_redirect)
19. [源码 | Flink 实时维表 join 方法总结（附项目源码）](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247497869&idx=1&sn=81324d4af641ce981d1a35a3072559bc&chksm=cf37ca80f8404396da1f0c2d29918d10503dab77c34984a84ce774e26b9cbf1ac7afd0a29ec3&scene=21#wechat_redirect)
20. [源码 | Flink Client 实现原理与源码解析](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247501043&idx=2&sn=c61470a64c1ae41af5740828943d2abd&chksm=cf37f6fef8407fe8db7d457a02ea5d6046a09af81948687430dd81e7e6ade61d7640f1d09dc7&scene=21#wechat_redirect)  
21. [实践 | 一文搞定实时数仓 CDC 案例实战](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247502046&idx=3&sn=63b24ca70ac3ac51a3ae1d43f6186e1c&chksm=cf37fad3f84073c5f4a544d8d0566f6462f0a285e5b09fa63d39a5e30b2975d9e2b003406e5e&scene=21#wechat_redirect)  
22. [实践 | Flink SQL 在字节跳动的实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247502511&idx=2&sn=d8ceb82ad3d749199ed32f0f819a08fb&chksm=cf37f8a2f84071b4e20a6d9cbef6009b7bd80cb800260eda5bc6c8fdbbaa58464fa956d26ed0&scene=21#wechat_redirect)  
23. 实践 | [Flink on Hive 构建流批一体实时数仓](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247501137&idx=1&sn=77ca5f487e7bdf4fa996801db4f1d91c&chksm=cf37f75cf8407e4a584265b392040a958a992fe3ac68b45100b81d673971b8f428dc63329cec&scene=21#wechat_redirect)  
24. [实践 | Flink + Iceberg  全场景实时数仓的建设实践](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247500469&idx=2&sn=a54599c96746d443257141156f810079&chksm=cf37f0b8f84079aecea364b42e17b1b6520c07db6780ea97d991e305b699d21da1b127aa2f91&scene=21#wechat_redirect)  
25. [实践 | Flink + ClickHouse 打造轻量级点击流实时数仓](http://mp.weixin.qq.com/s?__biz=Mzg3NjIyNjQwMg==&mid=2247504835&idx=2&sn=0249e935f8d9622d5af23cbb33e4140b&chksm=cf37e1cef84068d85849790cdf6bc794ca62bcb3a9210e9199f32dc7f3cf2c750eab23576da9&scene=21#wechat_redirect)

**编者寄语：** 

感谢大家一路陪伴。1000 + 个日日夜夜，如果你写过文章，你一定会懂。  

坚持比努力更可怕。

读者朋友学到东西，认可 "数据仓库与 Python 大数据" 的价值，职业生涯因此受益。

这才是我们坚持分享文章的初衷。

与君共勉。

![](https://mmbiz.qpic.cn/mmbiz_gif/V3ll7FMyGyMqQC7JRNgVZlsiaJibSyp27USlRia194K6Nqfvz8Wblg7HDceOn4Y3MekppS14lazRZTKLdt2BHuYGA/640?wx_fmt=gif)

**扫码关注公众号，接收更多干货！**

![](https://mmbiz.qpic.cn/mmbiz_png/1OYP1AZw0W1RBAeraR2GBy6APqmgKSMQnvBeVdsGrHxqq5dc7UyiaE3ddduxsoN1ictQdIkEUz8Rz3X5WqibUDticg/640?wx_fmt=jpeg)

**关于我们：** 

本公众号致力于建设大数据领域技术共享平台，3w + 关注，保持日更，每天 08:16 发文，为您提供优秀高质量的大数据领域的分享。欢迎推荐给同行朋友，加群或投稿或转载可加 v：iom1128，备注：数据，谢谢！ 
 [https://mp.weixin.qq.com/s/lztc1uQuqcSruAlSG1BLYQ](https://mp.weixin.qq.com/s/lztc1uQuqcSruAlSG1BLYQ)
