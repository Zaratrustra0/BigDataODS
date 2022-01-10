# mp.weixin.qq.com/s/lTdluk1CLjS2R8M8tDdXKQ
![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**一、问题分析概览**

流计算作业通常运行时间长，数据吞吐量大，且对时延较为敏感。但实际运行中，Flink 作业可能因为各种原因出现吞吐量抖动、延迟高、快照失败等突发情况，甚至发生崩溃和重启，影响输出数据的质量，甚至会导致线上业务中断，造成报表断崖、监控断点、数据错乱等严重后果。  

本文会对 Flink 常见的问题进行现象展示，从原理上说明成因和解决方案，并给出线上问题排查的工具技巧，帮助大家更好地应对 Flink 作业的异常场景。

### **如何分析** **F\*\***link 问题？\*\*

下图描述了遇到 Flink 问题时，建议的处理步骤：

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGHl3cuFzwAu3aEJ6IcRkcGq405VzzHJmnbHZQ7Tm1VicT7lYUv4HPt8A/640?wx_fmt=png)

发生问题时，首先要做的是现象记录，即检查作业的运行状态。如果运行状态不是运行中，那肯定没有数据正常输出了，需要进一步从日志中查找问题根因。如果作业在运行中，但是存在近期的重启记录，也表明可能发生了较严重的问题。此时需要整理问题发生的时间线，便于后续定位参考。

作业的吞吐和延时等指标是作业运行是否正常的判断标准。如果一个运行中的作业输出中断、数据量变小等现象，则首先需要观察是否存在严重的背压（也称反压，即 BackPressure. 后文会细讲如何判定）。如果存在背压，则需根据定位表，找到问题算子并进行瓶颈分析定位。随后还可以查看快照的时长和大小等信息，如果快照过大（例如大于 1GB）或很长时间才完成，则可能对内存造成较大压力。

如果从指标上不能完全判断问题原因，则需要结合完整的日志进行更细致的追查。后文会提到定位异常时常见的报错关键字，可以提升问题定位的速度。如果日志中没有太多有用的信息，则还需要对作业运行的环境进行检查，例如排除是否有其他进程干扰，系统是否被重启过，网络和磁盘是否存在瓶颈等等…

**二、常见问题处理**

这里我们总结了 Flink 作业的常见故障、确认方法和建议的解决措施。图中的 JM 表示 JobManager，TM 表示 TaskManager。

### **1.\*\***作业自动停止 \*\*

**现象：** 本应长期运行的作业，突然停止运行，且再也不恢复。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGKMPus0u6mwWHzRxfZTXWZtjJXQtTwHPYFpaKXVJscEjtwNLs8t0ISQ/640?wx_fmt=png)

如果 Flink 作业在编程时，源算子实现不当，则可能造成源算子处理完数据以后进入 FINISHED 状态。如果所有源算子都进入了 FINISHED 状态，那整个 Flink 作业也会跟着结束。

Flink 作业默认的容错次数是 2，即发生两次崩溃后，作业就自动退出了，不再进行重试。当出现此种场景时，TaskManager 的日志中会有 “restart strategy prevented it” 字样。我们首先要找到作业崩溃的原因，其次可以适当调大 RestartStrategy 中容错的最大次数，毕竟节点异常等外部风险始终存在，作业不会在理想的环境中运行。

此外，旧版 Flink（低于 1.11.0）的 RocksDB 内存使用不受管控，造成很容易由于超量使用而被外界（YARN、Kubernetes 等）KILL 掉。如果经常受此困扰，可以考虑升级 Flink 版本到最新，其默认开启自动内存管理功能。

### **2.\*\***输出量稳定但不及预期 \*\*

**现象：** 作业输出量较稳定，但是不及预期值（正常情况下，每核 5000 ~ 20000 条 / 秒）。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGpP4Cx8iajGvKdbicauSslzaAtx25jC3XZRUWaVIOCbKheRKakjVMevZA/640?wx_fmt=png)

如果作业输出量达不到预期，我们需要分别从 CPU、内存、磁盘、网络等方面逐一排查是否遇到了瓶颈。  

CPU 的瓶颈通常是因为序列化、反序列化开销较大，或者用户自定义算子的某个方法的时间复杂度高。CPU 瓶颈的定位较为简单，使用 JProfiler、jvmtop.sh 等工具均能较为准确的找到原因。

如果发现内存占比过高，那通常伴随着较长的 GC 时间，或者较多的 FullGC 次数。内存分析可以通过 jmap 把堆内存 dump 下来，然后使用 MemoryAnalyzer 等自动化工具进行泄露分析。

如果是因为数据倾斜原因，则网上已经有不少通用的解决方法，例如 key 打散、预聚合（Flink 中叫做 Local-Global Aggregation）等，建议结合具体业务场景选用。

还有一个常见的拖累吞吐的原因是访问外部（第三方）系统。假设每条数据都需要访问外部系统，每次需要 1ms，那么 1s 只能处理 1000 条数据，这当然是非常少的。如果需要频繁访问外部系统的话，建议充分利用批量存取和缓存、异步算子等功能，尽可能地减少交互次数。

### **3.\*\***输出量逐步减少或完全无输出 \*\*

**现象：** 作业输出量一开始较高，后来越来越少，甚至降到 0.

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGRIGnphjdRdEMnicaaNkKGjetnvicfj2FugV5xMh4VjBiaUEgo7QuiaKD6Q/640?wx_fmt=png)

作业输出量逐步减少的原因，最常见是背压较高和 Full GC 时间太长。当一个算子遇到 CPU 或者 I/O 瓶颈时，会造成输入缓冲区的数据积压，这样它的上游（运行图中的前一个算子）的输出缓冲区也会发生积压。就这样，一级一级向前传递，就会导致从数据源到问题算子的一条链路的数据都发生积压，这就是出现了 “背压” 现象。当然，如果算子的输出缓冲区写不出去（网络质量太差），也是可能引发背压的。

当我们在 Flink Web UI 界面上发现背压后，我们可以用后文中的 “背压分析表” 来定位可能的问题节点。

另外，如果没有发现严重的背压，但是数据输出量还是很少，就需要检查数据源、数据目的本身是否有问题。例如我们曾遇到过 MySQL 连接数满了导致数据源无法消费，或者下游数据目的经常连接超时造成数据无法稳定输出等。这些问题的排查思路都是控制变量法，去掉其他算子，构造一个单纯用到 Source 或 Sink 的作业，然后观察问题是否仍然存在。

最后，如果在日志里看到有数据错误的报错，尤其是那种疯狂写日志的场景，请务必引起重视。异常数据（数据输入格式与定义的 Schema 不一致）会造成计算结果错乱，还会造成磁盘空间被异常日志占满等严重问题。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGuULFT96Ux5mHuEZX5f2fMAmbXjfAicXYlribRbLX81x7SFKpj8qDxogw/640?wx_fmt=png)

如果我们发现算子背压较高，而且内存用量很大，JVM Full GC 时间很长，则说明堆内存用量太大了。Flink 的堆内存除了框架层面使用外，主要是用户定义的状态（含窗口等间接用到的状态）和运行时临时创建的对象占用了大部分内存。  

当状态过多时，如果启用了快照（Checkpoint），就会发现每次快照完成后的状态都很大，而且所需时间也较长。Flink 在快照过程中，会对所有状态做全量读取，如果是异步快照的话还有 Copy-On-Write 操作带来的内存压力，因此如果快照过大或者用时较长，也会造成内存中大量对象长期停留而无法被 GC 清理。

为了加速快照的执行，可以启用增量快照（目前只有 RocksDB State Backend 支持），或者如果有自定义快照的逻辑，请尽量避免 snapshotState() 方法耗时过长。另外如果在使用最新版本的 Flink（1.11 及以上），则可以开启 Unaligned Checkpoint 特性，该特性可以避免多个输入流的速度不同时（例如 JOIN 操作）快照带来的停顿和数据暂存开销。

窗口、GROUP BY 等算子（语句）都会用到大量状态数据，因此如果定义窗口的话，建议不要设置太大的窗口，或者太小的滑动时间（仅针对 Sliding Window 而言）。如果因为业务逻辑原因不得不用，则需要设置 Idle State Retention Time 以定期清理失效的状态。

如果用到了自定义的状态对象（StateDescriptor），则一定不要忘记清理或者设置 State TTL 以令 Flink 自动清理过期的状态。

还有一个不太常见但是会造成输出数据突然减少的原因是 Watermark 错乱。通常情况下我们的 Watermark 是基于输入数据时间戳来计算的，如果输入数据有明显的异常时间戳（例如 2050 年的某一天），则会将 Watermark 直接快进到那一天，从而令后续的正常数据被当作过期数据丢掉了。这就需要我们妥善定义 Watermark 的生成策略（忽略或矫正异常时间戳），或者对数据源的时间戳字段先做一遍清洗校验。相反，如果输入数据的时间戳一直不变（常见于测试数据，一直输入同一条），则会造成 Watermark 长期无法超过窗口的边界，这样窗口也会久久无法触发计算，从外部来看就是没有数据输出。

### **4.\*\***个别数据缺失 \*\*

**现象：** 作业输出整体稳定，但是个别数据缺失，造成结果的精度下降，甚至结果完全错乱。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGvh8VB1MCYg1rgYgibgY3OokfWOFiaCDwdVYX1KUicz1KEAb8Azlmo1MpA/640?wx_fmt=png)

当遇到怀疑数据缺失造成的计算结果不正确时，首先需要检查作业逻辑是否不小心过滤了一些正常数据。检查方法可以在本地运行一个 Mini Cluster，也可以在远端的调试环境进行远程调试或者采样等。具体技巧后文也会提到。

另外还有一种情况是，如果用户定义了批量存取的算子（通常用于与外部系统进行交互），则有可能出现一批数据中有一条异常数据，导致整批次都失败而被丢弃的情况。

对于数据源 Source 和数据目的 Sink，请务必保证 Flink 作业运行期间不要对其进行任何改动（例如新增 Kafka 分区、调整 MySQL 表结构等），否则可能造成正在运行的作业无法感知新增的分区或者读写失败。尽管 Flink 可以开启 Kafka 分区自动发现机制（在 Configuration 里设置 flink.partition-discovery.interval-millis 值），但分区发现仍然需要一定时间，数据的精度可能会稍有影响。

### **5.\*\***作业频繁重启 \*\*

**现象：** 作业频繁重启又自行恢复，陷入无尽循环，无法正常处理数据。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGkRFNxzibzcNShxiam4TYibZ2na3jQQpGEGb3CiaOWuO9lPiauNjObQ0Ifpg/640?wx_fmt=png)

作业频繁重启的成因非常多，例如异常数据造成的作业崩溃，可以在 TaskManager 的日志中找到报错。数据源或者数据目的等上下游系统超时也会造成作业无法启动而一直在重启。此外 TaskManager Full GC 太久造成心跳包超时而被 JobManager 踢掉也是常见的作业重启原因。如果系统内存严重匮乏，那么 Linux 自带的 OOM Killer 也可能把 TaskManager 所在的 JVM 进程 kill 了。  

当一个正常运行的作业失败时，日志里会有 from RUNNING to FAILED 的关键字，我们以此为着手点，查看它后面的 Exception 原因，通常最下面的 caused by 即是直接原因。当然，直接原因不一定等于根本原因，后者需要借助下文提到的多项技术进行分析。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGe26MicANGcH1LwicGJfas0ky4Q8kRfeP7O9WPWagTvUhXFlKCbiaFfajA/640?wx_fmt=png)

如果 JVM 的内存容量超出了平台方（例如 YARN 或 Kubernetes 等）的容器限制，则可能被 KILL。问题的确认方式也是查看作业日志以及平台组件的运行状态。值得一提的是，在最新的 Flink 版本中，只要设置 taskmanager.memory.process.size 参数，基本可以保证内存用量不会超过该值（前提是用户没有使用 JNI 等方式申请 native 内存）。  

作业的崩溃重启还有一些原因，例如使用了不成熟的第三方 so 库，或者连接数过多等，都可以从日志中找到端倪。

**三、问题追因技巧**

上面小节总结了 Flink 作业异常的常见现象和可能的原因，下面我们来介绍一下定位问题时常用的小工具和技巧，这对分析性能瓶颈非常有用。  

### **1.\*\***常用工具 \*\*

#### **内存**

• 堆内：jcmd、jmap、jstat、MemoryAnalyzer

• 堆外（Direct）：Natve Memory Tracking (NMT)

• Native：jemalloc（jeprof）、tcmalloc（pprof）

对于堆内内存，我们可以用 jcmd 命令开启 JavaFlight Recorder (JFR) 的录制功能，它会把 JVM 运行期间的各项指标等都保存在文件中，类似飞机的 “黑匣子”，可以后续分析。jmap 命令则可以把堆内存 dump 出来，随后可以配合 MemoryAnalyzer 分析是否有内存泄漏、占内存过多的对象等。jstat 命令则可以打印 GC 的统计指标，便于我们观察 GC 是否正常。

对于 JVM 的堆外内存，通常由 netty 等使用的 DirectByteBuffer 占用，可以使用 JVM 自带的 Native Memory Tracking 功能来记录和打印这些内存对象的分配情况。不过正常情况下用户代码不会涉及到这部分的内存。

如果使用 RocksDB 或者 JNI 调用了第三方的 so 库，那有可能会用到 malloc 函数。这样分配的内存是不受 JVM 管控的，因此如果需要定位这里的问题，需要使用 jemalloc 或 tcmalloc 动态替换原生的 malloc 实现，并开启 profiling 以追踪内存分配。

#### **CPU**

• 工具： JVisualVM、JProfiler、jstack、Arthas、JFR、jvmtop

很多工具都可以查看运行时 CPU 的使用情况。如果我们观察到 JVM 所在进程的 CPU 很繁忙，则需要找出热点方法（最高频、最耗时的）。如果空闲较多，则需要分析是否出现了死锁、I/O 等待等问题。

特别值得一提的是，jvmtop 是一个很好用的小工具，可以查看哪些方法占用 CPU 最高，并加以排序，这样我们可以很直观的找出潜在的问题点。JProfiler、Arthas 则是大而全的工具箱，里面提供了非常多的实用功能。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGF3Iyeg7yQEVoibjNcicE6dEwXOglv0NH1w2RvX8KIb0l7ortA3vfHic8w/640?wx_fmt=png)

图：使用 JProfiler 分析热点方法

#### **磁盘 I/O**

• iotop、dstat

如果想了解磁盘的使用情况，则可以用 iotop 等工具来查看 JVM 进程的磁盘读取和写入量。dstat 命令则可以持续的输出系统整体的磁盘读写情况。

#### **网络 I/O**

• 查看网络资源占用情况：nload、nethogs、iftop

• 检查是否丢包：tcpdump 抓包查看（注意内核缓冲区大小）

• 定位连接数过多的问题：netstat、ss

### **2.\*\***指标分析 \*\*

Flink 指标通常可以在自带的 Web UI 中查看，也可自定义 Metric Reporter，将指标输出到第三方系统，例如 Prometheus、InfluxDB、Elasticsearch 等等，随后可以展示为报表或进行告警等。

#### **重点关注的算子指标**

• 源算子（Source）：numRecordsOutPerSec（每秒产生的数据条数）、 numRecordsOut（产生的数据总条数）

• 目的算子（Sink）：numRecordsInPerSec（每秒接收的数据条数）、numRecordsIn（接收的数据总条数）

• 其他算子：除了观察上述吞吐量指标外，还可以观察 Watermark（时间戳是否符合预期）、Backpressure（红色 HIGH 表示背压高）。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGGU7iaA3InrcrUXYEkkrFtqn9dJaxTVzA4IX7fJIUBSYWacjUq7pqqFQ/640?wx_fmt=png)

图：查看算子的 Watermark、Backpressure 和各项统计指标

#### **背压分析**

首先我们来看一下为什么会出现背压高的现象。Flink 的每个算子都有输入缓冲区（InPool）和输出缓冲区（OutPool），它们的使用率分别在 Flink 指标里叫做 inPoolUsage 和 outPoolUsage。

特别提一下，在 Flink 1.9 及更高版本，inPoolUsage 还细分为 exclusiveBufferUsage（每个 Channel 独占的 Buffer）和 floatingBufferUsage（按照 Channel 需求，动态分配和归还的 Buffer）。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGwkn0brx7csvxoNvbF15NqTOLs69sHdpvxEKV78697EeONPJQnNQqog/640?wx_fmt=png)

因此，我们可以用下面的两个表格来定位背压产生的位置和可能原因（图片来自 Flink Network Stack Vol. 2: Monitoring, Metrics, and that Backpressure Thing 一文）。

如果 inPoolUsage 较低，而 outPoolUsage 也较低，则说明完全没有背压现象。若 inPoolUsage 较低，而 outPoolUsage 却很高，则说明处于临时状态，可能是背压刚开始，也可能是刚结束，需要再观察。

若 inPoolUsage 较高，而 outPoolUsage 低，那么通常情况下这个算子就是背压的根源了。但如果 inPoolUsage 较高，而 outPoolUsage 也较高的话，则说明这个算子是被其他下游算子反压而来的，并不是元凶。

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGA0Mz8UZoW0qMZPMNSDZY2AI4P74nt25fRfgH5eCABK7RY4ib386n1Kw/640?wx_fmt=png)

而对于 Flink 1.9 等以上版本，我们还可以用 floatingBufferUsage 和 exclusiveBufferUsage 来进一步定位问题，方法和上面的一致，即：首先看 floatingBufferUsage，然后看上游的 outPoolUsage，随后再看 exclusiveBufferUsage，这样就可以按表格中的对应项找到解答了：

![](https://mmbiz.qpic.cn/mmbiz_png/o2eWePa4j5AUjloGI1A8PmlgGjEc0UvGibR0toJ43HGM5516gFNYqyNwCmmk7Ytk461Q8fxNK4PcedStdZW1Vsg/640?wx_fmt=png)

当我们找到背压的算子以后，还需要用到上文提到的常用工具，进一步的定位产生的根源。

特别要注意的是，在背压定位过程中，建议关闭 Operator Chaining 优化，这样所有的算子可以单独拆分出来，不至于相互干扰。

### **3.\*\***日志分析 \*\*

#### **收集哪些日志**

• JobManager 日志

• TaskManager 日志

• GC 日志（ -XX:+PrintGCDetails -XX:+PrintGCDateStamps）

• YARN / Kubernetes 日志

• 系统日志（/var/log 下的日志，以及 journalctl 等）

#### **问题定位关键字**

• from RUNNING to FAILED 可以查看作业崩溃的直接原因

• exit 可以查看进程的 ExitCode

• fatal 或 Fatal 可以看 JVM Core Dump 的报错，或者 Akka 报错

• shutting down JVM 可以看 Akka 的 akka.jvm-exit-on-fatal-error 报错

• java.lang.OutOfMemoryError 可以查看是否发生过 OOM

• timeout 或 Timeout 表示发生了超时，此时可以查看网络质量，确认是否存在大量丢包等情况

• Failure 可以查看 Checkpoint 失败的信息

• 搜索 Total time for which application threads were stopped: ，查看后面是否有很长时间的停顿。这个是 Full GC 导致的全局停止时间

• Kill 可以看到超用资源而被 YARN 强行 kill 的情况

• Exception 则可以看到其他的异常（不一定是原因，仅供参考）

• WARN ERROR 等则可以看到一些报错的日志（也不一定是原因）

#### **日志中常见的错误码**

• 239（-17）：Fatal uncaught exception, 通常是 OOM。

• 1： 组件初始化时出错。

• 2： 组件运行时报错，通常有 Fatal error occurred in the cluster entrypoint 字样。

• 1443：作业状态变成 FAILED 时会出现，可搜索 to FAILED 寻找原因。

• 243（-13）：严重错误，较少见，通常有 FATAL ERROR 字样。

• 31：命令行解析错误，或者 YARN 初始化错误，通常不会遇到。

• 128~159 通常是 KILL 信号导致的。例如 134 (core dumped SIGABRT) 可能是 JVM 异常，也可能是第三方 so 的问题。

• 特别地，137 是 SIGKILL（kill -9 导致，可能是 OOM Killer 或者人工调用），143 是普通 SIGTERM，可能是 YARN Kill，也可能是人工调用。

**四、总结**

本文讲述了 Flink 问题定位的思路，以及常见的问题现象和解决方案，以及一些定位的小技巧。希望能够通过本文，增强大家对 Flink 的常见问题定位的思路。  

此外，如果遇到了难以解决的问题，通过上述的分析还是解决不了的话，还可以通过向社区发邮件（[https://flink.apache.org/zh/community.html）的方式来获取帮助。社区的](https://flink.apache.org/zh/community.html）的方式来获取帮助。社区的) Committer 通常会很快回复，如果确认是 Flink 的 bug 的话，则可以提交一个 JIRA 单来追踪这个问题。需要注意的是，提问时应当准确描述问题的现象、Flink 版本、最小复现方式等，最好可以附上日志和运行的环境等信息。

最后，祝各位 Flink 玩的愉快 ：） 
 [https://mp.weixin.qq.com/s/lTdluk1CLjS2R8M8tDdXKQ](https://mp.weixin.qq.com/s/lTdluk1CLjS2R8M8tDdXKQ)
