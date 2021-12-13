# Presto原理&调优&面试&实战全面升级版
原创 大数据技术与架构 大数据技术与架构 _2021-06-17 08:20_

点击上方**蓝色字体**，选择 “**设为星标**”

回复” 资源 “获取更多资源

![](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg)

很久之前，曾经写过一篇 《[Presto 在大数据领域的实践和探索](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247497505&idx=1&sn=6ae3547d253cf76afe4c813d850e999d&chksm=fd3eb1b4ca4938a253b7f0d54d35eaf110890f24d846a5c9ee85e48a2884bb4ad526ea0eccb1&scene=21#wechat_redirect)》 。文中详细讲解了 Presto 的原理和应用。

今天这篇文章是升级版本，把我个人读过的文章和书籍的笔记进行了系统整理。**从起源、原理、调优、面试、实践应用进行了全方位的升级**。希望对你们有帮助。

## 一、起源

Presto 是由 FaceBook 开源的一个 MPP 计算引擎，主要用来以解决 Facebook 海量 Hadoop 数据仓库的低延迟交互分析问题，Facebook 版本的 Presto 更多的是以解决企业内部需求功能为主，也叫 PrestoDB，版本号以 0.xxx 来划分。

后来，Presto 其中的几个人出来创建了更通用的 Presto 分支，取名 Presto SQL，版本号以 xxx 来划分，例如 345 版本，这个开源版本也是更为被大家通用的版本。前一段时间，为了更好的与 Facebook 的 Presto 进行区分，Presto SQL 将名字改为 Trino，除了名字改变了其他都没变。不管是 Presto DB 还是 Presto SQL，它们” 本是同根生 “，因此它们的大部分的机制原理是一样的。

### 我是谁？我从哪里来？要到哪里去？

    Presto is an open source distributed SQL query engine for running interactive analytic queries against data sources of all sizes ranging from gigabytes to petabytes.Presto allows querying data where it lives, including Hive, Cassandra, relational databases or even proprietary data stores. A single Presto query can combine data from multiple sources, allowing for analytics across your entire organization.Presto is targeted at analysts who expect response times ranging from sub-second to minutes. Presto breaks the false choice between having fast analytics using an expensive commercial solution or using a slow "free" solution that requires excessive hardware.

这是官网对 Presto 的定义，Presto 是由 Facebook 开源的大数据分布式 SQL 查询引擎，适用于交互式分析查询，可支持众多的数据源，包括 HDFS，RDBMS，KAFKA 等，而且提供了非常友好的接口开发数据源连接器。

## 二、特点及场景介绍

**1. 特点**

Presto 引擎相较于其他引擎的特点正如文章标题描述的这样，多源、即席。多源就是它可以支持跨不同数据源的联邦查询，即席即实时计算，将要做的查询任务实时拉取到本地进行现场计算，然后返回计算结果。除此之外，对于引擎本身，它有几个值得关注的特点：

（1）多租户：它支持并发执行数百个内存、I/O 以及 CPU 密集型的负载查询，并支持集群规模扩展到上千个节点；  
（2）联邦查询：它可以由开发者利用开放接口自定义开发针对不同数据源的连接器（Connector), 从而支持跨多种不同数据源的联邦数据查询；  
（3）内在特性：为了保证引擎的高效性，Presto 还进行了一些优化，例如基于 JVM 运行，Code- Generation 等。

**2. 场景**

Presto 的应用场景非常广泛，接下来我们主要介绍几种使用比较广泛的场景进行介绍。

（1）交互式分析：交互式查询是 Presto 主打的应用场景，Presto 的即席计算特性和内部设计机制就是为了能够更好地支持用户进行交互式分析。可以类比用户基于 Hive 交互式查询 HDFS 中的数据，用户可以基于 Presto 查询各种不同的数据源的数据。  
（2）批量 ETL。  
（3）Facebook 的 A/B Test 基础架构也是基于 Presto 构建的。

Presto 之所以能在各个内存计算型数据库中脱颖而出，在于以下几点：

-   清晰的架构，是一个能够独立运行的系统，不依赖于任何其他外部系统。例如调度，presto 自身提供了对集群的监控，可以根据监控信息完成调度。
-   简单的数据结构，列式存储，逻辑行，大部分数据都可以轻易的转化成 presto 所需要的这种数据结构。
-   丰富的插件接口，完美对接外部存储系统，或者添加自定义的函数。

## 三、整体架构

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnzt52wHb1uqlWN2zWPvcDmNibY3Tv2vgmLOpq9Ae4WpTiav3gN8hJ2icwicmg/640?wx_fmt=png)
图 1 Presto 架构图

Presto 主要是由 Client、Coordinator、Worker 以及 Connector 等几部分构成。

**1.SQL 语句提交：** 

用户或应用通过 Presto 的 JDBC 接口或者 CLI 来提交 SQL 查询，提交的 SQL 最终传递给 Coordinator 进行下一步处理；

**2. 词 / 语法分析：** 

首先会对接收到的查询语句进行词法分析和语法分析，形成一棵抽象语法树。  
然后，会通过分析抽象语法树来形成逻辑查询计划。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnzt1U1knon0OmQ9PtgBBsvibGNzaFCcb1Vc2icEoMus9qx2Muvodp3QDfSA/640?wx_fmt=png)
图 2 查询 SQL

**3. 生成逻辑计划：** 

图 2 是 TPC-H 测试基准中的一条 SQL 语句，表达的是两表连接同时带有分组聚合计算的例子，经过词法语法分析后，得到 AST，然后进一步分析得到如下的逻辑计划。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztkpJaOO9ppc3sd4h9Ik7lAANosgCSKOREtUwDpaRKleZ67OWA0cgBkg/640?wx_fmt=png)
图 3 逻辑计划

上图就是一棵逻辑计划树，每个节点代表一个物理或逻辑操作，每个节点的子节点作为该节点的输入。逻辑计划只是一个单纯描述 SQL 的执行逻辑，但是并不包括具体的执行信息，例如该操作是在单节点上执行还是可以在多节点并行执行，再例如什么时候需要进行数据的 shuffle 操作等。

**4. 查询优化：** 

Coordinator 将一系列的优化策略（例如剪枝操作、谓词下推、条件下推等）应用于与逻辑计划的各个子计划，从而将逻辑计划转换成更加适合物理执行的结构，形成更加高效的执行策略。

下面具体来说说优化器在几个方面所做的工作：

（1）自适应：Presto 的 Connector 可以通过 Data Layout API 提供数据的物理分布信息（例如数据的位置、分区、排序、分组以及索引等属性），如果一个表有多种不同的数据存储分布方式，Connector 也可以将所有的数据布局全部返回，这样 Presto 优化器就可以根据 query 的特点来选择最高效的数据分布来读取数据并进行处理。

（2）谓词下推：谓词下推是一个应用非常普遍的优化方式，就是将一些条件或者列尽可能的下推到叶子结点，最终将这些交给数据源去执行，从而可以大大减少计算引擎和数据源之间的 I/O，提高效率。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztL5Qiaia8NyZCVPb4yUnslzsyqm0icv998hj2xdYPKkt9DJb9qkaO2SoMQ/640?wx_fmt=png)
图 4 图 3 的逻辑计划进一步转换后的执行计划（未进行）

（3）节点间并行：不同 stage 之间的数据 shuffle 会带来很大的内存和 CPU 开销，因此，将 shuffle 数优化到最小是一个非常重要的目标。围绕这个目标，Presto 可以借助一下两类信息：

-   数据布局信息：上面我们提到的数据物理分布信息同样可以用在这里以减少 shuffle 数。例如，如果进行 join 连接的两个表的字段同属于分区字段，则可以将连接操作在在各个节点分别进行，从而可以大大减少数据的 shuffle。
-   再比如两个表的连接键加了索引，可以考虑采用嵌套循环的连接策略。

（4）节点内并行：优化器通过在节点内部使用多线程的方式来提高节点内对并行度，延迟更小且会比节点间并行效率更高。

-   交互式分析：交互式查询的负载大部分是一次执行的短查询，查询负载一般不会经过优化，这就会导致数据倾斜的现象时有发生。典型的表现为少量的节点被分到了大量的数据。
-   批量 ETL：这类的查询特点是任务会不加过滤的从叶子结点拉取大量的数据到上层节点进行转换操作，致使上层节点压力非常大。

针对以上两种场景遇到的问题，引擎可以通过多线程来运行单个操作符序列（或 pipeline），如图 5 所示的，pipeline1 和 2 通过多线程并行执行来加速 build 端的 hash-join。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztQwKgVplPPfs2dpRBHYvOWdlh8gHKfHRWwrzw6eAz5vPwVc5hy7b1ng/640?wx_fmt=png)
图 5 pipeline1 和 2 通过多线程并行执行来加速 build 端的 hash-join

当然，除了上述列举的 Presto 优化器已经实现的优化策略，Presto 也正在积极探索 Cascades framework，相信未来优化器会得到进一步的改进。

**5. 容错**

Presto 可以对一些临时的报错采用低级别的重试来恢复。Presto 依靠的是客户端的自动重跑失败查询。内嵌容错机制来解决 coordinator 或者 worker 节点坏掉的情况目前 Presto 支持的并不理想。

标准检查点或者部分修复技术是计算代价比较高的，而且很难在这种一旦结果可用就返回给客户端（即时查询类）的系统中实现。

### 四、资源和调度

我们借用美团的博客中的一张架构图：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztW0nyEVCb44HwMeIPabBTwhqBcP0KPYwyaibk6YdRmicIOZNGyjv5tjDA/640?wx_fmt=png)

Presto 查询引擎是一个 Master-Slave 的架构，由一个 Coordinator 节点，一个 Discovery Server 节点，多个 Worker 节点组成，Discovery Server 通常内嵌于 Coordinator 节点中。

Coordinator 负责解析 SQL 语句，生成执行计划，分发执行任务给 Worker 节点执行。

Worker 节点负责实际执行查询任务。Worker 节点启动后向 Discovery Server 服务注册，Coordinator 从 Discovery Server 获得可以正常工作的 Worker 节点。如果配置了 Hive Connector，需要配置一个 Hive MetaStore 服务为 Presto 提供 Hive 元信息，Worker 节点与 HDFS 交互读取数据。

**Presto 的服务进程**

Presto 集群中有两种进程，Coordinator 服务进程和 worker 服务进程。coordinator 主要作用是接收查询请求，解析查询语句，生成查询执行计划，任务调度和 worker 管理。worker 服务进程执行被分解的查询执行任务 task。

Coordinator 服务进程部署在集群中的单独节点之中，是整个 presto 集群的管理节点，主要作用是接收查询请求，解析查询语句，生成查询执行计划 Stage 和 Task 并对生成的 Task 进行任务调度，和 worker 管理。Coordinator 进程是整个 Presto 集群的 master 进程，需要与 worker 进行通信，获取最新的 worker 信息，有需要和 client 通信，接收查询请求。Coordinator 提供 REST 服务来完成这些工作。

Presto 集群中存在一个 Coordinator 和多个 Worker 节点，每个 Worker 节点上都会存在一个 worker 服务进程，主要进行数据的处理以及 Task 的执行。worker 服务进程每隔一定的时间会发送心跳包给 Coordinator。Coordinator 接收到查询请求后会从当前存活的 worker 中选择合适的节点运行 task。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztV4HxWKZosgDmI6X6Z1XicTxVjuBdyzdicCmfFnicW1yo9LymvjU02saPw/640?wx_fmt=png)

上图展示了从宏观层面概括了 Presto 的集群组件：1 个 coordinator，多个 worker 节点。用户通过客户端连接到 coordinator，可以短可以是 JDBC 驱动或者 Presto 命令行 cli。

Presto 是一个分布式的 SQL 查询引擎，组装了多个并行计算的数据库和查询引擎（这就是 MPP 模型的定义）。Presto 不是依赖单机环境的垂直扩展性。她有能力在水平方向，把所有的处理分布到集群内的各个机器上。这意味着你可以通过添加更多节点来获得更大的处理能力。

利用这种架构，Presto 查询引擎能够并行的在集群的各个机器上，处理大规模数据的 SQL 查询。Presto 在每个节点上都是单进程的服务。多个节点都运行 Presto，相互之间通过配置相互协作，组成了一个完整的 Presto 集群。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnzt0l3hHYAJBoiazDFYgicgfaZhqXicJOaP4tdf7PVibm4QpASAKOgDHGJUUw/640?wx_fmt=png)

上图展示了集群内 coordinator 和 worker 之间，以及 worker 和 worker 之间的通信。coordinator 向多个 worker 通信，用于分配任务，更新状态，获得最终的结果返回用户。worker 之间相互通信，向任务的上游节点获取数据。所有的 worker 都会向数据源读取数据。

**Coordinator**

Coordinator 的作用是：

-   从用户获得 SQL 语句
-   解析 SQL 语句
-   规划查询的执行计划
-   管理 worker 节点状态

Coordinator 是 Presto 集群的大脑，并且是负责和客户端沟通。用户通过 PrestoCLI、JDBC、ODBC 驱动、其他语言工具库等工具和 coordinator 进行交互。Coordinator 从客户端接受 SQL 语句，例如 select 语句，才能进行计算。

每个 Presto 集群必须有一个 coordinator，可以有一个或多个 worker。在开发和测试环境中，一个 Presto 进程可以同时配置成两种角色。

Coordinator 追踪每个 worker 上的活动，并且协调查询的执行过程。

Coordinator 给查询创建了一个包含多阶段的逻辑模型，一旦接受了 SQL 语句，Coordinator 就负责解析、分析、规划、调度查询在多个 worker 节点上的执行过程，语句被翻译成一系列的任务，跑在多个 worker 节点上。

worker 一边处理数据，结果会被 coordinator 拿走并且放到 output 缓存区上，暴露给客户端。

一旦输出缓冲区被客户完全读取，coordinator 会代表客户端向 worker 读取更多数据。

worker 节点，和数据源打交道，从数据源获取数据。因此，客户端源源不断的读取数据，数据源源源不断的提供数据，直到查询执行结束。

Coordinator 通过基于 HTTP 的协议和 worker、客户端之间进行通信。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztteJCczptWcZxkE4LGib3hR4avw3FR90TOcvYPl03day5JKL7nR9EFRQ/640?wx_fmt=png)

上图给我们展示了客户端、coordinator，worker 之间的通信。

**Workers**

Presto 的 worker 是 Presto 集群中的一个服务。它负责运行 coordinator 指派给它的任务，并处理数据。worker 节点通过连接器（connector）向数据源获取数据，并且相互之间可以交换数据。最终结果会传递给 coordinator。coordinator 负责从 worker 获取最终结果，并传递给客户端。

Worker 之间的通信、worker 和 coordinator 之间的通信采用基于 HTTP 的协议。下图展示了多个 worker 如何从数据源获取数据，并且合作处理数据的流程。直到某一个 worker 把数据提供给了 coordinator。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnzts0C8j0Fbayhf2xYvuGbsxLcIuIq0yG4LOEADF53MOH6YDz4TpUrXvw/640?wx_fmt=png)

**查询调度：** 

Presto 通过 Coordinator 将 stage 以 task 的形式分发到 worker 节点，coordinator 将 task 以 stage 为单位进行串联，通过将不同 stage 按照先后执行顺序串联成一棵执行树，确保数据流能够顺着 stage 进行流动。

Presto 引擎处理一条查询需要进行两套调度，第一套是如何调度 stage 的执行顺序，第二套是判断每个 stage 有多少需要调度的 task 以及每个 task 应该分发到哪个 worker 节点上进行处理。

（1）stage 调度

Presto 支持两种 stage 调度策略：All-at-once 和 Phased 两种。All-at- once 策略针对所有的 stage 进行统一调度，不管 stage 之间的数据流顺序，只要该 stage 里的 task 数据准备好了就可以进行处理；Phased 策略是需要以 stage 调度的有向图为依据按序执行，只要前序任务执行完毕开会开始后续任务的调度执行。例如一个 hash-join 操作，在 hash 表没有准备好之前，Presto 不会调度 left side 表。

（2）task 调度

在进行 task 调度的时候，调度器会首先区分 task 所在的 stage 是哪一类 stage：Leaf Stage 和 intermediate stage。Leaf Stage 负责通过 Connector 从数据源读取数据，intermediate stage 负责处理来此其他上游 stage 的中间结果；

-   leaf stages：在分发 leaf stages 中的 task 到 worker 节点的时候需要考虑网络和 connector 的限制。例如蚕蛹 shared- nothing 部署的时候，worker 节点和存储是同地协作，这时候调度器就可以根据 connector data Layout API 来决定将 task 分发到哪些 worker 节点。资料表明在一个生产集群大部分的 CPU 消耗都是花费在了对从 connector 读取到的数据的解压缩、编码、过滤以及转换等操作上，因此对于此类操作，要尽可能的提高并行度，调动所有的 worker 节点来并行处理。
-   intermediate stages：这里的 task 原则上可以被分发到任意的 worker 节点，但是 Presto 引擎仍然需要考虑每个 stage 的 task 数量，这也会取决于一些相关配置，当然，有时候引擎也可以在运行的时候动态改变 task 数。

（3）split 调度

当 Leaf stage 中的一个 task 在一个工作节点开始执行的时候，它会收到一个或多个 split 分片，不同 connector 的 split 分片所包含的信息也不一样，最简单的比如一个分片会包含该分片 IP 以及该分片相对于整个文件的偏移量。对于 Redis 这类的键值数据库，一个分片可能包含表信息、键值格式以及要查询的主机列表。Leaf stage 中的 task 必须分配一个或多个 split 才能够运行，而 intermediate stage 中的 task 则不需要。

（3）split 分配  
当 task 任务分配到各个工作节点后，coordinator 就开始给每个 task 分配 split 了。Presto 引擎要求 Connector 将小批量的 split 以懒加载的方式分配给 task。这是一个非常好的特点，会有如下几个方面的优点：

-   解耦时间：将前期的 split 准备工作与实际的查询执行时间分开；
-   减少不必要的数据加载：有时候一个查询可能刚出结果但是还没完全查询完就被取消了，或者会通过一些 limit 条件限制查询到部分数据就结束了，这样的懒加载方式可以很好的避免过多加载数据；
-   维护 split 队列：工作节点会为分配到工作进程的 split 维护一个队列，Coordinator 会将新的 split 分配给具有最短队列的 task，Coordinator 分给最短的。
-   减少元数据维护：这种方式可以避免在查询的时候将所有元数据都维护在内存中，例如对于 Hive Connector 来讲，处理 Hive 查询的时候可能会产生百万级的 split，这样就很容易把 Coordinator 的内存给打满。当然，这种方式也不是没有缺点，他的缺点是可能会导致难以准确估计和报告查询进度。

**资源管理**

Presto 适用于多租户部署的一个很重要的因素就是它完全整合了细粒度资源管理系统。一个单集群可以并发执行上百条查询以及最大化的利用 CPU、IO 和内存资源。

（1）CPU 调度

Presto 首要任务是优化所有集群的吞吐量，例如在处理数据是的 CPU 总利用量。本地（节点级别）调度又为低成本的计算任务的周转时间优化到更低，以及对于具有相似 CPU 需求的任务采取 CPU 公平调度策略。一个 task 的资源使用是这个线程下所有 split 的执行时间的累计，为了最小化协调时间，Presto 的 CPU 使用最小单位为 task 级别并且进行节点本地调度。

Presto 通过在每个节点并发调度任务来实现多租户，并且使用合作的多任务模型。任何一个 split 任务在一个运行线程中只能占中最大 1 秒钟时长，超时之后就要放弃该线程重新回到队列。如果该任务的缓冲区满了或者 OOM 了，即使还没有到达占用时间也会被切换至另一个任务，从而最大化 CPU 资源的利用。

当一个 split 离开了运行线程，Presto 需要去定哪一个 task（包含一个或多个 split）排在下一位运行。

Presto 通过合计每个 task 任务的总 CPU 使用时间，从而将他们分到五个不同等级的队列而不是仅仅通过提前预测一个新的查询所需的时间的方式。如果累积的 Cpu 使用时间越多，那么它的分层会越高。Presto 会为每一个曾分配一定的 CPU 总占用时间。

调度器也会自适应的处理一些情况，如果一个操作占用超时，调度器会记录他实际占用线程的时长，并且会临时减少它接下来的执行次数。这种方式有利于处理多种多样的查询类型。给一些低耗时的任务更高的优先级，这也符合低耗时任务往往期望尽快处理完成，而高耗时的任务对时间敏感性低的实际。

（2）内存管理

在像 Presto 这样的多租户系统中，内存是主要的资源管理挑战之一。

1\. 内存池

在 Presto 中，内存被分成用户内存和系统内存，这两种内存被保存在内存池中。用户内存是指用户可以仅根据系统的基本知识或输入数据进行推理的内存使用情况 (例如，聚合的内存使用与其基数成比例)。另一方面，系统内存是实现决策(例如 shuffle 缓冲区) 的副产品，可能与查询和输入数据量无关。换句话说，用户内存是与任务运行有关的，我们可以通过自己的程序推算出来运行时会用到的内存，系统内存可能更多的是一些不可变的。

Presto 引擎对单独对用户内存和总的内存（用户 + 系统）进行不同的规则限制，如果一个查询超过了全局总内存或者单个节点内存限制，这个查询将会被杀掉。当一个节点的内存耗尽时，该查询的预留内存会因为任务停止而被阻塞。

有时候，集群的内存可能会因为数据倾斜等原因造成内存不能充分利用，那么 Presto 提供了两种机制来缓解这种问题 -- 溢写和保留池。

2\. 溢写

当某一个节点内存用完的时候，引擎会启动内存回收程序，现将执行的任务序列进行升序排序，然后找到合适的 task 任务进行内存回收（也就是将状态进行溢写磁盘），知道有足够的内存来提供给任务序列的后一个请求。

3\. 预留池

如果集群的没有配置溢写策略，那么当一个节点内存用完或者没有可回收的内存的时候，预留内存机制就来解除集群阻塞了。这种策略下，查询内存池被进一步分成了两个池：普通池和预留池。这样当一个查询把普通池的内存资源用完之后，会得到所有节点的预留池内存资源的继续加持，这样这个查询的内存资源使用量就是普通池资源和预留池资源的加和。为了避免死锁，一个集群中同一时间只有一个查询可以使用预留池资源，其他的任务的预留池资源申请会被阻塞。这在某种情况下是优点浪费，集群可以考虑配置一下去杀死这个查询而不是阻塞大部分节点。

## 五、Presto 调优

**合理设置分区**  
与 Hive 类似，Presto 会根据元信息读取分区数据，合理的分区能减少 Presto 数据读取量，提升查询性能。

**使用列式存储**  
Presto 对 ORC 文件读取做了特定优化，因此在 Hive 中创建 Presto 使用的表时，建议采用 ORC 格式存储。相对于 Parquet，Presto 对 ORC 支持更好。

**使用压缩**  
数据压缩可以减少节点间数据传输对 IO 带宽压力，对于即席查询需要快速解压，建议采用 snappy 压缩

**预排序**  
对于已经排序的数据，在查询的数据过滤阶段，ORC 格式支持跳过读取不必要的数据。比如对于经常需要过滤的字段可以预先排序。

**内存调优**  
Presto 有三种内存池，分别为 GENERAL_POOL、RESERVED_POOL、SYSTEM_POOL。

-   GENERAL_POOL：用于普通查询的 physical operators。GENERAL_POOL 值为 总内存（Xmx 值）- 预留的（max-memory-per-node）- 系统的（0.4 \* Xmx）。
-   SYSTEM_POOL：系统预留内存，用于读写 buffer，worker 初始化以及执行任务必要的内存。大小由 config.properties 里的 resources.reserved-system-memory 指定。默认值为 JVM max memory \* 0.4。
-   RESERVED_POOL：大部分时间里是不参与计算的，只有当同时满足如下情形下，才会被使用，然后从所有查询里获取占用内存最大的那个查询，然后将该查询放到 RESERVED_POOL 里执行，同时注意 RESERVED_POOL 只能用于一个 Query。大小由 config.properties 里的 query.max-memory-per-node 指定，默认值为：JVM max memory \* 0.1。

这三个内存池占用的内存大小是由下面算法进行分配的：

    builder.put(RESERVED_POOL, new MemoryPool(RESERVED_POOL, config.getMaxQueryMemoryPerNode()));builder.put(SYSTEM_POOL, new MemoryPool(SYSTEM_POOL, systemMemoryConfig.getReservedSystemMemory()));long maxHeap = Runtime.getRuntime().maxMemory();maxMemory = new DataSize(maxHeap - systemMemoryConfig.getReservedSystemMemory().toBytes(), BYTE);DataSize generalPoolSize = new DataSize(Math.max(0, maxMemory.toBytes() - config.getMaxQueryMemoryPerNode().toBytes()), BYTE);builder.put(GENERAL_POOL, new MemoryPool(GENERAL_POOL, generalPoolSize));

简单的说，RESERVED_POOL 大小由 config.properties 里的 query.max-memory-per-node 指定；SYSTEM_POOL 由 config.properties 里的 resources.reserved-system-memory 指定，如果不指定，默认值为 Runtime.getRuntime().maxMemory() _0.4，即 0.4_ Xmx 值。而 GENERAL_POOL 值为：

总内存（Xmx 值）- 预留的（max-memory-per-node）- 系统的（0.4 \* Xmx）。

从 Presto 的开发手册中可以看到：

    GENERAL_POOL is the memory pool used by the physical operators in a query.SYSTEM_POOL is mostly used by the exchange buffers and readers/writers.RESERVED_POOL is for running a large query when the general pool becomes full.

简单说 GENERAL_POOL 用于普通查询的 physical operators；SYSTEM_POOL 用于读写 buffer；而 RESERVED_POOL 比较特殊，大部分时间里是不参与计算的，只有当同时满足如下情形下，才会被使用，然后从所有查询里获取占用内存最大的那个查询，然后将该查询放到 RESERVED_POOL 里执行，同时注意 RESERVED_POOL 只能用于一个 Query。

我们经常遇到的几个错误：

    Query exceeded per-node total memory limit of xx适当增加query.max-total-memory-per-node。Query exceeded distributed user memory limit of xx适当增加query.max-memory。Could not communicate with the remote task. The node may have crashed or be under too much load内存不够，导致节点crash，可以查看/var/log/message。

**并行度**  
调整线程数增大 task 的并发以提高效率。  
修改参数

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztibkrzUnHpOtd04yHjRxWcEUJXVVyYUfhnY1LaowMWhUUs714S0kmiaeQ/640?wx_fmt=png)

**SQL 优化**

-   只选择使用必要的字段：由于采用列式存储，选择需要的字段可加快字段的读取、减少数据量。避免采用 \* 读取所有字段
-   过滤条件必须加上分区字段
-   Group By 语句优化：合理安排 Group by 语句中字段顺序对性能有一定提升。将 Group By 语句中字段按照每个字段 distinct 数据多少进行降序排列， 减少 GROUP BY 语句后面的排序一句字段的数量能减少内存的使用.
-   Order by 时使用 Limit， 尽量避免 ORDER BY：Order by 需要扫描数据到单个 worker 节点进行排序，导致单个 worker 需要大量内存
-   使用近似聚合函数：对于允许有少量误差的查询场景，使用这些函数对查询性能有大幅提升。比如使用 approx_distinct() 函数比 Count(distinct x) 有大概 2.3% 的误差
-   用 regexp_like 代替多个 like 语句：Presto 查询优化器没有对多个 like 语句进行优化，使用 regexp_like 对性能有较大提升
-   使用 Join 语句时将大表放在左边：Presto 中 join 的默认算法是 broadcast join，即将 join 左边的表分割到多个 worker，然后将 join 右边的表数据整个复制一份发送到每个 worker 进行计算。如果右边的表数据量太大，则可能会报内存溢出错误。
-   使用 Rank 函数代替 row_number 函数来获取 Top N
-   UNION ALL 代替 UNION ：不用去重
-   使用 WITH 语句：查询语句非常复杂或者有多层嵌套的子查询，请试着用 WITH 语句将子查询分离出来

**元数据缓存**  
Presto 支持 Hive connector，元数据存储在 Hive metastore 中，调整元数据缓存的相关参数可以提高访问元数据的效率。  
修改参数

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztYLAfLfUTHODdKxeqpqIQMZIGWXUyMjqX0riaXTpwQPI0zssczomlaTQ/640?wx_fmt=png)

**Hash 优化**  
针对 Hash 场景的优化。  
修改参数

![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztGib8EX9M2kbySqNNITlrfjF8EoNST7Ge3SBFuhMYW9RaEOjc2eaibQJg/640?wx_fmt=jpeg)

**优化 OBS 相关参数**  
Presto 支持 on OBS，读写 OBS 过程中可以调整 OBS 客户端参数来提交读写效率。  
修改参数

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztmYVswazKL3GZDltlDnPKW8IAHwicmoVJ5m981cs4FwWVOiaQPWGTRvkA/640?wx_fmt=png)

### 六、Presto 数据模型

Presto 采取了三层表结构，我们可以和 Mysql 做一下类比：

-   catalog 对应某一类数据源，例如 hive 的数据，或 mysql 的数据
-   schema 对应 mysql 中的数据库
-   table 对应 mysql 中的表

在 Presto 中定位一张表，一般是 catalog 为根，例如：一张表的全称为 hive.testdata.test，标识 hive(catalog) 下的 testdata(schema) 中 test 表。

可以简理解为：数据源. 数据库. 数据表。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnztj1KDFzN4hAZKh7aibIKcNib5bC1BaGFrsHicmsgHztHdGE9LcvynEUicLg/640?wx_fmt=png)

另外，presto 的存储单元包括：

-   Page：多行数据的集合，包含多个列的数据，内部仅提供逻辑行，实际以列式存储。
-   Block：一列数据，根据不同类型的数据，通常采取不同的编码方式，了解这些编码方式，有助于自己的存储系统对接 presto。

Presto 中处理的最小数据单元是一个 Page 对象，Page 对象的数据结构如下图所示。一个 Page 对象包含多个 Block 对象，每个 Block 对象是一个字节数组，存储一个字段的若干行。多个 Block 横切的一行是真实的一行数据。一个 Page 最大 1MB，最多 16 \* 1024 行数据。

**核心问题之 Presto 为什么这么快？**

我们在选择 Presto 时很大一个考量就是计算速度，因为一个类似 SparkSQL 的计算引擎如果没有速度和效率加持，那么很快就就会被抛弃。

美团的博客中给出了这个答案：

-   完全基于内存的并行计算
-   流水线式的处理
-   本地化计算
-   动态编译执行计划
-   小心使用内存和数据结构
-   类 BlinkDB 的近似查询
-   GC 控制

和 Hive 这种需要调度生成计划且需要中间落盘的核心优势在于：Presto 是常驻任务，接受请求立即执行，全内存并行计算；Hive 需要用 yarn 做资源调度，接受查询需要先申请资源，启动进程，并且中间结果会经过磁盘。

### 七、行业典型应用

**Presto 在滴滴的应用**

滴滴 Presto 用了 3 年时间逐渐接入公司各大数据平台，并成为了公司首选 Ad-Hoc 查询引擎及 Hive SQL 加速引擎，支持了包含以下的业务场景：

-   Hive SQL 查询加速
-   数据平台 Ad-Hoc 查询
-   报表（BI 报表、自定义报表）
-   活动营销
-   数据质量检测
-   资产管理
-   固定数据产品

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnzticurVy6f6fgUGUra4u88ibL9NTb310YWaJHeIYc25PrV7QHomqUgKabQ/640?wx_fmt=png)

为了适配各个业务线，二次开发了 JDBC、Go、Python、Cli、R、NodeJs 、HTTP 等多种接入方式，打通了公司内部权限体系，让业务方方便快捷的接入 Presto 的，满足了业务方多种技术栈的接入需求。

Presto 接入了查询路由 Gateway，Gateway 会智能选择合适的引擎，用户查询优先请求 Presto，如果查询失败，会使用 Spark 查询，如果依然失败，最后会请求 Hive。在 Gateway 层，我们做了一些优化来区分大查询、中查询及小查询，对于查询时间小于 3 分钟的，我们即认为适合 Presto 查询，比如通过 HBO（基于历史的统计信息）及 JOIN 数量来区分查询大小，架构图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0NnztWc40IsmAL5rQHHgI4ibW4ROCpaFnoZTBvQCiaV8gJ3c7dFZJjiaSumxpg/640?wx_fmt=png)

在滴滴内部，Presto 主要用于 Ad-Hoc 查询及 Hive SQL 查询加速，为了方便用户能尽快将 SQL 迁移到 Presto 引擎上，且提高 Presto 引擎查询性能，我们对 Presto 做了大量二次开发。这些功能主要包括：

-   Hive SQL 兼容
-   物理资源隔离
-   直连 Druid 的 Connector
-   多租户等

Presto 在使用过程中会遇到很多稳定性问题，比如 Coordinator OOM，Worker Full GC 等。

滴滴给我们总结了 Coordinator 常见的问题和解决方法：

-   使用 HDFS FileSystem Cache 导致内存泄漏，解决方法禁止 FileSystem Cache，后续 Presto 自己维护了 FileSystem Cache
-   Jetty 导致堆外内存泄漏，原因是 Gzip 导致了堆外内存泄漏，升级 Jetty 版本解决
-   Splits 太多，无可用端口，TIME_WAIT 太高，修改 TCP 参数解决
-   Presto 内核 Bug，查询失败的 SQL 太多，导致 Coordinator 内存泄漏，社区已修复

而 Presto Worker 主要用于计算，性能瓶颈点主要是内存和 CPU。内存方面通过三种方法来保障和查找问题：

-   通过 Resource Group 控制业务并发，防止严重超卖
-   通过 JVM 调优，解决一些常见内存问题，如 Young GC Exhausted
-   善用 MAT 工具，发现内存瓶颈

**Presto 在有赞的应用**

有赞在 Presto 上主要用来进行以下业务支持：

-   数据平台 (DP) 的临时查询: 有赞的大数据团队使用临时查询进行探索性的数据分析的统一入口，同时也提供了脱敏，审计等功能。
-   BI 报表引擎：为商家提供了各类分析型的报表。
-   元数据数据质量校验等：元数据系统会使用 Presto 进行数据质量校验。
-   数据产品：比如 CRM 数据分析，人群画像等会使用 Presto 进行计算。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2MdPTD0NNkxDcvSQibH0Nnzt2uBibVcOfiaqcyib9QK4oiawoqL8AphtpXqkXSdH10rST7wKX5soEPnjJA/640?wx_fmt=png)

当然，有赞在使用 Presto 的过程中也经历了漫长的迭代：

-   第一阶段: Presto 和 Hadoop 混合部署
-   第二阶段: Presto 集群完全独立阶段
-   第三阶段: 低延时业务专用 Presto 集群阶段

在第二阶段的资源隔离主要还是靠 Resource Group，但是这种隔离方式相对比较弱，不能提供细粒度的隔离，任务之间还是会互相影响。此外，不同业务的 sql 类型，查询数据量，查询时间，可容忍的 SLA，可提供的最优配置都是不一样的。有些业务方需要一个特别低的响应时间保证，于是有赞给这类业务部署了专门的集群去处理。

部署在这个集群上的业务要求低延时，通常是 3 秒内，甚至有些能够达到 1 秒内，而且会有一定量的并发。不过这类业务通常数据量不是非常大，而且通常都是大宽表，也就不需要再去 Join 别的数据，Group By 形成的 Group 基数和产生的聚合数据量不是特别大，查询时间主要消耗在数据扫描读取时间上。同样也提供了资源完全独立，具有本地 HDFS 的专用 Presto 集群给这类业务方去使用。此外，会为这种业务提供深度的性能测试，调整相应的配置，比如将 Task Concurrency 改成 1，在并发量高的测试场景中，反而由于减少了线程间切换，性能会更好。

最后，有赞在使用 Presto 的过程中发生的主要问题包括：

-   HDFS 小文件问题

HDFS 小文件问题在大数据领域是个常见的问题。数仓 Hive 表有些表的文件有几千个，查询特别慢。Presto 下面这两个参数限制了 Presto 每个节点每个 Task 可执行的最大 Split 数目：

    node-scheduler.max-splits-per-node=100node-scheduler.max-pending-splits-per-task=10

-   多个列 Distinct 的问题

简单的说，正常的优化器应该使用 grouping sets 去将多个 group by 整合到一起来提升性能：

       SELECT a1, a2,..., an, F1(b1), F2(b2), F3(b3), ...., Fm(bm), F1(distinct c1), ...., Fm(distinct cm) FROM Table GROUP BY a1, a2, ..., an   转换为   SELECT a1, a2,..., an, arbitrary(if(group = 0, f1)),...., arbitrary(if(group = 0, fm)), F(if(group = 1, c1)), ...., F(if(group = m, cm)) FROM       SELECT a1, a2,..., an, F1(b1) as f1, F2(b2) as f2,...., Fm(bm) as fm, c1,..., cm group FROM         SELECT a1, a2,..., an, b1, b2, ... ,bn, c1,..., cm FROM Table GROUP BY GROUPING SETS ((a1, a2,..., an, b1, b2, ... ,bn), (a1, a2,..., an, c1), ..., ((a1, a2,..., an, cm)))       GROUP BY a1, a2,..., an, c1,..., cm group   GROUP BY a1, a2,..., an

但是很遗憾，Presto 并没有实现这样的功能。以上就是有赞在使用 Presto 的一些经验。

### 八、总结

小编在学习 Presto 的过程中和其他的 OLAP 一样，也是通过漫长的文档搜索，官网摸索主键精进的，事实上在任何一门新技术的使用过程中大家都会遇到各种各样的问题，如果利用现在有的资料解决问题就是考验我们的时候了。

![](https://mmbiz.qpic.cn/mmbiz_jpg/BSBqCXrZtzAicMToibKuIysLrB62M5A5YaLhZg6z86tI7ZeEZqTLLYyNrmlzrkyKUN5kNeUFicVC3bMP1GEqKz1OQ/640?wx_fmt=jpeg)

[Apache Iceberg 技术调研 & 在各大公司的实践应用大总结](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503879&idx=2&sn=bd009e298f2bdf9bb8abc9271b515143&chksm=fd3e9692ca491f84722c922000754aafc4d5a271b0ee40d525ce177de3c4442ef83b5ade8e05&scene=21#wechat_redirect)  

[放完假先收心 | 「个人经历不可替代」](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503810&idx=1&sn=1fbac632623be69a7d466d9d1566770d&chksm=fd3e8957ca4900414106f95e4e33f5dd949dbca7e76ed1656376448380eb3d4520ceebad309c&scene=21#wechat_redirect)  

[标签体系下的用户画像建设小指南](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503741&idx=1&sn=e5039be93123f2e337013756a818bfc3&chksm=fd3e89e8ca4900fe603b63c5722a6fb8a32bd63d6ba23e0028851948a71b877eb1f742d95087&scene=21#wechat_redirect)  

[4 万字长文 | ClickHouse 基础 & 实践 & 调优全视角解析](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503675&idx=1&sn=3ee6af64d0126c78b48cad219308f81e&chksm=fd3e89aeca4900b8b8954e9569ee3c0877881fac8c792bfafc22e7e9d3e8524da8eb860d33d8&scene=21#wechat_redirect)  

[数据仓库体系建模 & 实施 & 注意事项小总结](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503576&idx=1&sn=f9fc428799e0fcc78e94360e1cec7b95&chksm=fd3e884dca49015b6d38c437f603b4deffeeb0cefafd32d358a891bfe820f734116100395bba&scene=21#wechat_redirect) 
 [https://mp.weixin.qq.com/s/8CWXY864_QSmwNZ6Xh5B3w](https://mp.weixin.qq.com/s/8CWXY864_QSmwNZ6Xh5B3w)
