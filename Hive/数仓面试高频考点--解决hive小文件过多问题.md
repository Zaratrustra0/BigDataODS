# 数仓面试高频考点--解决hive小文件过多问题
五分钟学大数据，致力于大数据技术研究，如果你有任何问题或建议，可添加底部小编微信或直接后台留言

### 小文件产生原因

hive 中的小文件肯定是向 hive 表中导入数据时产生，所以先看下向 hive 中导入数据的几种方式

1.  直接向表中插入数据


    insert into table A values (1,'zhangsan',88),(2,'lisi',61);

这种方式每次插入时都会产生一个文件，多次插入少量数据就会出现多个小文件，但是这种方式生产环境很少使用，可以说基本没有使用的

2.  通过 load 方式加载数据


    load data local inpath '/export/score.csv' overwrite into table A  -- 导入文件load data local inpath '/export/score' overwrite into table A   -- 导入文件夹

使用 load 方式可以导入文件或文件夹，当导入一个文件时，hive 表就有一个文件，当导入文件夹时，hive 表的文件数量为文件夹下所有文件的数量

3.  通过查询方式加载数据


    insert overwrite table A  select s_id,c_name,s_score from B;

这种方式是生产环境中常用的，也是最容易产生小文件的方式

insert 导入数据时会启动 MR 任务，MR 中 reduce 有多少个就输出多少个文件

所以， 文件数量 = ReduceTask 数量 \* 分区数

也有很多简单任务没有 reduce，只有 map 阶段，则

文件数量 = MapTask 数量 \* 分区数

每执行一次 insert 时 hive 中至少产生一个文件，因为 insert 导入时至少会有一个 MapTask。  
像有的业务需要每 10 分钟就要把数据同步到 hive 中，这样产生的文件就会很多。

### 小文件过多产生的影响

1.  首先对底层存储 HDFS 来说，HDFS 本身就不适合存储大量小文件，小文件过多会导致 namenode 元数据特别大, 占用太多内存，严重影响 HDFS 的性能
2.  对 hive 来说，在进行查询时，每个小文件都会当成一个块，启动一个 Map 任务来完成，而一个 Map 任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的 Map 数量是受限的。

### 怎么解决小文件过多

#### 1. 使用 hive 自带的 concatenate 命令，自动合并小文件

使用方法：

    #对于非分区表alter table A concatenate;#对于分区表alter table B partition(day=20201224) concatenate;

举例：

    #向 A 表中插入数据hive (default)> insert into table A values (1,'aa',67),(2,'bb',87);hive (default)> insert into table A values (3,'cc',67),(4,'dd',87);hive (default)> insert into table A values (5,'ee',67),(6,'ff',87);#执行以上三条语句，则A表下就会有三个小文件,在hive命令行执行如下语句#查看A表下文件数量hive (default)> dfs -ls /user/hive/warehouse/A;Found 3 items-rwxr-xr-x   3 root supergroup        378 2020-12-24 14:46 /user/hive/warehouse/A/000000_0-rwxr-xr-x   3 root supergroup        378 2020-12-24 14:47 /user/hive/warehouse/A/000000_0_copy_1-rwxr-xr-x   3 root supergroup        378 2020-12-24 14:48 /user/hive/warehouse/A/000000_0_copy_2#可以看到有三个小文件，然后使用 concatenate 进行合并hive (default)> alter table A concatenate;#再次查看A表下文件数量hive (default)> dfs -ls /user/hive/warehouse/A;Found 1 items-rwxr-xr-x   3 root supergroup        778 2020-12-24 14:59 /user/hive/warehouse/A/000000_0#已合并成一个文件

> 注意：   
> 1、concatenate 命令只支持 RCFILE 和 ORC 文件类型。   
> 2、使用 concatenate 命令合并小文件时不能指定合并后的文件数量，但可以多次执行该命令。   
> 3、当多次使用 concatenate 后文件数量不在变化，这个跟参数 mapreduce.input.fileinputformat.split.minsize=256mb 的设置有关，可设定每个文件的最小 size。

#### 2. 调整参数减少 Map 数量

-   **设置 map 输入合并小文件的相关参数**：


    #执行Map前进行小文件合并#CombineHiveInputFormat底层是 Hadoop的 CombineFileInputFormat 方法#此方法是在mapper中将多个文件合成一个split作为输入set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; -- 默认#每个Map最大输入大小(这个值决定了合并后文件的数量)set mapred.max.split.size=256000000;   -- 256M#一个节点上split的至少的大小(这个值决定了多个DataNode上的文件是否需要合并)set mapred.min.split.size.per.node=100000000;  -- 100M#一个交换机下split的至少的大小(这个值决定了多个交换机上的文件是否需要合并)set mapred.min.split.size.per.rack=100000000;  -- 100M

-   **设置 map 输出和 reduce 输出进行合并的相关参数**:


    #设置map端输出进行合并，默认为trueset hive.merge.mapfiles = true;#设置reduce端输出进行合并，默认为falseset hive.merge.mapredfiles = true;#设置合并文件的大小set hive.merge.size.per.task = 256*1000*1000;   -- 256M#当输出文件的平均大小小于该值时，启动一个独立的MapReduce任务进行文件mergeset hive.merge.smallfiles.avgsize=16000000;   -- 16M 

-   **启用压缩**


    # hive的查询结果输出是否进行压缩set hive.exec.compress.output=true;# MapReduce Job的结果输出是否使用压缩set mapreduce.output.fileoutputformat.compress=true;

#### 3. 减少 Reduce 的数量

    #reduce 的个数决定了输出的文件的个数，所以可以调整reduce的个数控制hive表的文件数量，#hive中的分区函数 distribute by 正好是控制MR中partition分区的，#然后通过设置reduce的数量，结合分区函数让数据均衡的进入每个reduce即可。#设置reduce的数量有两种方式，第一种是直接设置reduce个数set mapreduce.job.reduces=10;#第二种是设置每个reduce的大小，Hive会根据数据总大小猜测确定一个reduce个数set hive.exec.reducers.bytes.per.reducer=5120000000; -- 默认是1G，设置为5G#执行以下语句，将数据均衡的分配到reduce中set mapreduce.job.reduces=10;insert overwrite table A partition(dt)select * from Bdistribute by rand();解释：如设置reduce数量为10，则使用 rand()， 随机生成一个数 x % 10 ，这样数据就会随机进入 reduce 中，防止出现有的文件过大或过小

#### 4. 使用 hadoop 的 archive 将小文件归档

Hadoop Archive 简称 HAR，是一个高效地将小文件放入 HDFS 块中的文件存档工具，它能够将多个小文件打包成一个 HAR 文件，这样在减少 namenode 内存使用的同时，仍然允许对文件进行透明的访问

    #用来控制归档是否可用set hive.archive.enabled=true;#通知Hive在创建归档时是否可以设置父目录set hive.archive.har.parentdir.settable=true;#控制需要归档文件的大小set har.partfile.size=1099511627776;#使用以下命令进行归档ALTER TABLE A ARCHIVE PARTITION(dt='2020-12-24', hr='12');#对已归档的分区恢复为原文件ALTER TABLE A UNARCHIVE PARTITION(dt='2020-12-24', hr='12');

> 注意:    
> **归档的分区可以查看不能 insert overwrite，必须先 unarchive**

#### 最后

如果是新集群，没有历史遗留问题的话，建议 hive 使用 orc 文件格式，以及启用 lzo 压缩。  
这样小文件过多可以使用 hive 自带命令 concatenate 快速合并。

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

文: 园陌

扫码收获更多技术

![](https://mmbiz.qpic.cn/mmbiz_gif/c6gqmhWiafyobvdkCN1GNh88geK5jbWxJ16lNcEJicicNgoRJDre75yTz7LicJBqaZMmNrojibsmOxWcia5Dk7qKC0eg/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zFXFP8Nqo5j6gk7icneiciaibL6PKich0rZzoNRPrlEJGZCqUyC8Uia5GLY9CJj97iaSphibTruU79jxE9uLQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_gif/TN05MmJLxMrNiboff6rhmPUYUNXdWTXZfCTse4EDYOEoRVrlpgoZF6t00NhnrdePKAibI1zaap7wibR0iclbJZFS1A/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/OJhVkq8ONwq-v-okcMKo-w](https://mp.weixin.qq.com/s/OJhVkq8ONwq-v-okcMKo-w)
