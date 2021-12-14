# 大佬都在用的方法：Spark 实现 分组 TopN 问题！
今天给大家分享一个大数据很常见的场景案例，分组取 TopN 问题。

大数据中按照某个 Key 进行分组，找出每个组内数据的 topN 时，这种情况就 是分组取 topN 问题。

解决分组取 TopN 问题有两种方式，第一种就是直接分组，对分组内的数据使用原生集合进行排序处理。在大数据场景，数据量特别大，性能也非常差。

考虑到每个人使用的语言不同，这次分别出了 2 个版本：Java 和 Scala。

### 原生集合排序取 TopN

先看看数据源，因为只是案例，所以所以只造了十几条数据，但其实代码逻辑是一样的。

![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFYZI7Z6NTfblmyv10EpvXibY16zO4nEDgmnGheoUutvj0ib6UA2qyG0DA/640?wx_fmt=png)

第一种使用原生集合排序，其代码逻辑就是取到每一条数据，然后排序，之后打印就好了（在实际场景中，可能是拿到前 3 做一些其它业务操作）

但数据量多，会占用 Executor 内存过多，有可能会导致 Executor  oom 问题。Java 代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFYjicGk8rH80eiaatScW7iadH1FrMrZhP28pfwhUT5iahO5zayvujFIuITw/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFo4kDAMZEKmX82TCQs2ZNx1I9PEAKwJpBpPyiawuxomibf2RFIjVRSS8A/640?wx_fmt=png)

Scala 代码如下，是不是觉得特别简洁：

![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vF2XHITgGvZPcHmzAuuoURvvmYaVVseVbmicOIc5VujQj3dw1dNictA5qQ/640?wx_fmt=png)

### 定长数组方式取 TopN

第二种方式就是直接使用定长数组的方式解决分组取 topN 问题，代码逻辑是先设定好一个 N 大小的数组（本案例假设 Top 3），并赋初始值 0，通过遍历数据集，比较大小，把最大的放到最前面，遇到更大的数，则把定长数组中的数往后移动，直到遍历完数据集，则定长数组便是 TopN 数据。

话不多说，直接上代码。

Java 版本：

![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFmkgMVicW55mPsICHWPfl0Tq5ibH5RIbYnoCbbSvmP6H7lIwqhPUFaGCg/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFbcLRNcJe4EkiayVLWzPhicnWYMeZFBOgxCdtsvC6ITJLQhpGvJBdmmBA/640?wx_fmt=png)

Scala 版本：![](https://mmbiz.qpic.cn/mmbiz_png/lH7qcFOtBkgXY3kdp9PqYWU16xghc8vFujU9RPfbaXvM2JPxRiagdcXTFicCjPUw1s1icyibEgQnjFXiazGBflboIDQ/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/1frknnECQO-h7msHFqhEJQ](https://mp.weixin.qq.com/s/1frknnECQO-h7msHFqhEJQ)
