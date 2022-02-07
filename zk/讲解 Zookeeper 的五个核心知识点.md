# 讲解 Zookeeper 的五个核心知识点
![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianM82mgaYdxeqvz0Vf1pS2xcIKeTscXBfcGTl4f44fWqEAMJIuU3HGjw/640?wx_fmt=png)

**ZooKeeper** 是一个开源的**分布式协调框架**，它的定位是为分布式应用提供一致性服务，是整个大数据体系的**管理员**。**ZooKeeper** 会封装好复杂易出错的关键服务，将高效、稳定、易用的服务提供给用户使用。

如果上面的官方言语你不太理解，你可以认为 **ZooKeeper** = **文件系统** + **监听通知机制**。

### 1.1 文件系统

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianzzRBtLyiaibt3zyrSMK8V3iccYGST92jgSu6XibNzQISxian8jJQqGP0Hfw/640?wx_fmt=png)

**Zookeeper**维护一个类似文件系统的树状数据结构，这种特性使得 **Zookeeper** 不能用于存放大量的数据，每个节点的存放数据上限为**1M**。每个子目录项如 NameService 都被称作为 **znode**(目录节点)。和文件系统一样，我们能够自由的增加、删除**znode**，在一个**znode**下增加、删除子**znode**，唯一的不同在于**znode**是可以存储数据的。默认有四种类型的**znode**：

1.  **持久化目录节点 PERSISTENT**：客户端与 zookeeper 断开连接后，该节点依旧存在。
2.  **持久化顺序编号目录节点 PERSISTENT_SEQUENTIAL**：客户端与 zookeeper 断开连接后，该节点依旧存在，只是 Zookeeper 给该节点名称进行顺序编号。
3.  **临时目录节点 EPHEMERAL**：客户端与 zookeeper 断开连接后，该节点被删除。
4.  **临时顺序编号目录节点 EPHEMERAL_SEQUENTIAL**：客户端与 zookeeper 断开连接后，该节点被删除，只是 Zookeeper 给该节点名称进行顺序编号。

### 1.2 监听通知机制

**Watcher** 监听机制是 **Zookeeper** 中非常重要的特性，我们基于  **Zookeeper** 上创建的节点，可以对这些节点绑定**监听**事件，比如可以监听节点数据变更、节点删除、子节点状态变更等事件，通过这个事件机制，可以基于  **Zookeeper** 实现分布式锁、集群管理等功能。

**Watcher** 特性：

> 当数据发生变化的时候，  **Zookeeper**  会产生一个 **Watcher** 事件，并且会发送到客户端。但是客户端只会收到一次通知。如果后续这个节点再次发生变化，那么之前设置 **Watcher** 的客户端不会再次收到消息。（**Watcher** 是一次性的操作）。可以通过循环监听去达到永久监听效果。

ZooKeeper 的 Watcher 机制，总的来说可以分为三个过程：

> 1.  客户端注册 Watcher，注册 watcher 有 3 种方式，getData、exists、getChildren。
> 2.  服务器处理 Watcher 。
> 3.  客户端回调 Watcher 客户端。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianEbvevhGd3ibof42d60FYPia6SMUWJz1UTeGFQq3ib4n9Aic42cAqonRq2A/640?wx_fmt=png)
**监听流程**：

> 1.  首先要有一个 main() 线程
> 2.  在 main 线程中创建 Zookeeper 客户端，这时就会创建两个线程，一个负责网络连接通信（connet），一个负责监听（listener）。
> 3.  通过 connect 线程将注册的监听事件发送给 Zookeeper。
> 4.  在 Zookeeper 的注册监听器列表中将注册的监听事件添加到列表中。
> 5.  Zookeeper 监听到有数据或路径变化，就会将这个消息发送给 listener 线程。
> 6.  listener 线程内部调用了 process() 方法。

### 1.3 Zookeeper 特点

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianEiaLJJJhHoYibFzOjOfq23G82K8D3b3QQvbzkl6HiaomQkVT5BSpC90Kw/640?wx_fmt=png)

1.  **集群**：Zookeeper 是一个领导者（Leader），多个跟随者（Follower）组成的集群。
2.  **高可用性**：集群中只要有半数以上节点存活，Zookeeper 集群就能正常服务。
3.  **全局数据一致**：每个 Server 保存一份相同的数据副本，Client 无论连接到哪个 Server，数据都是一致的。
4.  **更新请求顺序进行**：来自同一个 Client 的更新请求按其发送顺序依次执行。
5.  **数据更新原子性**：一次数据更新要么成功，要么失败。
6.  **实时性**：在一定时间范围内，Client 能读到最新数据。
7.  从`设计模式`角度来看，zk 是一个基于**观察者设计模式**的框架，它负责管理跟存储大家都关心的数据，然后接受观察者的注册，数据反生变化 zk 会通知在 zk 上注册的观察者做出反应。
8.  Zookeeper 是一个分布式协调系统，满足 CP 性，跟[SpringCloud](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247487072&idx=1&sn=3010908c4b668edef8fbd1f320343354&scene=21#wechat_redirect)中的 Eureka 满足 AP 不一样。

> 1.  分布式协调系统：Leader 会同步数据到 follower，用户请求可通过 follower 得到数据，这样不会出现单点故障，并且只要同步时间无限短，那这就是个好的 分布式协调系统。
> 2.  CAP 原则又称 CAP 定理，指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。

通过对 Zookeeper 中丰富的数据节点进行交叉使用，配合 **Watcher** 事件通知机制，可以非常方便的构建一系列分布式应用中涉及的核心功能，比如 **数据发布 / 订阅、负载均衡、命名服务、分布式协调 / 通知、集群管理、Master 选举、分布式锁和分布式队列** 等功能。

### 1. 数据发布 / 订阅

当某些数据由几个机器共享，且这些信息经常变化数据量还小的时候，这些数据就适合存储到 ZK 中。

-   **数据存储**：将数据存储到 Zookeeper 上的一个数据节点。
-   **数据获取**：应用在启动初始化节点从 Zookeeper 数据节点读取数据，并在该节点上注册一个数据变更 **Watcher**
-   **数据变更**：当变更数据时会更新 Zookeeper 对应节点数据，Zookeeper 会将数据变更**通知**发到各客户端，客户端接到通知后重新读取变更后的数据即可。

### 2. 分布式锁

关于分布式锁其实在 [**Redis**](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488832&idx=1&sn=5999893d7fe773f54f7d097ac1c2074d&scene=21#wechat_redirect) 中已经讲过了，并且 Redis 提供的分布式锁是比 ZK 性能强的。基于 ZooKeeper 的分布式锁一般有如下两种。

1.  保持独占

> **核心思想**：在 zk 中有一个唯一的临时节点，只有拿到节点的才可以操作数据，没拿到的线程就需要等待。**缺点**：可能引发`羊群效应`，第一个用完后瞬间有 999 个同时并发的线程向 zk 请求获得锁。

2.  控制时序

> 主要是避免了羊群效应，临时节点已经预先存在，所有想要获得锁的线程在它下面创建临时顺序编号目录节点，编号最小的获得锁，用完删除，后面的依次排队获取。

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBian3mKNDkyfibdlkibnBjMVGX33St57zc0p26kqObYp0ZqBuFlR7icqJMAVA/640?wx_fmt=png)

### 

3. 负载均衡

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianO9KCiaUGlGrCDaawOj46NxMJtmcA1yUp33d5MGrp2SWmgooPV98cfvA/640?wx_fmt=png)

多个相同的 jar 包在不同的服务器上开启相同的服务，可以通过 nginx 在服务端进行负载均衡的配置。也可以通过 ZooKeeper 在客户端进行负载均衡配置。

1.  多个服务注册
2.  客户端获取中间件地址集合
3.  从集合中随机选一个服务执行任务

**ZooKeeper 负载均衡和 Nginx 负载均衡区别**：

> 1.  **ZooKeeper**不存在单点问题，zab 机制保证单点故障可重新选举一个 leader 只负责服务的注册与发现，不负责转发，减少一次数据交换（消费方与服务方直接通信），需要自己实现相应的负载均衡算法。
> 2.  **Nginx**存在单点问题，单点负载高数据量大, 需要通过 **KeepAlived** + **LVS** 备机实现高可用。每次负载，都充当一次中间人转发角色，增加网络负载量（消费方与服务方间接通信），自带负载均衡算法。

### 4. 命名服务

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianId4lZl5oVdK3XHqK8OCpFm0pM8ANCcnvice5Y41oP92Su5HFZKcsmQA/640?wx_fmt=png)

命名服务是指通过指定的名字来获取资源或者服务的地址，利用 zk 创建一个全局唯一的路径，这个路径就可以作为一个名字，指向集群中的集群，提供的服务的地址，或者一个远程的对象等等。

### 5. 分布式协调 / 通知

1.  对于系统调度来说，用户更改 zk 某个节点的 value， ZooKeeper 会将这些变化发送给注册了这个节点的 watcher 的所有客户端，进行通知。
2.  对于执行情况汇报来说，每个工作进程都在目录下创建一个携带工作进度的临时节点，那么汇总的进程可以监控目录子节点的变化获得工作进度的实时的全局情况。

### 6. 集群管理

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianhiaDEYro7s0xbtmgu4gHT0bDicYnZZJRiasmIa1ZyDZf7CzTmM6O4ITgA/640?wx_fmt=png)

大数据体系下的大部分集群服务好像都通过**ZooKeeper**管理的，其实管理的时候主要关注的就是机器的动态上下线跟**Leader**选举。

1.  动态上下线：

> 比如在**zookeeper**服务器端有一个**znode**叫 **/Configuration**，那么集群中每一个机器启动的时候都去这个节点下创建一个**EPHEMERAL**类型的节点，比如**server1** 创建 **/Configuration/Server1**，**server2**创建**/Configuration /Server1**，然后**Server1**和**Server2**都**watch**  **/Configuration** 这个父节点，那么也就是这个父节点下数据或者子节点变化都会通知到该节点进行**watch**的客户端。

2.  Leader 选举：

> 1.  利用 ZooKeeper 的**强一致性**，能够保证在分布式高并发情况下节点创建的全局唯一性，即：同时有多个客户端请求创建 /Master 节点，最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很轻易的在分布式环境中进行集群选举了。
> 2.  就是动态 Master 选举。这就要用到 **EPHEMERAL_SEQUENTIAL**类型节点的特性了，这样每个节点会`自动被编号`。允许所有请求都能够创建成功，但是得有个创建顺序，每次选取序列号**最小**的那个机器作为**Master** 。

![](https://mmbiz.qpic.cn/mmbiz_gif/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianEzBYia9c9TekicNCTvKphkWwxYvubVtreiaN1IGxgicWMDicrz2w6MGjeDA/640?wx_fmt=gif)

**ZooKeeper**集群节点个数一定是**奇数**个，一般 3 个或者 5 个就 OK。为避免集群群龙无首，一定要选个大哥出来当 Leader。这是个高频考点。

### 3.1 预备知识

##### 3.1.1. 节点四种状态。

1.  **LOOKING**：寻 找 Leader 状态。当服务器处于该状态时会认为当前集群中没有 Leader，因此需要进入 Leader 选举状态。
2.  **FOLLOWING**：跟随者状态。处理客户端的非事务请求，转发事务请求给 Leader 服务器，参与事务请求 Proposal(提议) 的投票，参与 Leader 选举投票。
3.  **LEADING**：领导者状态。事务请求的唯一调度和处理者，保证集群事务处理的顺序性，集群内部个服务器的调度者 (管理 follower, 数据同步)。
4.  **OBSERVING**：观察者状态。3.0 版本以后引入的一个服务器角色，在不影响集群事务处理能力的基础上提升集群的非事务处理能力，处理客户端的非事务请求，转发事务请求给 Leader 服务器，不参与任何形式的投票。

##### 3.1.2 服务器 ID

既**Server id**，一般在搭建 ZK 集群时会在**myid**文件中给每个节点搞个唯一编号，`编号越大在 Leader 选择算法中的权重越大`，比如初始化启动时就是根据服务器 ID 进行比较。

### 3.1.3  ZXID

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianUdX199elT5DP7EwFPt8oEhdHnZXBpwUJ6jPcywWbnoiah37o18OgKvg/640?wx_fmt=png)

**ZooKeeper** 采用全局递增的事务 Id 来标识，所有 proposal(提议) 在被提出的时候加上了**ZooKeeper Transaction Id** ，zxid 是 64 位的 Long 类型，**这是保证事务的顺序一致性的关键**。zxid 中高 32 位表示纪元**epoch**，低 32 位表示事务标识**xid**。你可以认为 zxid 越大说明存储数据越新。

1.  每个 leader 都会具有不同的**epoch**值，表示一个纪元 / 朝代，用来标识 **leader** 周期。每个新的选举开启时都会生成一个新的**epoch**，新的 leader 产生的话**epoch**会自增，会将该值更新到所有的 zkServer 的**zxid**和**epoch**，
2.  **xid**是一个依次递增的事务编号。数值越大说明数据越新，所有 proposal（提议）在被提出的时候加上了**zxid**，然后会依据数据库的[两阶段过程](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485515&idx=1&sn=60763ddda77928943bfd3d57e0c9256e&scene=21#wechat_redirect)，首先会向其他的 server 发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。

### 3.2 Leader 选举

**Leader**的选举一般分为**启动时选举**跟 Leader 挂掉后的**运行时选举**。

##### 3.2.1 启动时 Leader 选举

![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianEiaLJJJhHoYibFzOjOfq23G82K8D3b3QQvbzkl6HiaomQkVT5BSpC90Kw/640?wx_fmt=png)
我们以上面的 5 台机器为例，只有超过半数以上，即最少启动 3 台服务器，集群才能正常工作。

1.  服务器 1 启动，发起一次选举。

> 服务器 1 投自己一票。此时服务器 1 票数一票，不够半数以上（3 票），选举无法完成，服务器 1 状态保持为**LOOKING**。

2.  服务器 2 启动，再发起一次选举。

> 服务器 1 和 2 分别投自己一票，此时服务器 1 发现服务器 2 的 id 比自己大，更改选票投给服务器 2。此时服务器 1 票数 0 票，服务器 2 票数 2 票，不够半数以上（3 票），选举无法完成。服务器 1，2 状态保持**LOOKING**。

3.  服务器 3 启动，发起一次选举。

> 与上面过程一样，服务器 1 和 2 先投自己一票，然后因为服务器 3id 最大，两者更改选票投给为服务器 3。此次投票结果：服务器 1 为 0 票，服务器 2 为 0 票，服务器 3 为 3 票。此时服务器 3 的票数已经超过半数（3 票），服务器 3 当选**Leader**。服务器 1，2 更改状态为**FOLLOWING**，服务器 3 更改状态为**LEADING**；

4.  服务器 4 启动，发起一次选举。

> 此时服务器 1、2、3 已经不是**LOOKING**状态，不会更改选票信息，交换选票信息结果。服务器 3 为 3 票，服务器 4 为 1 票。此时服务器 4 服从多数，更改选票信息为服务器 3，服务器 4 并更改状态为**FOLLOWING**。

5.  服务器 5 启动，发起一次选举

> 同 4 一样投票给 3，此时服务器 3 一共 5 票，服务器 5 为 0 票。服务器 5 并更改状态为**FOLLOWING**；

6.  最终

> **Leader**是服务器 3，状态为**LEADING**。其余服务器是**Follower**，状态为**FOLLOWING**。

##### 3.2.2 运行时 Leader 选举

运行时候如果 Master 节点崩溃了会走恢复模式，新 Leader 选出前会暂停对外服务，大致可以分为四个阶段  `选举`、`发现`、`同步`、`广播`。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianod76TCgeTydBrNmUd4vTZgkSpIHaITXtFXoDianwv18mYHPYnodQLgA/640?wx_fmt=png)

1.  每个 Server 会发出一个投票，第一次都是投自己，其中投票信息 = (myid，ZXID)
2.  收集来自各个服务器的投票
3.  处理投票并重新投票，处理逻辑：**优先比较 ZXID，然后比较 myid**。
4.  统计投票，只要超过半数的机器接收到同样的投票信息，就可以确定 leader，注意 epoch 的增加跟同步。
5.  改变服务器状态 Looking 变为 Following 或 Leading。
6.  当 Follower 链接上 Leader 之后，Leader 服务器会根据自己服务器上最后被提交的 ZXID 和 Follower 上的 ZXID 进行比对，比对结果要么回滚，要么和 Leader 同步，保证集群中各个节点的事务一致。
7.  集群恢复到广播模式，开始接受客户端的写请求。

### 3.3 脑裂

脑裂问题是集群部署必须考虑的一点，比如在 Hadoop 跟 Spark 集群中。而 ZAB 为解决脑裂问题，要求集群内的节点数量为 2N+1。当网络分裂后，始终有一个集群的节点数量**过半数**，而另一个节点数量小于 N+1, 因为选举 Leader 需要过半数的节点同意，所以我们可以得出如下结论：

> 有了过半机制，对于一个 Zookeeper 集群，要么没有 Leader，要没只有 1 个 Leader，这样就避免了脑裂问题

建议先看下  [浅谈大数据中的 2PC、3PC、Paxos、Raft、ZAB](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485515&idx=1&sn=60763ddda77928943bfd3d57e0c9256e&scene=21#wechat_redirect)  ，不然可能看的吃力。

### 4.1  ZAB 协议介绍

ZAB (Zookeeper Atomic Broadcast **原子广播协议**) 协议是为分布式协调服务 ZooKeeper 专门设计的一种支持**崩溃恢复**的一致性协议。基于该协议，ZooKeeper 实现了一种主从模式的系统架构来保持集群中各个副本之间的数据一致性。

分布式系统中 leader 负责外部客户端的**写**请求。follower 服务器负责**读跟同步**。这时需要解决俩问题。

1.  Leader 服务器是如何把数据更新到所有的 Follower 的。
2.  Leader 服务器突然间失效了，集群咋办？

因此 ZAB 协议为了解决上面两个问题而设计了两种工作模式，整个 Zookeeper 就是在这两个模式之间切换：

1.  原子广播模式：把数据更新到所有的 follower。
2.  崩溃恢复模式：Leader 发生崩溃时，如何恢复。

### 4.2 原子广播模式

你可以认为消息广播机制是简化版的 [2PC 协议](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247485515&idx=1&sn=60763ddda77928943bfd3d57e0c9256e&scene=21#wechat_redirect)，就是通过如下的机制**保证事务的顺序一致性**的。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianIjnFA186vSFoSbJENpxia7PIjJ2aRbU55ldaDVhrBlNibibsbXaZXaIkQ/640?wx_fmt=png)

1.  **leader**从客户端收到一个写请求后生成一个新的事务并为这个事务生成一个唯一的`ZXID`，
2.  **leader**将将带有 **zxid** 的消息作为一个提案 (**proposal**) 分发给所有 **FIFO**队列。
3.  **FIFO**队列取出队头**proposal**给**follower**节点。
4.  当 **follower** 接收到 **proposal**，先将 **proposal** 写到硬盘，写硬盘成功后再向 **leader** 回一个 **ACK**。
5.  **FIFO**队列把 ACK 返回给**Leader**。
6.  当**leader**收到超过一半以上的**follower**的**ack**消息，**leader**会进行**commit**请求，然后再给**FIFO**发送**commit**请求。
7.  当**follower**收到**commit**请求时，会判断该事务的**ZXID**是不是比历史队列中的任何事务的**ZXID**都小，如果是则提交，如果不是则等待比它更小的事务的**commit**(保证顺序性)

### 4.3 崩溃恢复

消息广播过程中，Leader 崩溃了还能保证数据一致吗？当 **Leader** 崩溃会进入崩溃恢复模式。其实主要是对如下两种情况的处理。

1.  **Leader** 在复制数据给所有 **Follwer** 之后崩溃，咋搞？
2.  **Leader** 在收到 Ack 并提交了自己，同时发送了部分 **commit** 出去之后崩溃咋办？

针对此问题，ZAB 定义了 2 个原则：

1.  ZAB 协议确保`执行`那些已经在 Leader 提交的事务最终会被所有服务器提交。
2.  ZAB 协议确保`丢弃`那些只在 Leader 提出 / 复制，但没有提交的事务。

至于如何实现**确保提交已经被 Leader 提交的事务，同时丢弃已经被跳过的事务**呢？关键点就是依赖上面说到过的 **ZXID**了。

### 4.4 ZAB 特性

1.  一致性保证

> 可靠提交 (Reliable delivery) ：如果一个事务 A 被一个 server 提交(committed) 了，那么它最终一定会被所有的 server 提交

2.  全局有序 (Total order)

> 假设有 A、B 两个事务，有一台 server 先执行 A 再执行 B，那么可以保证所有 server 上 A 始终都被在 B 之前执行

3.  因果有序 (Causal order)

> 如果发送者在事务 A 提交之后再发送 B, 那么 B 必将在 A 之后执行

4.  高可用性

> 只要大多数（法定数量）节点启动，系统就行正常运行

5.  可恢复性

> 当节点下线后重启，它必须保证能恢复到当前正在执行的事务

### 4.5 ZAB 和 Paxos 对比

相同点：

> 1.  两者都存在一个类似于 Leader 进程的角色，由其负责协调多个 Follower 进程的运行.
> 2.  Leader 进程都会等待超过半数的 Follower 做出正确的反馈后，才会将一个提案进行提交.
> 3.  ZAB 协议中，每个 Proposal 中都包含一个 epoch 值来代表当前的 Leader 周期，Paxos 中名字为 Ballot

不同点：

> ZAB 用来构建高可用的**分布式数据主备系统**（Zookeeper），Paxos 是用来构建**分布式一致性状态机系统**。

### 5.1 常见指令

Zookeeper 有三种部署模式：

> 1.  单机部署：一台机器上运行。
> 2.  集群部署：多台机器运行。
> 3.  伪集群部署：一台机器启动多个 Zookeeper 实例运行。

部署完毕后常见指令如下：

| 命令基本语法            | 功能描述                |
| ----------------- | ------------------- |
| help              | 显示所有操作命令            |
| ls path \[watch]  | 显示所有操作命令            |
| ls path \[watch]  | 查看当前节点数据并能看到更新次数等数据 |
| create            | 普通创建， -s  含有序列，     |
| -e  临时（重启或者超时消失）  |                     |
| get path \[watch] | 获得节点的值              |
| set               | 设置节点的具体值            |
| stat              | 查看节点状态              |
| delete            | 删除节点                |
| rmr               | 递归删除节点              |

### 5.2 Zookeeper 客户端

##### 5.2.1. Zookeeper 原生客户端

Zookeeper 客户端是异步的哦！需要引入 CountDownLatch 来确保连接好了再做下面操作。Zookeeper 原生 api 是不支持迭代式的创建跟删除路径的，具有如下弊端。

> 1.  会话的连接是异步的；必须用到回调函数 。
> 2.  Watch 需要重复注册：看一次 watch 注册一次 。
> 3.  Session 重连机制：有时 session 断开还需要重连接。
> 4.  开发复杂性较高：开发相对来说比较琐碎。

##### 5.2.2. ZkClient

开源的 zk 客户端，在原生 API 基础上封装，是一个更易于使用的 zookeeper 客户端，做了如下优化。

> 优化一 、在 session loss 和 session expire 时自动创建新的 ZooKeeper 实例进行重连。优化二、 将一次性 watcher 包装为持久 watcher。

##### 5.2.3. Curator

开源的 zk 客户端，在原生 API 基础上封装，apache 顶级项目。是 Netflix 公司开源的一套 Zookeeper 客户端框架。了解过 Zookeeper 原生 API 都会清楚其复杂度。Curator 帮助我们在其基础上进行封装、实现一些开发细节，包括接连重连、反复注册 Watcher 和 NodeExistsException 等。目前已经作为 Apache 的顶级项目出现，是最流行的 Zookeeper 客户端之一。

##### 5.2.4. Zookeeper 图形化客户端工具

工具名叫 ZooInspector，百度安装教程即可。![](https://mmbiz.qpic.cn/mmbiz_png/wJvXicD0z2dVavZ4ZibH7oem8qpv7CuBianibwIKeUkOXV3bAbUCgFruWzHqGOHEIHhp8VPGdyicqcKh3G8wIibJtzUg/640?wx_fmt=png)

### 5.3 ACL 权限控制机制

ACL 全称为 Access Control List 即访问控制列表，用于控制资源的访问权限。zookeeper 利用 ACL 策略控制节点的访问权限，如节点数据读写、节点创建、节点删除、读取子节点列表、设置节点权限等。

### 5.4 Zookeeper 使用注意事项

1.  集群中机器的数量并不是越多越好，一个写操作需要半数以上的节点 ack，所以集群节点数越多，整个集群可以抗挂点的节点数越多 (越可靠)，但是吞吐量越差。集群的数量必须为奇数。
2.  zk 是基于内存进行读写操作的，有时候会进行消息广播，因此不建议在节点存取容量比较大的数据。
3.  dataDir 目录、dataLogDir 两个目录会随着时间推移变得庞大，容易造成硬盘满了。建议自己编写或使用自带的脚本保留最新的 n 个文件。
4.  默认最大连接数 默认为 60，配置 maxClientCnxns 参数，配置单个客户端机器创建的最大连接数。

* * *

 [https://mp.weixin.qq.com/s/W6QgmFTpXQ8EL-dVvLWsyg](https://mp.weixin.qq.com/s/W6QgmFTpXQ8EL-dVvLWsyg)
