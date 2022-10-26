**1. 集群巡检**

HBase 是使用 HDFS 作为底层存储的 NoSQL 数据库，提供了满足实时性和随即读写功能的数据库服务。

每日早晚巡检 HBase 服务，检查各集群的 HMaster 和 RegionServer 状态，是否事务积压等问题。

**1.1 查看 Requests Per Second 和 Num.Regions**

若为图中所示为 0，则需要登录主机查看，通常这种情况会发生在重启节点主机后发生。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyDLSiaIqmBf7Bicgib7tDwSwwicHb8GprrdEp5SZl287j8BaZpuZqsbqRcw/640?wx_fmt=png)

**1.2 查看备用 HMaster**

每个库正常来说都有 3 个主节点，一个正在跑，两个备用，如图所示。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wybwic6ocR2dXlHzfsHbjyko0WsGmPZWYzuhkGBkAVEokMDV0LI9HZicqA/640?wx_fmt=png)

**1.3 查看 Software Attributes**

图中所示，比较重要的是两个标出部分：

1：代表着本库的 zookeeper 节点，如果出现异常，总数会不正常的

2：代表着本库的平均 region 数，理论超过 300 就要进行合并操作的，但这个是根据业务的需求进行操作，业务侧提出数据库卡顿了，再进行合并操作即可。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyRfC6OCnR6MoC8GRcI5PiaYhIjmOJCTqgJlpuhlm5OQyaAeN1fUEc4PA/640?wx_fmt=png)

**1.4 查看 Dead Region Servers**

此项是以库为单位，登录每个库的 HBase UI，若当前库内有 HBaseregionserver 宕掉的节点，则页面上会显示出如下情况：

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyY01icYXVxqhVuym3cXCPy5nHWDmLOydsLsg51h4JIozsGdT6gVxpfiag/640?wx_fmt=png)

出现这种情况，则说明当前库有非正常节点，可以尝试登陆该故障节点，查看故障原因（如 HBase 进程消失，主机意外重启，主机死机等）

**2. 参数调优**

**2.1 HBase HRegion 最大化压缩**

hbase.hregion.majorcompaction：所有 HStore-

Files“最大化” 压缩之间的时间，要禁用自动的最大化压缩，请将此值设置为 0。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyZ9ia4TuF8hZGKBxaT9qtVKa1SKric9pejlPNd0Io9C0iaJh9FARqEIXmg/640?wx_fmt=png)

**2.2 RegionServer 小型压缩线程计数**

hbase.regionserver.thread.compaction.small：

regionserver 做 Minor Compaction 时线程池里线程数目, 可以设置为 5

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyInbib6XrbLCo5D2zfLkVl0q7tOuG5SIOicGv0Jaho30sUy1de1BuwicgA/640?wx_fmt=png)

**2.3 HBase Region 分割限制**

hbase.regionserver.regionSplitLimit：控制最大的 region 数量, 超过则不可以进行 split 操作，默认是 2147483647，设置 1 可以禁止自动的 split，通过人工， 或者写脚本在集群空闲时执行。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wygH7tZgrfjLUPBFIJuZ7yvsZXXEvRFZichQsuDic166qxKDu9QJCuAR0Q/640?wx_fmt=png)

**2.4 HBase 文件最大大小**

hbase.hregion.max.filesize：默认是 10G， 如果任何一个 column familiy 里的 StoreFile 超过这个值, 那么这个 Region 会一分为二，因为 region 分 裂会有短暂的 region 下线时间 (通常在 5s 以内)，为减少对业务端的影响，建议手动定时分裂，可以设置大些。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wydUsgEYlAq1lro0ooa5SmRNibshOvTxlSglJxibPTeOEyTPASVceZ0uOg/640?wx_fmt=png)

**2.5 HBase 客户端写入缓冲**

hbase.client.write.buffer：客户端写 buffer，设置 autoFlush 为 false 时，当客户端写满 buffer 才 flush 默认为 2M，写缓存大小，推荐设置为 5M，单位是字节，当然越大占用的内存越多。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wyTPNHNf8zib75J01OSiaUF5OYmMrsY0DictUMUxlW8kRUdOwIKiaUXf8Etw/640?wx_fmt=png)

**2.6 HBase Region Server 处理程序计数**

hbase.regionserver.handler.count：该设置决定了处理 RPC 的线程数量，默认值是 30，通常可以调大，但不是越大越好，设置过大会占用过多的内存， 导致频繁的 gc，或者出现 oom。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wygZbyntbB9t0njEIxflVCm5ic0nmQP9T0YCY0DaAIMq7th8kxZcmCOaQ/640?wx_fmt=png)

**2.7 HFile 块缓存大小**

hfile.block.cache.size：默认值 0.25，regionser-

ver 的 block cache 的内存大小限制，在偏向读的业务中，可以适当调大该值，需要注意的是 hbase.regionserver.global.memstore.upperLimit 的值和 hfile.block.cache.size 的值之和必须小于 0.8。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wy0YFKbn8fJX8pLxzqhicmCbK0iasA5TC03Oae3nUXEYG1fsebKM2gqS8A/640?wx_fmt=png)

**2.8 RegionServer 中所有 Memstore 的最大大小**

hbase.regionserver.global.memstore.upperLimit,hbase.regionserver.global.memstore.size：默认值 0.4 这个参数的作用是防止内存占用过大，当 ReigonServer 内所有 region 的 memstores 所占用内存总和达到 heap 的 40% 时，HBase 会强制 block 所有的更新并 flush 这些 region 以释放所有 memstore 占用的内存。

**2.9 Memstore 刷新的低水位线**

hbase.regionserver.global.memstore.lowerLimit,hbase.regionserver.global.memstore.size.lower.limit：默认值 0.35 同 upperLimit，只不过 lowerLimit 在所有 region 的 memstores 所占用内存达到 Heap 的 35% 时，不 flush 所有的 memstore。它会找一个 memstore 内存占用最大的 region，做个别 flush，此时写更新还是会被 block。lowerLimit 算是一个在所有 region 强制 flush 导致性能降低前的补救措施。在日志中，表现为 “\*\* Flush thread woke up with memory above low water

**2.10 HBase Memstore 刷新大小**

hbase.hregion.memstore.flush.size：如 memst-

ore 大小超过此值（字节数），Memstore 将刷新到磁盘。这个参数的作用是当单个 Region 内所有的 memstore 大小总和超过指定值时，flush 该 region 的所有 memstore。RegionServer 的 flush 是通过将请求添加一个 队列，模拟生产消费模式来异步处理的。那这里就有一个问题，当队列来不及消费，产生大量积压请求时，可能会导致内存陡增，最坏的情况是触发 OOM。

![](https://mmbiz.qpic.cn/mmbiz_png/xqJ76ZLyAwAesdrFRQicaOwfyGzOLc4wy4In7U6Lar6klAB1Y1j6tKSJ3pYtiaHnuN4Ye3gAib5wGaZDOdgic2KRbw/640?wx_fmt=png)

 [https://mp.weixin.qq.com/s/qRI_6biH5Wt4Amk3Ph2Vvg](https://mp.weixin.qq.com/s/qRI_6biH5Wt4Amk3Ph2Vvg)
