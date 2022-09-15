# 实战引入Elasticsearch的系统架构
架构师（JiaGouX）

我们都是架构师！  
架构未来，你来不来？

**前 言**

对于该问题我相信大家就算没有面试被问到过，现实工作中同事之间的合作也会遇到。　

因此从我的角度重新去回答这个问题，有以下几点：

1\.**师出有名**，在软件工程里是针对问题场景提供解决方案的，如果脱离的实际问题（需求）去做技术选型，无疑是耍流氓。

大家可以回顾身边的 “架构师”、“技术 Leader” 是不是拍拍脑袋做决定，问他们为什么这么做，可能连个冠冕堂皇的理由都给不出。

2\.**信任度**，只有基于上面的条件，你才有理由建议引入新技术。领导愿不愿意引入新技术有很多原因：领导不了解这技术、领导偏保守、领导不是做技术的等。

那么我认为这几种都是信任度，这种 **信任度分人和事**，人就是引入技术的提出者，事就是提出引入的技术。

3\.**尽人事**，任何问题只是单纯解决 **事** 都是简单的，以我以往的做法，把基本资料收集全并以通俗易懂的方式归纳与讲解，最好能提供一些能量化的数据，这样更加有说服力。

知识普及 OK 后，就可以尝试写方案与做个 Demo，方案最好可以提供多个，可以分短期收益与长期收益的。完成上面几点可以说已经尽人事了，如果领导还不答应那么的确有他的顾虑，就算无法落实，到目前为止的收获也不错。

4\.**复杂的是人**，任何人都无法时刻站在理智与客观的角度去看待问题，**事是由人去办的**，所以同一件事由不同的人说出来的效果也不一样。

因此得学会向上管理、保持与同事之间合作融洽度，尽早的建立合作信任。本篇文章更多叙述的事，因此人方面不过多深究，有兴趣的我可以介绍一本书**《知行 技术人的管理之路》**。

本篇我的实践做法与上述一样，除了 4 无法体现。那么下文我分了 4 大模块：**业务背景介绍、基础概念讲解、方案的选用与技术细节**。

该篇文章不包含代码有 8000 多千字，花了我 3 天时间写，可能需要您花 10 分钟慢慢阅读，我承诺大家正文里面细节满满。

曾有朋友建议我拆开来写，但是我的习惯还是希望以一篇文章，这样更加系统化的展示给大家。当然大家有什么建议也可以在下方留言给我。

部分源码，我放到了[https://github.com/SkyChenSky/Sikiro](https://github.com/SkyChenSky/Sikiro) 的 Sikiro.ES.Api 里

**背 景**

而部分非核心业务原本应该超亿的量级了，但是因为从物理表的设计优化上进行了数据压缩，导致维持在一个比较稳定的数量。压缩数据虽然能减少存储量，优化提供一定的性能，但是同时带来的损失了业务可扩展性。

举个例子：

-   我们平台某个用户拥有**最后访问作品记录**和**总的阅读时长**，但是没有**某个用户的阅读明细**，那么这样的设计就会导致后续新增一个**抽奖业务**，需要在某个时间段内阅读了多长时间或者章节数量的作品，才能参加与抽奖；
-   或者运营想通过阅读记录统计或者分析出，**用户的爱好**和 **受欢迎的作品**。现有的设计对以上两种业务情况都是无法满足的。

此外我们平台还有作品搜索功能，like ‘% 搜索 %’查询是不走索引的而走 **全表扫描**，一张表 42W 全表扫描，数据库服务器配置可以的情况下还是可以的，但是存在 **并发请求**时候，资源消耗就特别厉害了，特别是在偶尔被爬虫爬取数据。

（我们平台 API 的并发峰值能达到 8w/s，每天的接口在淡季请求次数达到了 1 亿 1 千万）

关系型数据库拥有 **ACID 特性**，能通过金融级的事务达成数据的一致性，然而它却没有横向扩展性，只要在海量数据场景下，单实例，无论怎么在关系型数据库做优化，都是只是治标。

而 NoSQL 的出现很好的弥补了关系型数据库的短板，在马丁福勒所著的**《NoSQL 精粹》**对 NoSQL 进行了分类：**文档型、图形、列式，键值**，

从我的角度其实可以把搜索引擎纳入 NoSQL 范畴，因为它的确满足的 NoSQL 的 4 大特性：**易扩展、大数据量高性能、灵活的数据模型、高可用**。

我看过一些同行的见解，把 Elasticsearch 归为**文档型 NoSQL**，我个人是没有给他下过于明确的定义，这个上面说法大家见仁见智。

MongoDB 作为文档型数据库也属于我的技术选型范围，它的读写性能高且平衡、数据分片与横向扩展等都非常适合我们平台部分场景，最后我还是选择 Elasticsearch。原因有三：

-   我们运维相比于 MongoDB 更熟悉 Elasticsearch。
-   我们接下来有一些统计报表类的需求，Elastic Stack 的各种工具能很好满足我们的需求。
-   我们目前着手处理的场景以非实时、纯读为主的业务，Elasticsearch 近实时搜索已经能满足我们。

**Elasticsearch 优缺点**

> 百度百科 ：
>
> Elasticsearch 是一个基于 Lucene 的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于 RESTful web 接口。Elasticsearch 由 Java 语言开发的，是一种流行的企业级搜索引擎。Elasticsearch 用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。官方客户端在 Java、.NET（C#）、PHP、Python、Apache Groovy、Ruby 和许多其他语言中都是可用的。

对于满足当下的业务需求和未来支持海量数据的搜索，我选择了 Elasticsearch，其实原因主要以下几点：

\| **优点** \| **描述** \|
| 横向可扩展性 | 可单机、可集群，横向扩展非常简单方便，自动整理数据分片 |
| 快 | 索引被分为多个分片(Shard)，利用多台服务器，使用了分而治之的思想提升处理效率 |
| 支持搜索多样化 | 与传统关系型数据库相比，ES 提供了全文检索、同义词处理、相关度排名、复杂数据分析、海量数据的近实时处理等功能 |
| 高可用 | 提供副本 (Replica) 机制，一个分片可以设置多个副本，假如某服务器宕机后，集群仍能正常工作。 |
| 开箱即用 | 简易的运维部署，提供基于 Restful API，多种语言的 SDK |

那么我个人认为 Elasticsearch 比较大的 **缺点**只有 **吃内存**，具体原因可以看下文内存读取部分。

**Elasticsearch 为什么快？**

-   **内存读取**
-   **多种索引**


-   **倒排索引**
-   **doc values**


-   **集群分片**

## **内存读取**

Elasticsearch 是基于 Lucene， 而 Lucene 被设计为可以利用操作系统底层机制来缓存内存数据结构，换句话说 Elasticsearch 是依赖于操作系统底层的 **Filesystem Cache**，

查询时，操作系统会将磁盘文件里的数据自动缓存到 Filesystem Cache 里面去，因此要求 Elasticsearch 性能足够高，那么就需要服务器的提供的**足够内存给 Filesystem Cache 覆盖存储的数据**。

上一段最后一句话什么意思呢？

假如：Elasticsearch 节点有 3 台服务器各 64G 内存，3 台总内存就是 64 \* 3 = 192G。

每台机器给 Elasticsearch  jvm heap 是 32G，那么每服务器留给 **Filesystem Cache** 的就是 32G（50%），而集群里的 **Filesystem Cache** 的就是 32 \* 3 = 96G 内存。

此时，在 3 台 Elasticsearch 服务器共占用了 1T 的磁盘容量，那么每台机器的数据量约等于 341G，意味着每台服务器只有大概**10 分之 1 数据是缓存**在内存的，其余都得走硬盘。

说到这里大家未必会有一个直观得认识，因此我从**《大型网站技术架构：核心原理与案例分析》**第 36 页抠了一张表格下来：

\| **操作** \| **响应时间** \|
| 打开一个网站 | 几秒 |
| 在数据库中查询一条记录（有索引） | 十几毫秒 |
| 机械磁盘一次寻址定位 | 4 毫秒 |
\| **从机械磁盘顺序读取 1MB 数据** \| **2 毫秒** \|
| 从 SSD 磁盘顺序读取 1MB 数据 | 0.3 毫秒 |
| 从远程分布式缓存 Redis 读取一个数据 | 0.5 毫秒 |
\| **从内存中读取 1MB 数据** \| **十几微秒** \|
| Java 程序本地方法调用 | 几微秒 |
| 网络传输 2KB 数据 | 1 微秒 |

从上图加粗项看出，**内存读取性能**是机械磁盘的**200 倍**，是 SSD 磁盘约等于 30 倍，**假如读一次 Elasticsearch 走内存场景下耗时 20 毫秒，那么走机械硬盘就得 4 秒，走 SSD 磁盘可能约等于 0.6 秒**。讲到这里我相信大家对是否走内存的性能差异有一个直观的认识。

对于 Elasticsearch 有很多种索引类型，但是我认为核心主要是倒排索引和 doc values

## **倒排索引**

Lucene 将写入索引的所有信息组织为倒排索引（inverted index）的结构形式。倒排索引是一种将分词映射到文档的数据结构，可以认为倒排索引是面向分词的而不是面向文档的。

假设在测试环境的 Elasticsearch 存放了有以下三个文档：

-   Elasticsearch Server（**文档 1**）
-   Masterring Elasticsearch（**文档 2**）
-   Apache Solr 4 Cookbook（**文档 3**）

　　以上文档索引建好后，简略显示如下：

\| **词项** \| **数量** \| **文档** \|
| 4 | 1 | &lt;3> |
| Apache | 1 | &lt;3> |
| Cookbook | 1 | &lt;3> |
| Elasticsearch | 2 | &lt;1>&lt;2> |
| Mastering | 1 | &lt;1> |
| Server | 1 | &lt;1> |
| Solr | 1 | &lt;3> |

如上表格所示，每个词项指向该词项所出现过的文档位置，这种索引结构允许快速、有效的搜索出数据。

## **doc values**

对于分组、聚合、排序等某些功能来说，倒排索引的方式并不是最佳选择，这类功能操作的是文档而不是词项，这个时候就得把倒排索引逆转过来成正排索引，这么做会有两个缺点：

-   **构建时间长**
-   **内存占用大，易 OutOfMemory，且影响垃圾回收**

Lucene 4.0 之后版本引入了 doc values 和额外的数据结构来解决上面得问题，目前有五种类型的 doc values：**NUMERIC、BINARY、SORTED、SORTED_SET、SORTED_NUMERIC**，针对每种类型 Lucene 都有特定的压缩方法。

doc values 是列式存储的正排索引，通过 docID 可以快速读取到该 doc 的特定字段的值，列式存储存储对于聚合计算有非常高的性能。

## **集群分片**

**Elasticsearch**可以简单、快速利用多节点服务器形成**集群**，以此分摊服务器的**执行压力**。

此外数据可以进行**分片存储**，搜索时**并发**到不同服务器上的主分片进行搜索。

这里可以简单讲述下 Elasticsearch 查询原理，Elasticsearch 的查询分两个阶段：**分散阶段**与**合并阶段**。

任意一个 Elasticsearch 节点都可以接受客户端的请求。接受到请求后，就是**分散阶段**，并行发送子查询给其他节点；

然后是**合并阶段**，则从众多分片中收集返回结果，然后对他们进行合并、排序、取长等后续操作。最终将结果返回给客户端。

**机制如下图：** 

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYm7keCBhbjmfib7STZ2PTB1gYIHBbrE41GXztGQRia0XiaPic0JetSqGiasz1GicQgTaWIO0qJCnza7rGA/640?wx_fmt=png)

### **分页深度陷阱**

基于以上查询的原理，扩展一个分页深度的问题。

现需要查页长为 10、第 100 页的数据，实际上是会把每个 Shard 上存储的前 1000（10\*100） 条数据都查到一个协调节点上。

如果有 5 个 Shard，那么就有 5000 条数据，接着协调节点对这 5000 条数据进行一些合并、处理，再获取到最终第 100 页的 10 条数据。也就是实际上查的数据总量为 pageSize\*pageIndex\*shard，页数越深则查询的越慢。

因此 ElasticSearch 也会有要求，每次查询出来的**数据总数不会返回超过 10000 条**。

那么从业务上尽可能跟产品沟通避免分页跳转，使用滚动加载。而**Elasticsearch**使用的相关技术是 search_after、scroll_id。

**ElasticSearch 与数据库基本概念对比**

\| **ElasticSearch** \| **RDBMS** \|
| Index | 表 |
| Document | 行  
 \|
| Field | 列 |
| Mapping | 表结构 |

在**Elasticsearch 7.0**版本之前（&lt;7.0），有 type 的概念，而**Elasticsearch**和**关系型数据库**的关系是，index = database、type = table，但是在**Elasticsearch 7.0**版本后（>=7.0）弱化了 type 默认为\_doc，而官方会在 8.0 之后会彻底移除 type。  

**服务器选型**

在官方文档（[https://www.elastic.co/guide/cn/elasticsearch/guide/current/heap-sizing.html）里建议 \*\*Elasticsearch](https://www.elastic.co/guide/cn/elasticsearch/guide/current/heap-sizing.html）里建议**Elasticsearch)  JVM Heap**最大为**32G\*\*，同时不超过服务器内存的一半，

也就是说内存分别为 128G 和 64G 的服务器，JVM Heap 最大只需要设置 32G；而 32G 服务器，则建议 JVM Heap 最大 16G，剩余的内存将会给到**Filesystem Cache**充分使用。

如果不需要对分词字符串做聚合计算（例如，不需要 fielddata ）可以考虑降低 JVM Heap。JVM Heap 越小，会导致 Elasticsearch 的 GC 频率更高，但 Lucene 就可以的使用更多的内存，这样性能就会更高。

对于我们公司的未来新增业务还会有收集用户的访问记录来统计**PV(page view)、UV(user view)**，有一定的聚合计算，经过多方便的考虑与讨论，**平衡成本与需求**后选择了腾讯云的三台配置为 CPU 16 核、内存 64G，SSD 云硬盘的服务器，并给与 Elasticsearch **配置 JVM Heap = 32G**。

**需求场景选择**

经过商讨，选择了两个业务场景，**用户阅读作品的记录明细**与**作品搜索**，选择这两个业务场景原因如下：

-   **写场景**

我们平台的用户黏度比较高，阅读作品是一个高频率的调用，因此**用户阅读作品的记录明细**可在短时间内造成海量数据的场景。（现一个月已达到了 70G 的数据量，共 1 亿 1 千万条）

-   **读场景**

阅读记录需提供给未来新增的抽奖业务使用，可从阅读章节数、阅读时长等进行搜索计算。

**作品搜索**原有实现是通过关系型数据库 like 查询，已是具有潜在的性能问题与资源消耗的业务场景

对于上述两个业务，**用户阅读作品的记录明细**与**抽奖业务**属于新增业务，对于在投入成本相对较少，也无需过多的需要兼容旧业务的压力。

而作品搜索业务属于优化改造，得保证兼容原有的用户搜索习惯前提下，新增拼音搜索。同时最好以扩展的方式，尽可能的减少代码修改范围，如果使用效果不好，随时可以回滚到旧的实现方式。

**设计方案**

我使用. Net 5 WebApi 将 Elasticsearch 封装成 ES 业务服务 API，这样的做法主要用来隐藏技术细节（时区、分词器、类型转换等），暴露粗粒度的读写接口。

这种做法在马丁福勒所著的**《NoSQL 精粹》**称把数据库视为 “**应用程序数据库**”，简单来说就是只能通过**应用间接**的访问存储，对于这个应用由一个团队负责维护开发，也只有这个团队才知道存储的结构。

这样通过封装的 API 服务解耦了外部 API 服务与存储，调用方就无需过多关注存储的特性，像 Mongodb 与 Elasticsearch 这种**无模式**的存储，无需优先定义结构，换而言之就是对于存储已有结构可随意修改扩展，那么 “**应用程序数据库**” 的做法也避免了其他团队无意侵入的修改。

考虑到现在业务需求复杂度相对简单，MQ 消费端也一起集成到 ES 业务服务，若后续 MQ 消费业务持续增多，再考虑把 MQ 消费业务抽离到一个（或多个的）消费端进程。

目前以 **同步读、同步写、异步写**的三种交互方式，进行与其他服务通信。

## **阅读记录明细**

本需求是完全新增，因此引入相对简单，只需要在**【平台 API】** 使用**【RabbitMQ】** 进行解耦，使用异步方式写入 Elasticsearch，使用队列除了用来解耦，还对此用来缓冲高并发写压力的情况。

对于后续新增的业务例如抽奖服务，则只需要通过 RPC 框架对接**ES 业务 API**，以同步读取的方式查询数据。

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYm7keCBhbjmfib7STZ2PTB15CSwJ8GPiaKBMJ2gGHRXiccHmHm5BMibVw5zlOiaEZmqhRPl321TWsibRzg/640?wx_fmt=png)

## **作品搜索**

对于该业务，我第一反应采用 CQRS 的思想，原有的写入逻辑我无需过多的关注与了解，因此我只需要想办法把关系型数据库的**数据同步**到 Elasticsearch，然后提供**业务查询 API**替换原有**平台 API 的数据源**即可。

那么数据同步则一般都是分**推**和**拉**两种方式。

## **推**

推的实时性无疑是比拉要高，只需**增量的**推送做写入的数据（增、删、改）即可，无论是从性能、资源利用、时效各方面来看都比拉更有效。

实施该方案，可以选择**Debezium**和**SQL Server**开启**CDC 功能**。

**Debezium**由 RedHat 开源的，同时需要依赖于 kafka 的，一个将多种数据源实时变更数据捕获，形成数据流输出的开源工具，同类产品有 Canal, DataBus, Maxwell。

**CDC**全称 Change Data Capture，直接翻译过来为变更数据捕获，核心为监测服务捕获数据库的写操作（插入，更新，删除），将这些变更按发生的顺序完整记录下来。

**我个人在我博客文章多次强调架构设计的输入核心为两点：满足需求与组织架构**，在满足需求的前提应优先选择简单、合适的方案。技术选型应需要考虑自己的团队是否可以支撑。

在上述无论是额外加入 Debezium 和 kafka，还是需要针对 SQL Server 开启 CDC 都超出了我们运维所能承受的极限，引入新的中间件和技术是需要试错的，而试错是需要额外高的成本，在未知的情况下引入更多的未知，只会造成更大的成本和不可控。

## **拉**

拉无疑是最简单最合适的实现方式，只需要使用调度任务服务，每隔段时间定时去从数据库拉取数据写入到 Elasticsearch 就可。

然而拉取数据，分**全量同步**与**增量同步**：

对于**增量同步**，只需要每次查询数据源**Select \* From Table_A Where RowVersion > LastUpdateVersion**，则可以过滤出需要同步的数据。

但是这个方式有点致命的缺点，数据源已被删除的数据是无法查询出来的，如果把 Elasticsearch 反向去跟 SQL Server 数据做对比又是一件比较愚蠢的方式，因此只能放弃该方式。

而**全量同步**，只要每次从 SQL Server 数据源全量新增到 Elasticsearch，并替换旧的 Elasticsearch 的 Index，因此该方案得全删全增。

但是这里又引申出新的问题，如果先删后增，那么在删除后再新增的这段真空期怎么办？

假如有 5 分钟的真空期是没有数据，用户就无法使用搜索功能。那么只能先增后删，先新增到一个 Index_Temp，全量新增完后，把原有 Index 改名成 Index_Delete，然后再把 Index_Temp 改成 Index，最后把 Index_Delete 删除。

这么一套操作下来，有没有觉得很繁琐很费劲？Elasticsearch 有一个叫别名（Aliases）的功能，别名可以一对多的指向多个 Index，也可以以原子性的进行别名指向 Index 的切换，具体实现可以看下文。

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYm7keCBhbjmfib7STZ2PTB1QY46WWZeX93qQgrp5Nb5gAc4zicCydz5dqOOpicBZia7vJz6JL1TSkPaQ/640?wx_fmt=png)

**阅读记录实现细节**

优先定义了个抽象类 ElasticsearchEntity 进行复用，对于实体定义有三个注意的细节点：

1\. 对于 ElasticsearchEntity 我定义两个属性\_id 与 Timestamp，Elasticsearch 是**无模式的（无需预定义结构）**，如果实体本身没有\_id，写入到 Elasticsearch 会自动生成一个\_id，为了后续的使用便捷性，我仍然自主定义了一个。

2\. 基于上述的分页深度的问题，因此在后续涉及的业务尽可能会以 search_after + 滚动加载的方式落实到我们的业务。

原本我们只需要使用 DateTime 类型的字段用 DateTime.Now 记录后，再使用 search_after 后会自动把 DateTime 类型字段转换成毫秒级的 Timestamp，

但是我在实现 demo 的时候，去制造数据，在程序里以 for 循环 new 数据的时候，发现生成的速度会在微秒级之间，那么假设用毫秒级的 Timestamp 进行 search_after 过滤，同一个毫秒有 4、5 条数据，那么容易在使用滚动加载时候少加载了几条数据，这样就到导致数据返回不准确了。

因此我扩展了个**\[DateTime.Now.DateTimeToTimestampOfMicrosecond()]**生成微秒级的 Timestamp，以此尽可能减少出现漏加载数据的情况。

3\. 对于 Elasticsearch 的操作实体的日期时间类型均以 DateTimeOffset 类型声明，因为 Elasticsearch 存储的是 UTC 时间，而且会因为 Http 请求的日期格式不同导致存放的日期时间也会有所偏差，为了避免日期问题使用 DateTimeOffset 类型是一种保险的做法。

而对于 WebAPI 接口或者 MQ 的 Message 接受的时间类型可以使用 DateTime 类型，DTO(传输对象) 与 DO（持久化对象）使用 Mapster 或者 AutoMapper 类似的对象映射工具进行转换即可。

（注意 DateTimeOffset 转 DateTime 得定义转换规则 \[TypeAdapterConfig&lt;DateTimeOffset,DateTime>.NewConfig().MapWith(dateTimeOffset=> dateTimeOffset.LocalDateTime)]）。

**如此一来，把 Elasticsearch 操作细节隐藏在 WebAPI 里，以友好、简单的接口暴露给开发者使用，降低了开发者对技术细节认知负担。** 

```cs
    [ElasticsearchType(RelationName = "user_view_duration")]
    public class UserViewDuration : ElasticsearchEntity
    {
        
        
        
        [Number(NumberType.Long, Name = "entity_id")]
        public long EntityId { get; set; }

        
        
        
        [Number(NumberType.Long, Name = "entity_type")]
        public long EntityType { get; set; }

        
        
        
        [Number(NumberType.Long, Name = "charpter_id")]
        public long CharpterId { get; set; }

        
        
        
        [Number(NumberType.Long, Name = "user_id")]
        public long UserId { get; set; }

        
        
        
        [Date(Name = "create_datetime")]
        public DateTimeOffset CreateDateTime { get; set; }

        
        
        
        [Number(NumberType.Long, Name = "duration")]
        public long Duration { get; set; }

        
        
        
        [Ip(Name = "Ip")]
        public string Ip { get; set; }
    }
```

```cs
public abstract class ElasticsearchEntity
    {
        private Guid? _id;

        public Guid Id
        {
            get
            {
                _id ??= Guid.NewGuid();
                return _id.Value;
            }
            set => _id = value;
        }

        private long? _timestamp;

        [Number(NumberType.Long, Name = "timestamp")]
        public long Timestamp
        {
            get
            {
                _timestamp ??= DateTime.Now.DateTimeToTimestampOfMicrosecond();
                return _timestamp.Value;
            }
            set => _timestamp = value;
        }
    }
```

异步写入  

* * *

**对于异步写入有两个细节点：** 

1\. 该数据从 RabbtiMQ 订阅消费写入到 Elasticsearch，从下面代码可以看出，我刻意以月的维度建立 Index，格式为 userviewrecord-2021-12，这么做的目的是为了方便管理 Index 和资源利用，有需要的情况下会删除旧的 Index。

2\. 消息订阅与 WebAPI 暂时集成到同一个进程，这样做主要是开发、部署都方便，如果后续订阅多了，在把消息订阅相关的业务抽离到独立的进程。

**按需演变，避免过度设计**

订阅消费逻辑

```typescript
public class UserViewDurationConsumer : BaseConsumer<UserViewDurationMessage>
    {
        private readonly ElasticClient _elasticClient;

        public UserViewDurationConsumer(ElasticClient elasticClient)
        {
            _elasticClient = elasticClient;
        }

        public override void Excute(UserViewDurationMessage msg)
        {
            var document = msg.MapTo<Entity.UserViewDuration>();

            var result = _elasticClient.Create(document, a => a.Index(typeof(Entity.UserViewDuration).GetRelationName() + "-" + msg.CreateDateTime.ToString("yyyy-MM"))).GetApiResult();
            if (result.Failed)
                LoggerHelper.WriteToFile(result.Message);
        }
    }
```

```typescript
/// <summary>
    /// 订阅消费
    /// </summary>
    public static class ConsumerExtension
    {
        public static IApplicationBuilder UseSubscribe<T, TConsumer>(this IApplicationBuilder appBuilder, IHostApplicationLifetime lifetime) where T : EasyNetQEntity, new() where TConsumer : BaseConsumer<T>
        {
            var bus = appBuilder.ApplicationServices.GetRequiredService<IBus>();
            var consumer = appBuilder.ApplicationServices.GetRequiredService<TConsumer>();

            lifetime.ApplicationStarted.Register(() =>
            {
                bus.Subscribe<T>(msg => consumer.Excute(msg));
            });

            lifetime.ApplicationStopped.Register(() => bus?.Dispose());

            return appBuilder;
        }
    }
```

订阅与注入

```cs
public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            ......
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env, IHostApplicationLifetime lifetime)
        {
            app.UseAllElasticApm(Configuration);

            app.UseHealthChecks("/health");

            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "SF.ES.Api v1");
                c.RoutePrefix = "";
            });

            app.UseRouting();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });

            app.UseSubscribe<UserViewDurationMessage, UserViewDurationConsumer>(lifetime);
        }
    }
```

**查询接口**

**查询接口此处有两个细节点：** 

如果不确定月份，则使用通配符查询 userviewrecord-\*，当然有需要的也可以使用别名处理。

因为 Elasticsearch 是记录 UTC 时间，因此时间查询得指定 TimeZone。

```typescript
[HttpGet]
        [Route("record")]
        public ApiResult<List<UserMarkRecordGetRecordResponse>> GetRecord([FromQuery] UserViewDurationRecordGetRequest request)
        {
            var dataList = new List<UserMarkRecordGetRecordResponse>();

            string dateTime;

            if (request.BeginDateTime.HasValue && request.EndDateTime.HasValue)
            {
                var month = request.EndDateTime.Value.DifferMonth(request.BeginDateTime.Value);

                if(month <= 0 )
                    dateTime = request.BeginDateTime.Value.ToString("yyyy-MM");
                else
                    dateTime = "*";
            }
            else
                dateTime = "*";

            var mustQuerys = new List<Func<QueryContainerDescriptor<UserViewDuration>, QueryContainer>>();

            if (request.UserId.HasValue)
                mustQuerys.Add(a => a.Term(t => t.Field(f => f.UserId).Value(request.UserId.Value)));

            if (request.EntityType.HasValue)
                mustQuerys.Add(a => a.Term(t => t.Field(f => f.EntityType).Value(request.EntityType)));

            if (request.EntityId.HasValue)
                mustQuerys.Add(a => a.Term(t => t.Field(f => f.EntityId).Value(request.EntityId.Value)));

            if (request.CharpterId.HasValue)
                mustQuerys.Add(a => a.Term(t => t.Field(f => f.CharpterId).Value(request.CharpterId.Value)));

            if (request.BeginDateTime.HasValue)
                mustQuerys.Add(a => a.DateRange(dr =>
                    dr.Field(f => f.CreateDateTime).GreaterThanOrEquals(request.BeginDateTime.Value).TimeZone(EsConst.TimeZone)));

            if (request.EndDateTime.HasValue)
                mustQuerys.Add(a =>
                    a.DateRange(dr => dr.Field(f => f.CreateDateTime).LessThanOrEquals(request.EndDateTime.Value).TimeZone(EsConst.TimeZone)));

            var searchResult = _elasticClient.Search<UserViewDuration>(a =>
                a.Index(typeof(UserViewDuration).GetRelationName() + "-" + dateTime)
                    .Size(request.Size)
                    .Query(q => q.Bool(b => b.Must(mustQuerys)))
                    .SearchAfterTimestamp(request.Timestamp)
                    .Sort(s => s.Field(f => f.Timestamp, SortOrder.Descending)));

            var apiResult = searchResult.GetApiResult<UserViewDuration, List<UserMarkRecordGetRecordResponse>>();
            if (apiResult.Success)
                dataList.AddRange(apiResult.Data);

            return ApiResult<List<UserMarkRecordGetRecordResponse>>.IsSuccess(dataList);
        }
```

**作品搜索实现细节**

**实体定义**  

SearchKey 是原有 SQL Server 的数据，现需要同步到 Elasticsearch，仍是继承抽象类 ElasticsearchEntity 实体定义，同时这里有三个细节点：

1. public string KeyName，我定义的是**Text 类型**，在 Elasticsearch 使用 Text 类型才会分词。

2\. 在实体定义我没有给 KeyName 指定分词器，因为我会使用两个分词器：拼音和默认分词，而我会在批量写入数据创建 Mapping 时定义。

3\. 实体里的**public List<int> SysTagId** 与**SearchKey**在 SQL Server 是两张不同的物理表，是一对多的关系，在代码表示如下，

但是在关系型数据库是无法与之对应和体现的，这就是咱们所说的 “**阻抗失配**”，但是能在以文档型存储系统（MongoDB、Elasticsearch）里很好的解决这个问题，可以以一个**聚合**的方式写入，避免多次查询关联。

```cs
 [ElasticsearchType(RelationName = "search_key")]
    public class SearchKey : ElasticsearchEntity
    {
        [Number(NumberType.Integer, Name = "key_id")]
        public int KeyId { get; set; }

        [Number(NumberType.Integer, Name = "entity_id")]
        public int EntityId { get; set; }

        [Number(NumberType.Integer, Name = "entity_type")]
        public int EntityType { get; set; }

        [Text(Name = "key_name")]
        public string KeyName { get; set; }

        [Number(NumberType.Integer, Name = "weight")]
        public int Weight { get; set; }

        [Boolean(Name = "is_subsidiary")]
        public bool IsSubsidiary { get; set; }

        [Date(Name = "active_date")]
        public DateTimeOffset? ActiveDate { get; set; }

        [Number(NumberType.Integer, Name = "sys_tag_id")]
        public List<int> SysTagId { get; set; }
    }
```

**数据同步**

数据同步我采用了 Quartz.Net 定时调度任务框架，因此时效不高，所以每 4 小时同步一次即可，有 42W 多的数据，分批进行同步，每次查询 1000 条数据同时进行一次批量写入。全量同步一次的时间大概 2 分钟。因此使用 RPC 调用\[ES 业务 API 服务]。

因为具体业务逻辑已经封装在\[ES 业务 API 服务]，因此同步逻辑也相对简单，查询出 SQL Server 数据源、聚合整理、调用\[ES 业务 API 服务]的批量写入接口、重新绑定别名到新的 Index。

```typescript
　[DisallowConcurrentExecution]
    public class SearchKeySynchronousJob : BaseJob
    {
        public override void Execute()
        {
            var rm = SFNovelReadManager.Instance();

            var maxId = 0;
            var size = 1000;
            string indexName = "";

            while (true)
            {
                
                var searchKey = sm.searchKey.GetList(size, maxId);

                if (!searchKey.Any())
                    break;

                var entityIds = searchKey.Select(a => a.EntityID).Distinct().ToList();

                var sysTagRecord = rm.Novel.GetSysTagRecord(entityIds);

                var items = searchKey.Select(a => new SearchKeyPostItem
                {
                    Weight = a.Weight,
                    EntityType = a.EntityType,
                    EntityId = a.EntityID,
                    IsSubsidiary = a.IsSubsidiary ?? false,
                    KeyName = a.KeyName,
                    ActiveDate = a.ActiveDate,
                    SysTagId = sysTagRecord.Where(c => c.EntityID == a.EntityID).Select(c => c.SysTagID).ToList(),
                    KeyID = a.KeyID
                }).ToList();

                
                var postResult = new SearchKeyPostRequest
                {
                    IndexName = indexName,
                    Items = items
                }.Excute();

                if (postResult.Success)
                {
                    indexName = (string)postResult.Data;
                    maxId = searchKey.Max(a => a.KeyID);
                }
            }

            
            var renameResult = new SearchKeyRenameRequest
            {
                IndexName = indexName
            }.Excute();
        }
    }
}
```

**业务 API 接口**

**批量新增接口**这里有 2 个细节点：

1\. 在第一次有数据进来的时候需要创建 Mapping，因为得对 KeyName 字段定义分词器，其余字段都可以使用 AutoMap 即可。

2\. 新创建的 Index 名称是精确到秒的 **SearchKey-202112261121**

```cs
　 
        
        
        
        
        [HttpPost]
        public ApiResult Post(SearchKeyPostRequest request)
        {
            if (!request.Items.Any())
                return ApiResult.IsFailed("无传入数据");

            var date = DateTime.Now;
            var relationName = typeof(SearchKey).GetRelationName();
            var indexName = request.IndexName.IsNullOrWhiteSpace() ? (relationName + "-" + date.ToString("yyyyMMddHHmmss")) : request.IndexName;

            if (request.IndexName.IsNullOrWhiteSpace())
            {
                var createResult = _elasticClient.Indices.Create(indexName,
                    a =>
                        a.Map<SearchKey>(m => m.AutoMap().Properties(p =>
                            p.Custom(new TextProperty
                            {
                                Name = "key_name",
                                Analyzer = "standard",
                                Fields = new Properties(new Dictionary<PropertyName, IProperty>
                                {
                                    { new PropertyName("pinyin"),new TextProperty{ Analyzer = "pinyin"} },
                                    { new PropertyName("standard"),new TextProperty{ Analyzer = "standard"} }
                                })
                            }))));

                if (!createResult.IsValid && request.IndexName.IsNullOrWhiteSpace())
                    return ApiResult.IsFailed("创建索引失败");
            }
            
            var document = request.Items.MapTo<List<SearchKey>>();

            var result = _elasticClient.BulkAll(indexName, document);

            return result ? ApiResult.IsSuccess(data: indexName) : ApiResult.IsFailed();
        }
```

**重新绑定别名接口**这里有 4 个细节点：

1\. 别名使用 searchkey，只会有一个**Index\[searchkey-yyyyMMddHHmmss]**会跟 searchkey 绑定.

2\. 优先把已绑定的 Index 查询出来，方便解绑与删除。

3\. 别名绑定在 Elasticsearch 虽然是原子性的，但是不是数据一致性的，因此得先 Add 后 Remove。

4\. 删除旧得 Index 免得占用过多资源。

```cs
 
        
        
        
        [HttpPut]
        public ApiResult Rename(SearchKeyRanameRequest request)
        {
            var aliasName = typeof(SearchKey).GetRelationName();
            var getAliasResult = _elasticClient.Indices.GetAlias(aliasName);

            
            var bulkAliasRequest = new BulkAliasRequest
            {
                Actions = new List<IAliasAction>
                {
                    new AliasAddDescriptor().Index(request.IndexName).Alias(aliasName)
                }
            };

            
            if (getAliasResult.IsValid)
            {
                var indeNameList = getAliasResult.Indices.Keys;
                foreach (var indexName in indeNameList)
                {
                    bulkAliasRequest.Actions.Add(new AliasRemoveDescriptor().Index(indexName.Name).Alias(aliasName));
                }
            }

            var result = _elasticClient.Indices.BulkAlias(bulkAliasRequest);

            
            if (getAliasResult.IsValid)
            {
                var indeNameList = getAliasResult.Indices.Keys;
                foreach (var indexName in indeNameList)
                {
                    _elasticClient.Indices.Delete(indexName);
                }
            }

            return result != null && result.ApiCall.Success ? ApiResult.IsSuccess() : ApiResult.IsFailed();
        }
```

**查询接口**这里跟前面细节得差不多：

但是这里有一个得**特别注意**的点，可以看到这个查询接口同时使用了 should 和 must，这里得设置 minimumShouldMatch 才能正常像 SQL 过滤。

should 可以理解成 SQL 的 Or，Must 可以理解成 SQL 的 And。

默认情况下 minimumShouldMatch 是等于 0 的，等于 0 的意思是，**should 不命中任何的数据仍然会返回 must 命中的数据**，

也就是你们可能想搜索**（keyname.pinyin=’chengong‘ or keyname.standard=’chengong‘) and id > 0**，但是 es 里没有存 keyname='chengong'的数据，会把 id> 0 而且 keyname != 'chengong' 数据给查询出来。

因此我们得对 minimumShouldMatch=1，就是**should**条件**必须得任意命中一个**才能返回结果。

**在 should 和 must 混用的情况下必须得注意 minimumShouldMatch 的设置！！！**

```typescript

        
        
        
        
        [HttpPost]
        [Route("search")]
        public ApiResult<List<SearchKeyGetResponse>> Get(SearchKeyGetRequest request)
        {
            var shouldQuerys = new List<Func<QueryContainerDescriptor<SearchKey>, QueryContainer>>();
            int minimumShouldMatch = 0;
            if (!request.KeyName.IsNullOrWhiteSpace())
            {
                shouldQuerys.Add(a => a.MatchPhrase(m => m.Field("key_name.pinyin").Query(request.KeyName)));
                shouldQuerys.Add(a => a.MatchPhrase(m => m.Field("key_name.standard").Query(request.KeyName)));
                minimumShouldMatch = 1;
            }

            var mustQuerys = new List<Func<QueryContainerDescriptor<SearchKey>, QueryContainer>>
            {
                a => a.Range(t => t.Field(f => f.Weight).GreaterThanOrEquals(0))
            };

            if (request.IsSubsidiary.HasValue)
                mustQuerys.Add(a => a.Term(t => t.Field(f => f.IsSubsidiary).Value(request.IsSubsidiary.Value)));

            if (request.SysTagIds != null && request.SysTagIds.Any())
                mustQuerys.Add(a => a.Terms(t => t.Field(f => f.SysTagId).Terms(request.SysTagIds)));

            if (request.EntityType.HasValue)
            {
                if (request.EntityType.Value == ESearchKey.EntityType.AllNovel)
                {
                    mustQuerys.Add(a => a.Terms(t => t.Field(f => f.EntityType).Terms(ESearchKey.EntityType.Novel, ESearchKey.EntityType.ChatNovel, ESearchKey.EntityType.FanNovel)));
                }
                else
                    mustQuerys.Add(a => a.Term(t => t.Field(f => f.EntityType).Value((int)request.EntityType.Value)));
            }

            var sortDescriptor = new SortDescriptor<SearchKey>();
            sortDescriptor = request.Sort == ESearchKey.Sort.Weight
                ? sortDescriptor.Field(f => f.Weight, SortOrder.Descending)
                : sortDescriptor.Field(f => f.ActiveDate, SortOrder.Descending);

            var searchResult = _elasticClient.Search<SearchKey>(a =>
                a.Index(typeof(SearchKey).GetRelationName())
                    .From(request.Size * request.Page)
                    .Size(request.Size)
                    .Query(q => q.Bool(b => b.Should(shouldQuerys).Must(mustQuerys).MinimumShouldMatch(minimumShouldMatch)))
                    .Sort(s => sortDescriptor));

            var apiResult = searchResult.GetApiResult<SearchKey, List<SearchKeyGetResponse>>();

            if (apiResult.Success)
                return apiResult;

            return ApiResult<List<SearchKeyGetResponse>>.IsSuccess("空集合数据");
```

**APM 监控**

虽然在上面我做了足够的实现准备，但是对于上生产后的实际使用效果我还是希望有一个直观的体现。我之前写了一篇文章**《.Net 微服务实战之可观测性》**很好叙述了该种情况，有兴趣的可以移步去看看。

在之前公司做微服务的时候的 APM 选型我们使用了 Skywalking，但是现在这家公司的运维没有接触过，但是对于 Elastic Stack 他相对比较熟悉，如同上文所说**架构设计的输入核心为两点：满足需求与组织架构，**秉着我的技术选型原则是基于团队架构，我们采用了 Elastic APM + Kibana（7.4 版本），如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYm7keCBhbjmfib7STZ2PTB1qS5jUhd011XDWxgUCZTEhIyVFewyYd3A79SzRvb6htL4ZCYBGph3QQ/640?wx_fmt=png)

**结 尾**

如喜欢本文，请点击右上角，把文章分享到朋友圈  
如有想了解学习的技术点，请留言给若飞安排分享

**·END·**

**相关阅读：** 

* * *

-   [一张图看懂微服务架构路线](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650357440&idx=1&sn=c4b353d9143ab7f7ef74b122dc2f383f&chksm=83004a22b477c334645c79a76659d71586c7237cdcc13c7586a4fa7409697e7e7392d1fd425b&scene=21#wechat_redirect)  
-   [基于 Spring Cloud 的微服务架构分析](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650353906&idx=1&sn=5cb209febf8867841d31ecb2b32a059d&chksm=83004410b477cd066f62fbdc032eec5e2057d74894dddbc5994c120393b5e6a2c104283048b5&scene=21#wechat_redirect)
-   [微服务等于 Spring Cloud？了解微服务架构和框架](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650352201&idx=1&sn=ad5cbac0687d5dc3c9c314aa959c07fe&chksm=83005eabb477d7bd445342f43baaa61fa2ca5dfae7c75ef6fb3b650569e8bcefba86318aa013&scene=21#wechat_redirect)  
-   [如何构建基于 DDD 领域驱动的微服务？](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650353129&idx=1&sn=73c8cb5919e18d32c21b8089646256be&chksm=8300590bb477d01dbad276ea44d04695e63b3f604d9cf0204da7e2653d37f07b9634c4368473&scene=21#wechat_redirect)
-   [小团队真的适合引入 SpringCloud 微服务吗？](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650352055&idx=1&sn=f9cbd707459421370aac6ac71f016052&chksm=83005d55b477d443649c5ffeb79fd371abe92a80375dc55063dcf5a2879a471bdfca84a4f37b&scene=21#wechat_redirect)  
-   [DDD 兴起的原因以及与微服务的关系](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650351363&idx=1&sn=de8f029590d26376f5dac6ededded2fb&chksm=830053e1b477daf7adc3a72ffd2f0f0f0c6a22030bace8cdee1fba6a2dc14e9cb5699d08741c&scene=21#wechat_redirect)
-   [微服务之间的最佳调用方式](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650351218&idx=1&sn=88c6116fa0105b90addcf4d460b99fbe&chksm=83005290b477db86bd1b6a7cb96de4f397955b1887f5ef9eee6d5393a1d6b457f8d68a64fa38&scene=21#wechat_redirect)  
-   [微服务架构设计总结实践](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650350733&idx=1&sn=fec8de69cdb8df73c79b22cbdd7c0775&chksm=8300506fb477d9794e3aa90a2b19621523146c3358126d25a22df0ab386f193f9fd87292ba8a&scene=21#wechat_redirect)  
-   [基于 Kubernetes 的微服务项目设计与实现](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650350395&idx=1&sn=90722cb88bc6cdfafdcdecaf9edd5f29&chksm=830057d9b477decf84744fff10e352de3e719f9b8fce8dc40a9b4c6a4a9f9b44cb2f1eddae4c&scene=21#wechat_redirect)
-   [微服务架构 - 设计总结](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650348667&idx=1&sn=fa8aec8cd09a448235ba0b21e27d99b4&chksm=83006899b477e18fdb56322a98f156aaa0745b35137e3c0c6bbe0a940f8a7a24a4eaf3318625&scene=21#wechat_redirect)
-   [为什么微服务一定要有网关？](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650348663&idx=1&sn=f476299d36695fead2eff2d7a29bad16&chksm=83006895b477e1831c66d7709b52a74246d97986157587a14785a51b4e23248fc1a327841a47&scene=21#wechat_redirect)
-   [主流微服务全链路监控系统之战](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650348314&idx=1&sn=84c54dd629f219ca80a157939e5c546f&chksm=83006ff8b477e6ee2c006d3f3b393c2787b110d8123927e5c3a7639bc4515dc09ccdc4eb8845&scene=21#wechat_redirect)  
-   [微服务架构实施原理详解](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650346739&idx=1&sn=f9855c471d182e468d823cab80ce53f1&chksm=83006011b477e907f1641eb217f797afb1a088b8c826c152d6dd658649a4e0ec570da7396be8&scene=21#wechat_redirect)  
-   [微服务的简介和技术栈](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650346693&idx=1&sn=cbb4a27496a01aaf19c651484116d55e&chksm=83006027b477e93118c61fb80b0d55dc5ff68c92e8aab67708e0ca2e9781b7eb2dc3692cd0cf&scene=21#wechat_redirect)
-   [微服务场景下的数据一致性解决方案](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650346733&idx=1&sn=7df8de06a8c29a3712c30903f4464353&chksm=8300600fb477e91956c891af0f8a6a342705332e64f7f81b59678cc729e1a4c85c919904c421&scene=21#wechat_redirect)
-   [设计一个容错的微服务架构](http://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=2650345318&idx=1&sn=beb39fcca1b8af9c8dea348398326723&chksm=83007b84b477f292c01238686e50caf4e43304a3c748f3419ed5b876197f1c4995f2a9e4bfd0&scene=21#wechat_redirect)

> 作者：陈珙
>
> 来源：www.cnblogs.com/skychen1218/p/15720522.html
>
> 版权申明：内容来源网络，仅供分享学习，版权归原创者所有。除非无法确认，我们都会标明作者及出处，如有侵权烦请告知，我们会立即删除并表示歉意。谢谢!

**架构师**

我们都是架构师！

![](https://mmbiz.qpic.cn/mmbiz/sXiaukvjR0RB58TtkIHwhn4lpsqLnZgian9d5tr1BibP7XpibGTFFib1nq9YuYq209XZUEfCOqMzepDOBbN9KD9wMSg/640?wx_fmt=jpeg)

\***\* 关注**架构师 (JiaGouX)，添加 “星标”\*\*

**获取每天技术干货，一起成为牛逼架构师**  

**技术群请 \*\***加若飞：\*\* **1321113940\*\***进架构师群 \*\*

投稿、合作、版权等邮箱：**admin@137x.com** 
 [https://mp.weixin.qq.com/s/FluPfO_D18nIU2P0VRJvqw](https://mp.weixin.qq.com/s/FluPfO_D18nIU2P0VRJvqw)
