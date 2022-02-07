# 图解 ElasticSearch 原理
点击上方 “Java 基基”，选择 “设为星标”

做积极的人，而不是积极废人！

每天 **14:00** 更新文章，每天掉亿点点头发...

源码精品专栏

-   [原创 | Java 2021 超神之路，很肝~](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)  
-   [中文详细注释的开源项目](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486264&idx=1&sn=475ac3f1ef253a33daacf50477203c80&chksm=fa497489cd3efd9f7298f5da6aad0c443ae15f398436aff57cb2b734d6689e62ab43ae7857ac&scene=21#wechat_redirect)  
-   [RPC 框架 Dubbo 源码解析](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484647&idx=1&sn=9eb7e47d06faca20d530c70eec3b8d5c&chksm=fa497b56cd3ef2408f807e66e0903a5d16fbed149ef7374021302901d6e0260ad717d903e8d4&scene=21#wechat_redirect)
-   [网络应用框架 Netty 源码解析](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485054&idx=2&sn=9f3b85f7b8454634da6c5c2ded9b4dba&chksm=fa4979cfcd3ef0d9d2dd92d8d1bd8f1553abc6e2095a5d743e0b2c2afe4955ea2bbbd7a4b79d&token=55862109&lang=zh_CN&scene=21#wechat_redirect)
-   [消息中间件 RocketMQ 源码解析](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486256&idx=1&sn=81daccd3fcd2953456c917630636fb26&chksm=fa497481cd3efd97d9239f5eab060e49dea9876a6046eadba0effb878d2fb51f3ba5733e4c0b&scene=21#wechat_redirect)  
-   [数据库中间件 Sharding-JDBC 和 MyCAT 源码解析](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486257&idx=1&sn=4d3c9c675f8833157641a2e0b48e498c&chksm=fa497480cd3efd96fe17975b0b8b141e87fd0a62673e6a30b501460de80b3eb997056f09de08&scene=21#wechat_redirect)
-   [作业调度中间件 Elastic-Job 源码解析](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486258&idx=1&sn=ae5665ae9c3002b53f87cab44948a096&chksm=fa497483cd3efd950514da5a37160e7fd07f0a96f39265cf7ba3721985e5aadbdcbe7aafc34a&scene=21#wechat_redirect)
-   [分布式事务中间件 TCC-Transaction 源码解析](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486259&idx=1&sn=b023cf3dbf97e5da59db2f4ee632f5a6&chksm=fa497482cd3efd9402d71469f71863f71a6998b27e12ca2e00446b8178d79dcef0721d8e570a&scene=21#wechat_redirect)
-   [Eureka 和 Hystrix 源码解析](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486260&idx=1&sn=8f14c0c191d6f8df6eb34202f4ad9708&chksm=fa497485cd3efd93937143a648bc1b530bc7d1f6f8ad4bf2ec112ffe34dee80b474605c22db0&scene=21#wechat_redirect)
-   [Java 并发源码](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486261&idx=1&sn=bd69f26aadfc826f6313ffbb95e44ee5&chksm=fa497484cd3efd92352d6fb3d05ccbaebca2fafed6f18edbe5be70c99ba088db5c8a7a8080c1&scene=21#wechat_redirect)

[来源：cnblogs.com/richaaaard/  
p/5226334.html](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

-   [摘要](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [内容](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)


-   [图解 ElasticSearch](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [图解 Lucene](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [搜索发生时](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [缓存的故事](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [在 Shard 中搜索](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [如何 Scale](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
-   [一个真实的请求](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)

* * *

## [摘要](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

先自上而下，后自底向上的介绍 ElasticSearch 的底层工作原理，试图回答以下问题：

-   为什么我的搜索 \*_\_foo-bar__ \* 无法匹配 \_foo-bar_ ？
-   为什么增加更多的文件会压缩索引（Index）？
-   为什么 ElasticSearch 占用很多内存？

> “
>
> 推荐下自己做的 Spring Boot 的实战项目：
>
> [https://github.com/YunaiV/ruoyi-vue-pro](https://github.com/YunaiV/ruoyi-vue-pro)

## [内容](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

### [图解 ElasticSearch](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [云上的集群](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiast1tVqgIMGamfoCI84XsG0lkQHeSdiberSzDnrIFphH7qyb8pt4sR7A/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [集群里的盒子](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

云里面的每个白色正方形的盒子代表一个节点——Node。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiadX9Dne4vj8mNGBHRxHtbbByJNOeYC1cQ9hPR0XyuZ4jBfRn4YW4ZWw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [节点之间](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

在一个或者多个节点直接，多个绿色小方块组合在一起形成一个 ElasticSearch 的索引。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiao374xh1kdHHODXTI7HKvfzD8n5CvicLtibiaibUQiaJdPlpVwJwQiaPxFaicA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [索引里的小方块](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

在一个索引下，分布在多个节点里的绿色小方块称为分片——Shard。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaYEMCGpSv8rMsQCgDCdz8NtCbCcwjSscuAWIqVgvUoicRHCTgMKE3wibg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Shard＝Lucene Index](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

一个 ElasticSearch 的 Shard 本质上是一个 Lucene Index。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiakLgGkaUCiaSyC0EHIlCE8yibSIzljcfO3MiaPcwVdxjEanpcmtpUzHNlw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

Lucene 是一个 Full Text 搜索库（也有很多其他形式的搜索库），ElasticSearch 是建立在 Lucene 之上的。接下来的故事要说的大部分内容实际上是 ElasticSearch 如何基于 Lucene 工作的。

### [图解 Lucene](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Mini 索引——segment](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

在 Lucene 里面有很多小的 segment，我们可以把它们看成 Lucene 内部的 mini-index。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiajQA2dASmrb3fGu8Kt2tHVskwtzLE9UVGsDzK0241JJR9sGyLaXWs4w/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Segment 内部](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

有着许多数据结构

-   Inverted Index
-   Stored Fields
-   Document Values
-   Cache

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiakYM6iaDSUB8tiarvLCAiay4JQO4tVofUlSficLm63AODe5XGS4cWVdpoibw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [最最重要的 Inverted Index](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaBrUDNOnUiaSOjBDqb977k025xHRy4YQHxw1TDQlcZfP4HI4EpYhQcdQ/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

Inverted Index 主要包括两部分：

1.  一个有序的数据字典 Dictionary（包括单词 Term 和它出现的频率）。
2.  与单词 Term 对应的 Postings（即存在这个单词的文件）。

当我们搜索的时候，首先将搜索的内容分解，然后在字典里找到对应 Term，从而查找到与搜索相关的文件内容。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaFFMWGfJ14Le42lEw6auWSGficHc7Iy1beZzxztvapqKD8iaoBg2ibO7uQ/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

##### [查询 “the fury”](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaZ73Oy8JkcpWmsBQHJlNtz40UuS0poZ9kP9R0L1IA3L4gS6tX16L0gA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

##### [自动补全（AutoCompletion-Prefix）](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

如果想要查找以字母 “c” 开头的字母，可以简单的通过二分查找（Binary Search）在 Inverted Index 表中找到例如 “choice”、“coming” 这样的词（Term）。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaG0tloquGb6WicC7ibWGRuhh2HMXxp2NuVyMdEQXXswhuadGbccIBdiadw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

##### [昂贵的查找](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

如果想要查找所有包含 “our” 字母的单词，那么系统会扫描整个 Inverted Index，这是非常昂贵的。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiabfWowxuW5stwJswoUaaPuP2mUicbTxq6kVleLBc93jZJb21yNatxvpA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

在此种情况下，如果想要做优化，那么我们面对的问题是如何生成合适的 Term。

##### [问题的转化](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaIpJyl7gZoYIDaOyVn7dObYndicyibUylBXwqXiaegfpuyf2J7hLIG8qyw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

对于以上诸如此类的问题，我们可能会有几种可行的解决方案：

-   \* suffix -> xiffus \*

    如果我们想以后缀作为搜索条件，可以为 Term 做反向处理。
-   (60.6384, 6.5017) -> u4u8gyykk

    对于 GEO 位置信息，可以将它转换为 GEO Hash。
-   123 -> {1-hundreds, 12-tens, 123}

    对于简单的数字，可以为它生成多重形式的 Term。

##### [解决拼写错误](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

一个 Python 库 为单词生成了一个包含错误拼写信息的树形状态机，解决拼写错误的问题。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaKRDul3ga8g9hoBrxNQtOrJgkoaQprO9gPsbTlZSKUE8cpZHJUia60Mw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Stored Field 字段查找](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

当我们想要查找包含某个特定标题内容的文件时，Inverted Index 就不能很好的解决这个问题，所以 Lucene 提供了另外一种数据结构 Stored Fields 来解决这个问题。本质上，Stored Fields 是一个简单的键值对 key-value。默认情况下，ElasticSearch 会存储整个文件的 JSON source。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia0kfoANZH1uMqmFGvwJQ0Rw04m0jd004xJR5kXzBMVziatbNgHUaUFicg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Document Values 为了排序，聚合](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

即使这样，我们发现以上结构仍然无法解决诸如：排序、聚合、facet，因为我们可能会要读取大量不需要的信息。

所以，另一种数据结构解决了此种问题：Document Values。这种结构本质上就是一个列式的存储，它高度优化了具有相同类型的数据的存储结构。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaJ62TtQPeLXwzTVz5DPGPoqfLDI1OfQnEQI0cEvxicbfnoTqrGdGKib1g/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

为了提高效率，ElasticSearch 可以将索引下某一个 Document Value 全部读取到内存中进行操作，这大大提升访问速度，但是也同时会消耗掉大量的内存空间。

总之，这些数据结构 Inverted Index、Stored Fields、Document Values 及其缓存，都在 segment 内部。

### [搜索发生时](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

搜索时，Lucene 会搜索所有的 segment 然后将每个 segment 的搜索结果返回，最后合并呈现给客户。

Lucene 的一些特性使得这个过程非常重要：

-   Segments 是不可变的（immutable）


-   **Delete?** 当删除发生时，Lucene 做的只是将其标志位置为删除，但是文件还是会在它原来的地方，不会发生改变
-   **Update?** 所以对于更新来说，本质上它做的工作是：先**删除** ，然后**重新索引（Re-index）**


-   随处可见的压缩

    Lucene 非常擅长压缩数据，基本上所有教科书上的压缩方式，都能在 Lucene 中找到。
-   缓存所有的所有

    Lucene 也会将所有的信息做缓存，这大大提高了它的查询效率。

### [缓存的故事](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

当 ElasticSearch 索引一个文件的时候，会为文件建立相应的缓存，并且会定期（每秒）刷新这些数据，然后这些文件就可以被搜索到。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaIsJT9ibhWMv4pbNSuPJq4EkQpGDh3TlCBHscId5mVK28RCQ38pZdeGg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

随着时间的增加，我们会有很多 segments，

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaRaSAJtM57tt27TIdmGnCc2cKcUADXsdwIOWdIDxmiceUFw9HAbxgIiag/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

所以 ElasticSearch 会将这些 segment 合并，在这个过程中，segment 会最终被删除掉

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaVCVcpbrv9Nj7FzaNX7vLrU5bCia8M1PPEIScDTZpUn54EKcttxR12pw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

这就是为什么增加文件可能会使索引所占空间变小，它会引起 merge，从而可能会有更多的压缩。

#### [举个栗子](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

有两个 segment 将会 merge

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaQeeKjibibOVktJUltH4zbZmLkeC42wtnGicM2rRmb4SuiawrHjKltNC4eA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

这两个 segment 最终会被删除，然后合并成一个新的 segment

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiawSYZV16eYicWtR9IibviboUdBFxLVPl6UvFctyHURYJ30W4mQNBBhkpJw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

这时这个新的 segment 在缓存中处于 cold 状态，但是大多数 segment 仍然保持不变，处于 warm 状态。

以上场景经常在 Lucene Index 内部发生的。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaf6lsjFRZ66veP9XGQ9BlmB84EEAicJmnhLnrXmAiaLpAueTS4licl9qOg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

### [在 Shard 中搜索](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

ElasticSearch 从 Shard 中搜索的过程与 Lucene Segment 中搜索的过程类似。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia1blgXEx3BcHHQia9zGWH8X01jA1UXerMjcsv4G5ZXSRfDhXdPXnDeqg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

与在 Lucene Segment 中搜索不同的是，Shard 可能是分布在不同 Node 上的，所以在搜索与返回结果时，所有的信息都会通过网络传输。

需要注意的是：

1 次搜索查找 2 个 shard ＝ 2 次分别搜索 shard

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaCSPJHW4dJ89saicdEAjiaEk2M69s0IMod0NpXm1PmlaibPuPMVDnX0AVQ/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [对于日志文件的处理](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

当我们想搜索特定日期产生的日志时，通过根据时间戳对日志文件进行分块与索引，会极大提高搜索效率。

当我们想要删除旧的数据时也非常方便，只需删除老的索引即可。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiafYLSyuZUy4TMd4O5CCABAzYByCkxJDZdxVfHh7mws9uzw1O8TQecfg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

在上种情况下，每个 index 有两个 shards

### [如何 Scale](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoian5R9yukiaxDDa0InDH2TXYEOibzd1vE5icVYtw5BZJqvWPxNLHXX4elJw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

shard 不会进行更进一步的拆分，但是 shard 可能会被转移到不同节点上

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoian5R9yukiaxDDa0InDH2TXYEOibzd1vE5icVYtw5BZJqvWPxNLHXX4elJw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

所以，如果当集群节点压力增长到一定的程度，我们可能会考虑增加新的节点，这就会要求我们对所有数据进行重新索引，这是我们不太希望看到的，所以我们需要在规划的时候就考虑清楚，如何去平衡足够多的节点与不足节点之间的关系。

#### [节点分配与 Shard 优化](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

-   为更重要的数据索引节点，分配性能更好的机器
-   确保每个 shard 都有副本信息 replica

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaHsrI3HjM2kgfI5u3hkIe5ZGyyuGRwC3h9ZmFmkgMot99ewfsGf6kMg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [路由 Routing](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

每个节点，每个都存留一份路由表，所以当请求到任何一个节点时，ElasticSearch 都有能力将请求转发到期望节点的 shard 进一步处理。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaBqLCgctmG5CHusibCerBGhnjJv3I1nUlnicMuaZkQvA6icAUJJic4H8orw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

### [一个真实的请求](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia7h2GicibXoFDDEnHpGDSgHlWUyxGw6pe0CHsiaxEXIHB7QEv9vLas2gDg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [Query](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia7h2GicibXoFDDEnHpGDSgHlWUyxGw6pe0CHsiaxEXIHB7QEv9vLas2gDg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

Query 有一个类型 filtered，以及一个 multi_match 的查询

#### [Aggregation](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiae8IyqB7VtibsicHVa3xwIb2rkp9mbJKUjxCJ068BoSgc0GI7znQbGsRw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

根据作者进行聚合，得到 top10 的 hits 的 top10 作者的信息

#### [请求分发](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

这个请求可能被分发到集群里的任意一个节点

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaciaVSJoWUf0UgSU7ib25mic0As79d8lHn7oKTuGRtUYrrvSKLHYHsQwGg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [上帝节点](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiars9kMic3mjQS5nicnuj0Kk3KqA8mg1SAklrTfr6qqVDsfcpHliaPFZXZg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

这时这个节点就成为当前请求的协调者（Coordinator），它决定：

-   根据索引信息，判断请求会被路由到哪个核心节点
-   以及哪个副本是可用的
-   等等

#### [路由](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaFP2qFRcUeEMNWr7A6dnmk6NfcP8RcWLMFQib8sbCicD64ic4Rfuazibaibg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

#### [在真实搜索之前](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

ElasticSearch 会将 Query 转换成 Lucene Query

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoianCVG2J6iaBwXw1DrAhicZoAVXOV288pfG4mTKA2zQPVcibHeuspLMCIgg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

然后在所有的 segment 中执行计算

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia1v8ibAClZLtHuDm4nwTKQsQNaA2ibKBnTmMia91iacAicJzqqKibWHXFWXKA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

对于 Filter 条件本身也会有缓存

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiatOic0AiavPsMiaCnVmJ1V9ZvhibIqnLsqJic34XXnRicItapU8sQicdy8oJbg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

但 queries 不会被缓存，所以如果相同的 Query 重复执行，应用程序自己需要做缓存

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaiaIuYUwFQTYsxCwz7ib0n4KzOJmiadzoFSaeutSOoSbUNricUgLIdI9rWA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

所以，

-   filters 可以在任何时候使用
-   query 只有在需要 score 的时候才使用

#### [返回](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

搜索结束之后，结果会沿着下行的路径向上逐层返回。

[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaWic1p9c7yNNM0qiap9fUORX8quRn4QATlicUrGOl9XxjQdUOXGk2rg15w/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoia3R73pCql8P7SU0NEfCRCGpbh4KtgvmuCNMiaJgCiaO0IsBgcUeuQrVOQ/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaKaQ8VfgXgJshnxJcRUwibSCyS3sicjCF5gNUDe96awJGgkOORchXP8pA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiatJT5wI9c6erRpwtqrcuRewlQAEvjRg1VVcBZm2X9u6ToqDZMQ1b5fg/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWfHbOMIG122pqeTSCmReoiaJYYz7mozQicZLlZCmedCicZs9d8lg7XicKfBt3mLVz4cJa4g9pxqhl0Ug/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

* * *

* * *

欢迎加入我的知识星球，一起探讨架构，交流源码。加入方式，**长按下方二维码噢**：  

![](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupWvCPbtnO1ap7Octib250ghicPloeaiakSGrRLuGAdvNpA7oHpJV74U7ZQsiaLiaP9Wrxgc68F7dteNIIQ/640?wx_fmt=png)

已在知识星球更新源码解析如下：  

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWVian1HZ76BOz52l4pkqX0IMicM14rRFyiaO0vQENMOufUhDVVtPiadDdoKjQ/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWViaExAeJx1CZeSaJ9qxh0X70s4JGjIVVlT5ZqBGu51YTedMNfO49DKb6g/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWViaCibrYIXNgebWPd5g7Or9dcToN660aLAEJEhz4wLpBBiaFhejsaGDGd2g/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWVia4qmv743xvlia1HYmqCDPBLpo3HXtw8Hmo76GkGK5wCqvicAKxd9ET3ow/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)

最近更新《芋道 SpringBoot 2.X 入门》系列，已经 101 余篇，覆盖了 MyBatis、Redis、MongoDB、ES、分库分表、读写分离、SpringMVC、Webflux、权限、WebSocket、Dubbo、RabbitMQ、RocketMQ、Kafka、性能测试等等内容。  

提供近 3W 行代码的 SpringBoot 示例，以及超 6W 行代码的电商微服务项目。

获取方式：点 “**在看**”，关注公众号并回复 **666** 领取，更多内容陆续奉上。

```


**文章有帮助的话，在看，转发吧。** 

**谢谢支持哟 (*^__^*）**


```

 [https://mp.weixin.qq.com/s/P3MJw2gcfjCqtGxKq0J1AA](https://mp.weixin.qq.com/s/P3MJw2gcfjCqtGxKq0J1AA)
