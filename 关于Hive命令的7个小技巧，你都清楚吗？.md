# 关于Hive命令的7个小技巧，你都清楚吗？
关注上方 “Python 数据科学”，选择星标，

关键时间，第一时间送达！

[☞500g + 超全学习资源免费领取](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247493636&idx=4&sn=2de693fd3948498b19368b08cd24aeb5&chksm=fad79f09cda0161f671aef9c2964856e5a5ba7afea474b4d9371bee46f8d81e6312567ecabe1&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/NOM5HN2icXzxJPiaqLtEqcPdnod10rXQ2pT7JyF9g5LicdGs5jqzNiaPFUAuTvQcmg0dzdzib552oQXLqQDeTFCNNxQ/640?wx_fmt=png)

## 前言

最近在看冰河大佬写的《海量数据处理与大数据技术实战》，该书涵盖以 Hadoop 为主的多款大数据技术框架实战的内容，兼顾理论与实操，是市面上难得的技术好书。本篇文章，我就分享一下从中学习到的关于**Hive 命令的 7 个小技巧**，受益的朋友记得来发三连⭐支持一下哟~

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnibPiapyxcibBibUzlNKkmRAwU4fviarG3PVORkic49MNYY3qOYMia4NbVibpXQ/640?wx_fmt=png)

## Hive 命令说明

在 Hive 提供的所有连接方式中，命令行界面是最常用的一种方式。用户可以使用 Hive 的命令行对 Hive 中的数据库、数据表和数据进行各种操作。

### 1、Hive 命令选项

在服务器上启动 Hadoop 之后，输入 “Hive” 命令就能够进入 Hive 的命令行。也可以输入如下命令查看 Hive 的命令选项：

hive --help

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnpujJrtE6NX9Xq3KLqSl7sEicnqD9dSIUlqricnA7ljBtENoAnHDaqjIw/640?wx_fmt=png)

可以看到，输出了 Hive 的一些命令选项，说明用户可以通过`--service serviceName`的方式启动某个服务。以下信息列出了 Hive 主要的命令行选项：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnCDt5F8k8ViaCtVRpB77SqtTWJYaMPSKnnccJMUmqejbp1fpZOsu37Vg/640?wx_fmt=png)

其中，部分重要选项的说明如下：  

（1） cli：命令行界面

（2）hiveserver2：启动 Hive 远程模式时需要启动的服务，其可以监听来自其他进程的连接

（3）jar：扩展自 hadoop jar 命令，可以执行需要 Hive 环境的应用程序

（4）metastore：启动一个 Hive 元数据服务

接下来，在 CentOS6.9 服务器的命令行中输入如下命令，查看 Hive 的 CLI 选项：

hive --help --service cli

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnU58NfzhfNF8fIyhK8ppjaUzf3IfticlGBAcXF1AY8GMxKdUJpVToCsQ/640?wx_fmt=png)

选项说明如下：

 (1）-d，--define&lt;key=value>：主要用来定义变量，如 -d A=B 或者 --define A=B

(2) --databases: 指定使用的数据库名称

(3) -e：从服务器命令行执行 SQL 语句

(4) -f：从文件中执行 SQL 语句

(5) -H:--help ：输出帮助信息

(6) --hiveconf&lt;property=value>: 设置 Hive 的属性值，能够覆盖 hive-site.xml 文件中配置的属性值

(7) --hivevar&lt;key=value>：在 Hive 命令中替换参数

(8) -i：初始化 SQL 文件

(9) -S，-- silent：集成模式下开启静默模式

(10) -v，-- verbose：输出详细信息

2、Hive 命令的使用

在命令行输入 “hive” 命令，即可进入 Hive 命令行终端，如下所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnvc0eks6hSYj4JNYAFVFVpmwQ9NHsKoTvm35e1MtGTgpdgZq34wgdlg/640?wx_fmt=png)
我们写个查询语句

hive (default)> select \* from testdb.student;  
OK  
student.s_id    student.s_name  student.s_birth student.s_sex  
01      永昌    1990-01-01      男  
02      鸿哲    1990-12-21      男  
03      文景    1990-05-20      男  
04      李云    1990-08-06      男  
05      妙之    1991-12-01      女  
06      雪卉    1992-03-01      女  
07      秋香    1989-07-01      女  
08      王丽    1990-01-20      女  
Time taken: 1.197 seconds, Fetched: 8 row(s)  

很多时候，执行一条查询语句并不需要打开命令行界面。此时可以使用 “hive -e” 形式的命令，如下所示：

\[root@node01 hive-1.1.0-cdh5.14.0]# hive -e "select count(\*) from testdb.student"  
Logging initialized using configuration in jar:file:/export/servers/hive-1.1.0-cdh5.14.0/lib/hive-common-1.1.0-cdh5.14.0.jar!/hive-log4j.properties  
Query ID = root_20201108231818_becc7952-05a5-49fc-915d-b6648d429f08  
Total jobs = 1  
Launching Job 1 out of 1  
Number of reduce tasks determined at compile time: 1  
In order to change the average load for a reducer (in bytes):  
  set hive.exec.reducers.bytes.per.reducer=<number>  
In order to limit the maximum number of reducers:  
  set hive.exec.reducers.max=<number>  
In order to set a constant number of reducers:  
  set mapreduce.job.reduces=<number>  
Starting Job = job_1604845856822_0001, Tracking URL = [http://node01:8088/proxy/application\\\_1604845856822\\\_0001/](http://node01:8088/proxy/application\_1604845856822\_0001/)  
Kill Command = /export/servers/hadoop-2.6.0-cdh5.14.0/bin/hadoop job  -kill job_1604845856822_0001  
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1  
2020-11-08 23:18:36,501 Stage-1 map = 0%,  reduce = 0%  
2020-11-08 23:18:37,649 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.88 sec  
2020-11-08 23:18:38,739 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 2.18 sec  
MapReduce Total cumulative CPU time: 2 seconds 180 msec  
Ended Job = job_1604845856822_0001  
MapReduce Jobs Launched:  
Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 2.18 sec   HDFS Read: 11544 HDFS Write: 580032 SUCCESS  
Total MapReduce CPU Time Spent: 2 seconds 180 msec  
OK  
\_c0  
8  

 如果不需要输出过多的日志信息，则可以在 hive 后面加 -S 选项，如下所示：

\[root@node01 hive-1.1.0-cdh5.14.0]# hive -S -e "select count(\*) from testdb.student"  
\_c0  
8  

如果需要一次性执行多条语句，可以将多条语句保存到以 .hql 结尾的文件中，如下所示：

vim test.sql  
select count(_) from testdb.student;  
select _ from testdb.student;  

使用`hive -f`命令来执行 hql 文件中的语句，如下所示：

\[root@node01 tmpfile]# hive -f test.sql  
\_c0  
8  
Time taken: 11.551 seconds, Fetched: 1 row(s)  
OK  
student.s_id    student.s_name  student.s_birth student.s_sex  
01      永昌    1990-01-01      男  
02      鸿哲    1990-12-21      男  
03      文景    1990-05-20      男  
04      李云    1990-08-06      男  
05      妙之    1991-12-01      女  
06      雪卉    1992-03-01      女  
07      秋香    1989-07-01      女  
08      王丽    1990-01-20      女  
Time taken: 0.071 seconds, Fetched: 8 row(s)

 可以用 “--” 添加注释，如下所示：

hive (default)> select \* from testdb.student  -- 测试查询数据;  
OK  
student.s_id    student.s_name  student.s_birth student.s_sex  
01      永昌    1990-01-01      男  
02      鸿哲    1990-12-21      男  
03      文景    1990-05-20      男  
04      李云    1990-08-06      男  
05      妙之    1991-12-01      女  
06      雪卉    1992-03-01      女  
07      秋香    1989-07-01      女  
08      王丽    1990-01-20      女  
Time taken: 0.073 seconds, Fetched: 8 row(s)  

### 3、hiverc 文件

**在 ${HIVE_HOME}/bin 目录下有个. hiverc 文件，它是隐藏文件，我们可以用 Linux 的 ls -a 命令查看。我们在启动 Hive 的时候会去加载这个文件中的内容**，所以我们可以在这个文件中配置一些常用的参数，如下：

cd /export/servers/hive-1.1.0-cdh5.14.0/bin  
vim .hiverc

select \* from testdb.student;

set hive.cli.print.current.db=true;  
#查询出来的结果显示列的名称  
set hive.cli.print.header=true;  
#启用桶表  
set hive.enforce.bucketing=true;  
#压缩 hive 的中间结果  
set hive.exec.compress.intermediate=true;  
#对 map 端输出的内容使用 BZip2 编码 / 解码器  
set mapred.map.output.compression.codec=org.apache.hadoop.io.compress.BZip2Codec;  
#压缩 hive 的输出  
set hive.exec.compress.output=true;  
#对 hive 中的 MR 输出内容使用 BZip2 编码 / 解码器  
set mapred.output.compression.codec=org.apache.hadoop.io.compress.BZip2Codec;  
#让 hive 尽量尝试 local 模式查询而不是 mapred 方式  
set hive.exec.mode.local.auto=true;

例如，我们这里在`.hiverc`文件中添加了一句 HQL 查询语句，那么每次我们启动 hive 命令行，都会自动去执行这个命令。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnl32MQ91nFvn0tX9ccspDx73Mw0yCibxOjhbKiaChoibv7rXe5S3konTPQ/640?wx_fmt=png)
可以看到，输入 “hive” 命令启动 Hive 命令行，自动执行了 .hiverc 文件中的语句。

当我们需要频繁使用某些命令时，就可以将这些命令保存在`.hiverc`文件中。

### 4、Hive 操作命令历史

Hive 将最近执行的 10000 条命令记录到当前用户的 home 目录下的`.hivehistory`文件中，用户可以输入如下命令查看这个文件 (当前用户为 root)：

cat vim /root/.hivehistory

![](https://mmbiz.qpic.cn/sz_mmbiz_png/BFibC8wtke3uCaStGLDWfVrVVdWEloiavnLNRcxoJ3pugx5CT4lT6pfwTsibfb1l6Y7fuAV34icotbJ5tHtrERGMcg/640?wx_fmt=png)

### 5、在 Hive 命令行执行系统命令

在 Hive 命令行下执行操作系统命令非常简单，只需要在执行的系统命令前加上 “!”，并以“；” 结尾即可，如下所示：

hive (default)> !echo "hello world";  
"hello world"  
hive (default)> !jps;  
11985 ResourceManager  
18308 Jps  
12420 RunJar  
12085 NodeManager  
12519 RunJar  
11545 NameNode  
18138 RunJar  
11837 SecondaryNameNode  
11646 DataNode  

### 6、在 Hive 命令行执行 Hadoop 命令

在 Hive 命令行可以执行 Hadoop 命令，只需要将 Hadoop 命令中的关键字 hdfs 去掉，最后添加一个 “；” 即可。

例如，在操作系统命令行执行 Hadoop 的如下命令：

\[root@node01 ~]# hdfs dfs -ls /  
Found 3 items  
drwxr-xr-x   - root supergroup          0 2020-01-03 02:28 /aa  
drwxr-xr-x   - PC   supergroup          0 2020-03-30 10:33 /aaa  
drwxr-xr-x   - root supergroup          0 2019-12-27 05:42 /abc  

 在 Hive 命令行执行 Hadoop 命令，如下所示：

hive (default) > dfs -ls /;  
Found 3 items  
drwxr-xr-x   - root supergroup          0 2020-01-03 02:28 /aa  
drwxr-xr-x   - PC   supergroup          0 2020-03-30 10:33 /aaa  
drwxr-xr-x   - root supergroup          0 2019-12-27 05:42 /abc  

可以看到，得出的结果是一致的。

7、在 Hive 命令行显示查询字段名

使用 Hive 命令查询数据时，可以显示查询数据的字段名称，此时需要将 Hive 的 `hive.cli.print.header` 属性设置为 `true`，默认为 false，如下所示：

hive (default)> set hive.cli.print.header=true;  
hive (default)> select \* from testdb.student;  
OK  
student.s_id    student.s_name  student.s_birth student.s_sex  
01      永昌    1990-01-01      男  
02      鸿哲    1990-12-21      男  
03      文景    1990-05-20      男  
04      李云    1990-08-06      男  
05      妙之    1991-12-01      女  
06      雪卉    1992-03-01      女  
07      秋香    1989-07-01      女  
08      王丽    1990-01-20      女  
Time taken: 0.056 seconds, Fetched: 8 row(s)  

## 小结

本篇文章主要分享了关于 Hive 的 7 个小技巧，后边会持续为大家分享优质有趣的内容，受益或感兴趣的朋友记得三连支持一下呀~**你知道的越多，你不知道的就越多**！

回复「Java 学习」获得 1024G Java 学习资料

回复「Python 学习」获得 100G Python 学习资料

![](https://mmbiz.qpic.cn/mmbiz_jpg/Iefry9dPrYLcoBANuN5iaIXEzuOiaiaJswqke19Stic8AQwKqOKRBQPiaSEtl6XibnduR4JibeCIiciahh8dloZXHsWvotw/640?wx_fmt=jpeg)

🧐**分享、点赞、在看**，给个**三连击**呗！**👇** 
 [https://mp.weixin.qq.com/s/Z6BvKNV_mchxR8b0wwtV6Q](https://mp.weixin.qq.com/s/Z6BvKNV_mchxR8b0wwtV6Q)
