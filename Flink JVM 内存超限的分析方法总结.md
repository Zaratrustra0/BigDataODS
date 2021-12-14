# Flink JVM 内存超限的分析方法总结
[问题背景  
](http://mp.weixin.qq.com/s?__biz=MzIxMTE0ODU5NQ==&mid=2650246786&idx=1&sn=e09a1c66a43e1d504d4ee030e8ab6e7b&chksm=8f5ae4deb82d6dc851e765bfede239140802b5c020babaeeb0bec265eb5bc01d26ac0577fc4d&scene=21#wechat_redirect)

前段时间，某客户的大作业（并行度 200 左右）遇到了 TaskManager JVM 内存超限（实际内存用量 4.1G > 容器设定的最大阈值 4.0G），被 YARN 的 pmem-check 机制检测到并发送了 SIGTERM（kill）信号终止，最终导致作业出现崩溃。这个问题近期出现了好几次，客户希望能找到解决方案，避免国庆期间线上业务受到影响。

在 Flink 配置项中，提供了很多内存参数设定。我们逐一检查了客户作业的设置，发现最大值加起来也只有 3.75GB 左右（不含  JVM 自身 Native 内存区），离设定的 4.0G 阈值还有 256M 的空间。

用户作业并没有用到 RocksDB、GZip 等常见的需要使用 Native 内存且容易造成内存泄漏的第三方库，而且从 GC 日志来看，堆内各个区域远远没有用满，说明余量还是比较充足的。

那究竟是什么原因造成实际内存用量（RSS）超限了呢？

### Flink 内存模型

要分析问题，首先要了解 Flink 和 JVM 的内存模型。官方文档 \[1] 和很多第三方博客 \[2] \[3] 都对此有较为详尽的分析，这里只做流程的简单说明，不再详尽描述每个区域的具体计算过程。

下图展示了 Flink 内存各个区域的配置参数，其中左边是 Flink 配置项中的内存参数，中间是参数对应的内存区域，右边是这个作业配置的参数值。

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQXmrNVXDIbuZ2puBIg8tFDN4Vhc3kSL90nicacnib2NSDTFoXBlattIxZw/640?wx_fmt=png)

最上面深绿色的（taskmanager.memory.process.size）表示 JVM 所在容器的硬限制，例如 Kubernetes Pod YAML 的 resource limits。它的相关类为 ClusterSpecification，里面描述了 JobManager、TaskManager 容器所允许的最大内存用量，以及每个 TaskManager 的 Slot（运行槽）数等。

TaskManager 各个区域的内存用量是由 TaskExecutorProcessSpec 类来描述的。首先 Flink 的 ResourceManager 会调用 TaskExecutorFlinkMemoryUtils 工具类，从用户和系统的各项配置 Configuration 中获取各个内存区域的大小（ TaskExecutorFlinkMemory 对象，不含 Metaspace 和 Overhead 部分）。这中间要考虑到旧版本参数的兼容性，所以有很多绕来绕去的封装代码。总而言之，优先级是 新配置 > 旧配置 > 无配置（计算推导 + 默认值）。随后再根据配置和上述的计算结果，推导出 JvmMetaspaceAndOverhead，最终封装为包含各个区域内存大小定义的 TaskExecutorProcessSpec 对象。

最右边浅绿色文字的表示 Flink 内存参数最终翻译成的 JVM 参数（例如堆区域的 -Xmx、-Xms，Direct 内存区的 -XX:MaxDirectMemorySize 等），这个是 JVM 进程最终运行时的内存区域划分依据，是 ProcessMemoryUtils 这个工具类从上述的 TaskExecutorProcessSpec 对象中生成的。

### 堆内内存的分析

堆内内存（JVM Heap），指的是上图的 Framework Heap 和 Task Heap 部分。Task Heap 是 Flink 作业内存分配的重点区域，也是 JVM OutOfMemoryError: Java heap space 问题的发生地，当 OOM 问题发生时如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQX1NvoctllEyiaFGFdwIPpKJ1A3tgeZ48xn2JNqXGgW8LMSKatbuNiaISQ/640?wx_fmt=png)

如果这个区域内存占满了，也会出现不停的 GC，尤其是 Full GC。这些可以从监控指标面板看到，也可以通过 jstat 等命令查看。如果我们通过 Arthas、async-profiler \[4] 等工具对 JVM 进行运行时火焰图采样的话，也可以看到类似下面的结果：GC 相关的线程占了很大的时间片比例：

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQXB0qyayLvuJlfMvpeOHdujstf1Zr7Cnup1PY3JHjXCSibYuRWUSKygwQ/640?wx_fmt=png)

对于堆内内存的泄漏分析，如果进程即将崩溃但是还存活，可以使用 jmap 来获取一份堆内存的 dump：

    jmap -dump:live,format=b,file=/tmp/dump.hprof JVM进程PID   # 先做一次 Full GC 再 dumpjmap -dump:format=b,file=/tmp/dump.hprof JVM进程PID        # 直接进行 dump

如果进程崩溃难以捕捉，可以在 Flink 配置的 JVM 启动参数中增加：

    env.java.opts.taskmanager: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/taskmanager.hprof

这样 JVM 在发生 OOM 的时刻，会将堆内存 dump 保存到指定路径后再退出。

拿到堆内存 dump 文件以后，我们可以使用 MAT \[5] 这个开源的小工具来分析潜在的内存泄漏情况，并输出报表。

如果 MAT 不能满足需求，还有 JProfiler 等更全面的工具可以进行堆内存的高级分析。

当然，很不幸的是，这个出问题的作业的堆内存区域并没有用满，GC 日志看起来一切正常，堆内存泄漏的可能性排除。那么还需要进一步涉足堆外内存的各个神秘区域。

### 堆外内存的分析

JVM 堆外内存又分为多个区域，例如 Flink HybridMemorySegment 会用到 Java NIO 的 DirectByteBuffer 使用的 Direct 内存区（MaxDirectMemorySize 参数限制的区域），类加载等使用的 Metaspace 区（MaxMetaspaceSize 参数限制的区域，JDK 8 以前叫做 PermGen）。

如果 Direct 内存区发生了 OOM，JVM 会报出 OutOfMemoryError: Direct buffer memory 错误；而 Metaspace 区 OOM 则会报出 OutOfMemoryError: Metaspace 错误。但是这个作业日志中并没有看到任何 OutOfMemoryError 的错误，因此这些地方内存泄漏的可能性也不大。

使用 Native Memory Tracking 查看 JVM 的各个内存区域用量 JVM 自带了一个很有用的详细内存分配追踪工具：NMT \[6]，可以通过配置 JVM 启动参数来开启（可能造成 10% ~ 20% 的性能下降，线上慎用）：

    -XX:+UnlockDiagnosticVMOptions -XX:+PrintNMTStatistics -XX:NativeMemoryTracking=summary

随后可以对运行中的 JVM 进程执行：

    jcmd 进程 VM.native_memory summary

来获取此时此刻的 JVM 各区域的内存用量报表。

下面是一个典型的返回结果（// 为本文备注内容，标出了占用较多内存的区域含义）：

    Total: reserved=5249055KB, committed=3997707KB // 总物理内存申请量为 3.81G-                 Java Heap (reserved=3129344KB, committed=3129344KB)  // 堆内存占了 2.98G 物理内存                            (mmap: reserved=3129344KB, committed=3129344KB)-                     Class (reserved=1130076KB, committed=90824KB)  // 类的元数据占用了 88.7M 物理内存                            (classes #13501) // 加载的类数                            (malloc=1628KB #17097)                            (mmap: reserved=1128448KB, committed=89196KB)-                    Thread (reserved=136084KB, committed=136084KB)  // 线程栈占用了 132.9M 物理内存                            (thread #132)  // 线程数                            (stack: reserved=135504KB, committed=135504KB)                            (malloc=425KB #692)                            (arena=155KB #249)-                      Code (reserved=256605KB, committed=44513KB)                            (malloc=7005KB #11435)                            (mmap: reserved=249600KB, committed=37508KB)-                        GC (reserved=69038KB, committed=69038KB)                            (malloc=58846KB #618)                            (mmap: reserved=10192KB, committed=10192KB)-                  Compiler (reserved=394KB, committed=394KB)                            (malloc=263KB #811)                            (arena=131KB #18)-                  Internal (reserved=432708KB, committed=432704KB) // Direct 内存等部分占了 422.6M 物理内存                            (malloc=432672KB #31503)                            (mmap: reserved=36KB, committed=32KB)-                    Symbol (reserved=23801KB, committed=23801KB)                            (malloc=21875KB #165235)                            (arena=1926KB #1)-    Native Memory Tracking (reserved=3582KB, committed=3582KB)                            (malloc=20KB #226)                            (tracking overhead=3563KB)-               Arena Chunk (reserved=1542KB, committed=1542KB)                            (malloc=1542KB)-                   Unknown (reserved=65880KB, committed=65880KB)                            (mmap: reserved=65880KB, committed=65880KB)

可以看到，堆内存、Direct 等部分还是占了大部分，其他部分占用量相对较小。这个 JVM 总共统计到了 3.81G 的实时内存申请量。

但是，使用 top 命令查看这个 JVM 进程的实时用量时，发现 RSS（物理内存占用）已经升高到了 4.2G 左右，与上述结果不符，说明还是有部分内存没有追踪到：

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQXk2GUjkHN2VrA1iaibuZKaq3sJyRN4l0wnQF07rg2BblxubkWLbdjeNLA/640?wx_fmt=png)

使用 jemalloc 替代 ptmalloc 并统计内存动态分配 既然 JVM 自己统计的内存分配与实际占用仍然有较多偏差，而搜索了网上的各种资料时，经常会遇到因为 glibc malloc 64M 缓存造成内存超标的问题 \[7]。

由于 jemalloc 并没有这个 64M 的问题，而且可以通过 profiler 来统计 malloc 调用的动态分配情况，因此决定先使用 jemalloc \[8] 来替换 glibc 自带的分配函数，并进行统计。当然，使用 strace 等命令也可以拦截内存分配和释放情况（追踪 mmap、munmap、brk 等系统调用），不过结果太多了，分析起来并不方便。

下载解压 jemalloc 的发行包以后，进入相关目录，编译并安装它：

    ./configure --enable-prof --enable-stats --enable-debug --enable-fill && make && make install

随后在 Flink 参数里加上这些内容：

    containerized.taskmanager.env.LD_PRELOAD: "/usr/local/lib/libjemalloc.so.2"containerized.taskmanager.env.MALLOC_CONF: "prof:true,lg_prof_interval:29,lg_prof_sample:17"

重新运行作业，就可以不断地采集内存分配情况，并输出 .heap 文件到 JVM 进程的工作目录（例如 jeprof.951461.7.i7.heap）。

随后可以安装 graphviz，再使用 jemalloc 自带的 jeprof 命令对结果进行绘图（尽量在进程退出前绘图，避免地址无法解析）：

    yum install -y graphvizjeprof --svg `which java` 采集的.heap文件名 > ~/result.svg

结果如下：

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQXenznc9Ta3D1PeqbiaTN29S18pebibBwtLJ8xlLxoX6MVkztSBlTaWEzQ/640?wx_fmt=png)

从左边的分支来看，71.1% 的内存分配请求主要由  Unsafe.allocateMemory() 调用的（例如 Flink MemoryManager 分配的堆外 MemorySegments）。

中间分支的 init 是 JVM 启动期间分配的，也是正常范围。

右边分支主要是 JVM 内部的 ParNew & CMS GC、Class 解析所需的符号表、代码缓存所需的内存，也是正常的。因此并未观察到较大的第三方库造成的内存泄漏情况，因此间接引入第三方库造成内存泄漏的可能性也基本排除了。

### 使用 pmap 命令定期采样内存区块分配

既然 JVM NMT 上报的内存分区快照、jemalloc 统计的动态分配情况都没有找到准确的问题根源，我们还可以从底层出发，使用 pmap 命令来查看 JVM 进程的各个内存区域的分配情况，看是否有异常的条目。

可以使用下面的命令，从 Flink TaskManager 启动开始采样：

    while truedo        pmap -x JVM进程的PID > /tmp/pmap.`date +%Y-%m-%d-%H-%M-%S`.log        sleep 30sdone

随后可以使用文件比较工具，对比不同时间点的内存分配情况（例如下图是刚启动和崩溃前的最后一个记录），看是否有大块的不能解释的分配区段：

![](https://mmbiz.qpic.cn/mmbiz_png/1flHOHZw6RvyRsAKGibcTYicHw2Y8yAsQXhcuutYv97xLkRia0TiaQYcA8RhB91Lw14IAW2CY8w9fwIqReU1tZg1ew/640?wx_fmt=png)

上图中，除了堆内存区有大幅增长（只是稍微超出一些 Xmx 的限制），其他区域的增长都比较小，因此说明 JVM 内存超限基本上是因为堆内存区域随着使用自然扩展 + JVM 自身较大的 Overhead（内部所需内存）造成的。并且这部分内存在 NMT 报告里统计的并不准确，还需要进一步跟进。

初步总结 在上面的分析中，我们先从最容易分配也是占比最大的堆内存区域开始分析，逐步进入堆外内存的深水区。由于堆外内存除了 Java 自带的 NMT 机制外，并没有综合的分析工具可用，因此这里的分析过程往往繁杂而耗时，且较难得到准确原因。

本次问题的初步结论是 JVM 自身运行所需的内存（Overhead）占用较大，而用户对 Flink 的参数 taskmanager.memory.jvm-overhead.{min,fraction,max} 设定值过小（为了给堆内存留出更大空间，在这里只设置了 256MB 的阈值，而实际的内存占用不止这些）。

需要注意的是，这个参数并不意味着 Flink 能 “限制”JVM 内部的内存用量。相反，它的用途是令 Flink 在计算各区域（Heap、Off-Heap、Network 等）的内存空间时，能考虑到 JVM 这部分 Overhead 空间并不能被自己使用，应当减去这部分不受控的余量后再分配。

特别地，当用到了 RocksDB 等 JNI 调用的原生库时，请务必继续调大 taskmanager.memory.jvm-overhead.fraction 和 taskmanager.memory.jvm-overhead.max 参数的值（例如给到 1~2GB），避免余量不够而造成的总内存用量超标的问题。

### 参考阅读

\[1] Flink 官方文档 · 内存模型详解 [https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/ops/memory/mem_detail.html](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/ops/memory/mem_detail.html)

\[2] Flink 内存配置 [https://www.jianshu.com/p/a29b7b7feaaf](https://www.jianshu.com/p/a29b7b7feaaf)

\[3] Flink 内存设置思路 [https://www.cnblogs.com/lighten/p/13053828.html](https://www.cnblogs.com/lighten/p/13053828.html)

\[4] jvm-profiling-tools/async-profiler [https://github.com/jvm-profiling-tools/async-profiler](https://github.com/jvm-profiling-tools/async-profiler)

\[5] Memory Analyzer (MAT) [https://www.eclipse.org/mat/](https://www.eclipse.org/mat/)

\[6] Native Memory Tracking [https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html)

\[7] 疑案追踪：Spring Boot 内存泄露排查记 [https://mp.weixin.qq.com/s/aYwIH0TN3nSzNaMR2FN0AA](https://mp.weixin.qq.com/s/aYwIH0TN3nSzNaMR2FN0AA)

\[8] jemalloc [https://github.com/jemalloc/jemalloc/releases](https://github.com/jemalloc/jemalloc/releases) 
 [https://mp.weixin.qq.com/s/F0PdGjSXodWnCa88XMeQNQ](https://mp.weixin.qq.com/s/F0PdGjSXodWnCa88XMeQNQ)
