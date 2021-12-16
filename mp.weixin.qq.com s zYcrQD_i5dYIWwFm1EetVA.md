# mp.weixin.qq.com/s/zYcrQD_i5dYIWwFm1EetVA
我们线上有一套 clickhouse 集群，5 分片 2 副本总计 10 个实例，每个实例独占 1 台物理机，配套混布一个 3 节点 zookeeper 集群。

软件版本：centos 7.5  + CK 19.7.3 + ZK 3.4.13

从昨天开始应用写入日志开始堆积，并不断的报错 zookeeper session timeout。

登录机器查看 clickhouse 的 errlog，大量的 timeout 信息：

`2021.09.29 05:48:19.940814 [32] {} <Warning> app.log_dev_local (ReplicatedMergeTreeRestartingThread): ZooKeeper session has expired. Switching to a new session.  
2021.09.29 05:48:19.949000 [ 25 ] {} <Warning> app.log_k8s_local (ReplicatedMergeTreeRestartingThread): ZooKeeper session has expired. Switching to a new session.  
2021.09.29 05:48:19.952341 [ 30 ] {} <Error> app.log_dev_local (ReplicatedMergeTreeRestartingThread): void DB::ReplicatedMergeTreeRestartingThread::run(): Code: 999, e.displayText() = Coordination::Exception: All con  
nection tries failed while connecting to ZooKeeper. Addresses: 10.1.1.1:2181, 10.1.1.1.2:2181, 10.1.1.3:2181  
`

查看 zookeeper 状态，3 个实例都执行

`echo stat|nc 127.0.0.1 2181  
`

![](https://mmbiz.qpic.cn/mmbiz_png/qaWuZTGK0vzCow8qNgibPJQA1tdZjicRVehMpCKj8jia87aL7kXuhh3yNsusBjyZic68sPJHkl5h6cnHAsnzrgFdTw/640?wx_fmt=png)

返回 1 个 leader 2 个 follower，集群状态是正常的，但是该命令执行很慢。

尝试登录 zk 实例

`sh /usr/lib/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181  
`

进入登录界面后执行 ls / 卡顿半天，然后返回 timeout。应该是 ZK 集群通信出了问题，先对其进行滚动重启，重启后问题依然存在。

尝试调优 zk 参数，当前参数为

`tickTime= 2000   
syncLimit = 10  
minSessionTimeout = 4000  
maxSessionTimeout = 120000  
forceSync=yes  
leaderServes = yes  
`

调整成

`tickTime= 2000   
syncLimit = 100  
minSessionTimeout = 40000  
maxSessionTimeout = 600000  
forceSync= no  
leaderServes = no  
`

重启集群后问题依然存在。

我们线上有 4 套 CK 集群，每套都独占一套 zk，其余 3 套集群的 zookeeper 的内存只有几十 K，Node count 只有几万，而出问题的这套 Node count 有 2 千多万，zookeeper 进程内存 30G 左右。

现在怀疑是 Node count 过多，导致节点通信拥堵，于是想办法降低 Node 数量：

-   该环境有一套 kafka 混用了 ZK 集群，为其搭建了一套专属 ZK 集群并将 ZK 元数据目录删除，node count 和物理内存依然很高，问题没有解决。
-   清理无用表，找出 600 多个表，执行 drop 后，node count 和物理内存依然很高，问题没有解决。降低 Node count 的尝试失败。

排查到现在，基本能想到的招数都已用到，再整理一下思路：

-   ZK 节点响应很慢，但是集群状态是正常的；
-   CK 的 insert 经常超时，但是偶尔能执行成功；
-   增大 ZK 的超时参数，没有丝毫改善
-   ZK 的 node count 非常多，当前的 3 个 ZK 实例占用内存很高 (RSS 一直在 30G 上下浮动)

zookeeper 实例本质是 1 个 java 进程，有没有可能是达到内存上限频繁的触发 full gc，进而导致 ZK 服务响应经常性卡顿？

搜索半天没有在机器上发现 full gc 的日志记录，只能直接验证一下猜想。

在 / usr/lib/zookeeper/conf 目录下新建 1 个文件 java.env，内容如下：

`export JVMFLAGS="-Xms16384m -Xmx32768m $JVMFLAGS"  
`

滚动重启 ZK 集群，启动完毕后问题依然存在，但是 ZK 实例的 RSS 从原来的 30G 上升到了 33G，超出了 Xmx 上限。

应该是没吃饱，修改一下文件参数

`export JVMFLAGS="-Xms16384m -Xmx65536m $JVMFLAGS"  
`

再次重启 ZK 集群，ZK 实例的 RSS 飙升到 55G 左右就不再上升，困扰多时的问题也自动消失了，看来刚刚的 full gc 猜想是正确的。

既然已经证明是 JVM heap 内存的问题，那么刚刚调整的 ZK 参数就全部回滚，然后滚动重启 ZK 集群。

系统自此稳定了，但是 zk 进程占用的物理内存越来越大，没几天就达到了 64G，照这个消耗速度，256G 内存被耗光是迟早的事情。

为什么这套 zk 的 node count 会这么多，zk 进程的 RSS 这么大？

登录 zk，随意翻找一个表的副本目录，发现 parts 目录居然有 8000 多个 znode，![](https://mmbiz.qpic.cn/mmbiz_png/qaWuZTGK0vzCow8qNgibPJQA1tdZjicRVezBb2OaKvj2n4PnGWuOib6Qf6icgibHA5fUjpstV49lvH12ia8zNgMlXmvA/640?wx_fmt=png)

登录到 ck 实例，执行

`use system  
select substring(name,1,6),count(*) from zookeeper where path='/clickhouse/tables/01-01/db/table/replicas/ch1/parts' group by substring(name,1,6) order by substring(name,1,6);  
`

![](https://mmbiz.qpic.cn/mmbiz_png/qaWuZTGK0vzCow8qNgibPJQA1tdZjicRVe93AHsIxjIicorV5hUpvOibDXbNlM3rqlmhMW0SKzBLZsnZ5iaXmbKDPpw/640?wx_fmt=png)

该表自从 7 月后 znode part 数量就一路飙升，在 9 月末达到最高值。

尝试执行 optimize table table final，对降低 part 没什么用。

经和开发沟通后获悉，在 7 月的时候部分表的 insert 从每 10s 执行 1 次改成了 1s 执行 1 次，对应的就是 part 数量的飙升。

将这些表的 insert 统统改回了每 10s 执行 1 次，截止目前 (10 月 28 号)，10 月份的 part 基本回落到了一个正常值。

至于如何清理已有的 znode，目前有 2 种方法：

-   将部分离线表导出后 drop，然后再导入，操作后 znode 从 2400w 下降到了 1700w
-   大部分表的数据都有生命周期，N 个月后将不再需要的历史分区直接 drop

至少目前可以确信 znode 不会再暴涨，zk 进程的内存也不会继续增加，可以保证 clickhouse 集群稳定的运行下去。

这次案例前后耗费了 2 天的时间才得以定位原因并解决，又耗费了更长的时间才找到问题根源，距离发稿截止日期已经过了整整 1 个月，期间没有再复发过。  
java 进程对 Xms 和 Xmx 设置很敏感，线上应用要密切关注其内存占用情况。

> 作者简介: 任坤，现居珠海，先后担任专职 Oracle 和 MySQL DBA，现在主要负责 MySQL、mongoDB、Redis 和 Clickhouse 维护工作。 
>  [https://mp.weixin.qq.com/s/zYcrQD_i5dYIWwFm1EetVA](https://mp.weixin.qq.com/s/zYcrQD_i5dYIWwFm1EetVA)
