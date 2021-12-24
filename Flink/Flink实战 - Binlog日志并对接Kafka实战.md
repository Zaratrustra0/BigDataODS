# Flink实战 - Binlog日志并对接Kafka实战
点击上方**蓝色字体**，选择 “**设为星标**”

回复” 资源 “获取更多资源

![](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg)

[![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2OflopvKZ9wvmiaF2Mm2eU12FNITkgCB4EJmYLnYicozYViaHmuU4U0kaO6nYRIAwy5hpBqNnbqiaxHyQ/640?wx_fmt=jpeg)
](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

[**大数据技术与架构**](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

点击右侧关注，大数据开发领域最强公众号！

[![](https://mmbiz.qpic.cn/mmbiz_jpg/UdK9ByfMT2MSGY5RTojm5sPV2pSicy3oK14Micdx2DKIZ4BIg0U5ic0nA0IhQ07XHQB52oOvibaI1OibicKUOYFgL77g/640?wx_fmt=jpeg)
](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247486054&idx=3&sn=b0aae1d5212d401d980066500c20872b&chksm=fd3d4cf3ca4ac5e5e415ec8b057c6c393972cae6f105d5f94364efbde6d21012c0286318dfee&token=1387203581&lang=zh_CN&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M4nqUxqq6rrK75dxa4sSvxqS5oDP92jCAkZ4kgMpcIpibLw1n89xYsthOdYdzolntbSadIyekJ0uw/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)

**[大数据](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)真好玩**

点击右侧关注，大数据真好玩！

[![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2PicAKtwvjRxPMic7Bia3foLlzB01ia6f1SITvsCNveIlJXLGzvb5icuQ1nBpcmqH68AAUO3owU01Z9dlA/640?wx_fmt=png)
](https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247484145&idx=8&sn=834b97ecf6a5889a57fcd16010b9ce0e&chksm=e9bc13dddecb9acb949f61a30185fbbd050e49c13a11ec35df9f062f3cae92753a95d3b795f9&token=224321657&lang=zh_CN&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10icrrCFShT3Z5lylZy797kQfk42lZOjRtg3UJ5lV1L6KsYR8Vv5Nr2uQ/640?wx_fmt=png)

对于 Flink 数据流的处理，一般都是去直接监控 xxx.log 日志的数据，至于如何实现关系型数据库数据的同步的话网上基本没啥多少可用性的文章，基于项目的需求，经过一段时间的研究终于还是弄出来了，写这篇文章主要是以中介的方式记录下来，也希望能帮助到在做关系型数据库的实时计算处理流的初学者。

#### 一、设计流程图

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10N59oUzsicjV2x9TDPcVhkBRKWqIMm0vpLeOKPYHWDZ2SkWEibYtlfGfA/640?wx_fmt=png)

#### 二、MySQL 的 Binlog 日志的设置

找到 MySQL 的配置文件并编辑：

    [root@localhost etc]# vim /etc/my.cnf[mysqld]# 其它配置省略。。。。。。lower_case_table_names=1## Replicationserver_id                       =2020041006     # 唯一log_bin                         =mysql-bin-1       # 唯一relay_log_recovery              =1binlog_format                   =row   # 格式必须是 row，否则 ogg 监控有问题master_info_repository          =TABLErelay_log_info_repository       =TABLE#rpl_semi_sync_master_enabled    =1rpl_semi_sync_master_timeout    =1000rpl_semi_sync_slave_enabled     =1binlog-do-db                    =dsout    # 要生成binlog 的数据库sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

这里注意的是配置完 my.cnf 文件之后要重启 MySQL 服务器才能生效。查看配置的 状态 和 serverid 命令请参见这篇文章：

#### 三、下载 OGG 并安装部署

下载地址：[https://www.oracle.com/middleware/technologies/goldengate-downloads.html](https://www.oracle.com/middleware/technologies/goldengate-downloads.html)

1\. 下载下来的压缩包解压并放入指定的文件夹中去

    mkdir -p /opt/module/ogg/oggservicetar -xvf ggs_Linux_x64_MySQL_64bit.tar -C /opt/module/ogg/oggservice/chown -R root:root oggservice/     # 授权成指定的用户及用户组

2\. 进入 ogg 并启动

    cd oggservice/./ggsci

3\. 源系统的操作步骤及配置信息如下：

    GGSCI (localhost.localdomain) 1> create subdirs   # 创建目录GGSCI (localhost.localdomain) 3> dblogin sourcedb dsout@192.168.x.xxx:3306,userid 用户,password 密码;   # 监控日志GGSCI (localhost.localdomain) 3> edit params mgrport 7015AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3GGSCI (localhost.localdomain) 4> edit params ext1   # 抽取进程EXTRACT ext1setenv (MYSQL_HOME="/usr/local/mysql")dboptions host 192.168.x.xx:3306, connectionport 3306tranlogoptions altlogdest /usr/local/mysql/data/mysql-bin-1.indexSOURCEDB dsout@192.168.x.xx:3306,userid 用户,password 密码EXTTRAIL ./dirdat/etdynamicresolutionGETUPDATEBEFORESNOCOMPRESSDELETESNOCOMPRESSUPDATEStable dsout.employees;table dsout.departments;GGSCI (localhost.localdomain) 5> edit params pump1  # 推送进程EXTRACT pump1SOURCEDB dsout@192.168.x.xx:3306,userid 用户,password 密码RMTHOST 目标服务器的IP地址, MGRPORT 2021RMTTRAIL ./dirdat/xdtable dsout.*;    # 要推送的表#为数据库的binlog添加监控和推送进程GGSCI (localhost.localdomain DBLOGIN as dsout) 8> add extract ext1, tranlog,begin nowGGSCI (localhost.localdomain DBLOGIN as dsout) 9> add exttrail ./dirdat/et, extract ext1GGSCI (localhost.localdomain DBLOGIN as dsout) 10> add extract pump1, exttrailsource ./dirdat/etGGSCI (localhost.localdomain DBLOGIN as dsout) 11> add rmttrail ./dirdat/rt,extract pump1# 配置 defgen 进程GGSCI (localhost.localdomain) 6> edit param defgendefsfile ./dirdef/defgen.defsourcedb dsout@192.168.x.xx:3306,userid 用户,password 密码table dsout.*;# 生成 defgen.prm 文件[mysql@localhost oggformysql]$ ./defgen paramfile ./dirprm/defgen.prm

4\. 进入 ogg 查看各个配置的服务进程：

    GGSCI (localhost.localdomain) 5> info all

效果图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10BzElFpn4FzAicShpGnIyP0BegZoe3BFfpRYGaibaiaibZ0OKc6tVLh5sag/640?wx_fmt=png)

到此为止源系统的 ogg 已经配置完成，接下来我们要在目标端配置接收到的数据将其以 json 的形式发送到 kafka。

5\. 解压并授权

    mkdir -p /opt/module/ogg/oggservicetar -xvf OGG_BigData_Linux_x64_19.1.0.0.1.tar -C /opt/module/ogg/oggservice/chown -R root:root oggservice/

6\. 配置依赖包

    find / -name libjvm.sovim ~/.bash_profileexport LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64/jre/lib/amd64/server/source ~/.bash_profile

7\. 启动并配置相关进程

    cd oggservice/./ggsci GGSCI (cdh102) 1> create subdirs GGSCI (cdh102) 1> edit param mgr   # 配置主进程PORT 2021ACCESSRULE, PROG *, IPADDR *, ALLOWGGSCI (cdh102) 2> edit param rep2  # 配置复制进程replicat rep2sourcedefs ./dirdef/defgen.defTARGETDB LIBFILE libggjava.so SET property=./dirprm/kafkaxd.propsMAP dsout.*, TARGET dsout.*;# (注意，这里的exttrail必须和源端的dump配置一致）GGSCI (cdh102) 5> add replicat rep2, exttrail ./dirdat/rt

8\. 创建对接 kafka 的配置文件

    cd ./dirprm[root@cdh102 dirprm]# vim kafkaxd.props   # -> 配置文件内容如下gg.handlerlist = kafkahandlergg.handler.kafkahandler.type=kafkagg.handler.kafkahandler.KafkaProducerConfigFile=xindai_kafka_producer.properties   # kafka 生产者属性文件#######The following resolves the topic name using the short table namegg.handler.kafkahandler.topicMappingTemplate=xindai-topic   # 主题############The following selects the message key using the concatenated primary keys############gg.handler.kafkahandler.keyMappingTemplate=${primaryKeys}###########gg.handler.kafkahandler.format=avro_opgg.handler.kafkahandler.SchemaTopicName=xindai-topic    # 主题gg.handler.kafkahandler.BlockingSend =falsegg.handler.kafkahandler.includeTokens=falsegg.handler.kafkahandler.mode=opgg.handler.kafkahandler.format=json#########gg.handler.kafkahandler.format.insertOpKey=I#######gg.handler.kafkahandler.format.updateOpKey=U#########gg.handler.kafkahandler.format.deleteOpKey=D#######gg.handler.kafkahandler.format.truncateOpKey=Tgoldengate.userexit.writers=javawriterjavawriter.stats.display=TRUEjavawriter.stats.full=TRUEgg.log=log4jgg.log.level=INFOgg.report.time=30sec##########Sample gg.classpath for Apache Kafka  这里一定要指定kafka依赖包的路径gg.classpath=dirprm/:/opt/cloudera/parcels/CDH-6.3.1-1.cdh6.3.1.p0.1470567/lib/kafka/libs/*##########Sample gg.classpath for HDP#########gg.classpath=/etc/kafka/conf:/usr/hdp/current/kafka-broker/libs/* javawriter.bootoptions=-Xmx512m -Xms32m -Djava.class.path=ggjava/ggjava.jar

9\. 配置 KafkaProducerConfigFile 属性文件

    [root@cdh102 dirprm]# vim xindai_kafka_producer.propertiesbootstrap.servers=cdh101:9092,cdh102:9092,cdh103:9092acks=1reconnect.backoff.ms=1000value.serializer=org.apache.kafka.common.serialization.ByteArraySerializerkey.serializer=org.apache.kafka.common.serialization.ByteArraySerializer######## 100KB per partitionbatch.size=16384linger.ms=0key.converter=org.apache.kafka.connect.json.JsonConvertervalue.converter=org.apache.kafka.connect.json.JsonConverterkey.converter.schemas.enable=falsevalue.converter.schemas.enable=false

10\. 启动进程

    # 目标端GGSCI (gpdata) 6> start mgrGGSCI (gpdata) 7> start rep2# 源端GGSCI (localhost.localdomain) 1> start mgrGGSCI (localhost.localdomain) 2> start ext1GGSCI (localhost.localdomain) 3> start pump1   # (先起目标的 mgr 才不会报错)

效果图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl102wwPOO4YPvs88cg77H3sMDJkV9xgnh78wJjywzNVomOZV4mibByp2uw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10gqZYT0pd22eJhtCKe5pV9DXbGsL4FTZlHzibsmeGxYpr9Rz5nzxYHPw/640?wx_fmt=png)

#### 四、验证

1\. 启动 kafka 消费者

    [root@cdh102 kafka]# bin/kafka-console-consumer.sh --bootstrap-server cdh101:9092,cdh102:9092,cdh103:9092 --topic xindai-topic --from-beginning

2\. 向库的监控表中对数据进行增、删、改 操作

    INSERT INTO employees VALUES('101','changyin',6666.66,'2020-05-05 16:12:20','syy01');INSERT INTO employees VALUES('102','siling',1234.12,'2020-05-05 16:12:20','syy01');

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10SiaWlM5rUYLickHLtgAntysfKGJuC8dSLztSqBFSQB8SWyVicc3qrAwVA/640?wx_fmt=png)

3\. 查看 Kafka 消费者的数据

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2NFibhZwoNyj3dJPCaVycl10xX9pGiaQhQiaw8iaIjjDhGEbAUlB5Jt88YlXnbrSgt46f8zyUEsiatS7AQ/640?wx_fmt=png)

到此，我们已经成功的配置好了 使用 Ogg 监控 MySQL - Binlog 日志，然后将数据以 Json 的形式传给 Kafka 的消费者的整个流程；这是项目实践中总结出来的，为了方便以后查询，在此做了下记录，希望也能帮到志同道合的同学们。

![](https://mmbiz.qpic.cn/mmbiz_gif/UdK9ByfMT2O2R2sWRGyQHreSAGqmpRwtrmHudAsc2zib2AiatXJ6goxkskB4JEFrld8ia4WYOCsVdVVmr5hgfDdpg/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/UdK9ByfMT2O2R2sWRGyQHreSAGqmpRwtNhh4iaJx03bCpDFmIGtWfNCxp2qoM4UlGsfKsnYfajRKa5feialNm5uw/640?wx_fmt=gif)

**版权声明：** 

本文为大数据技术与架构整理，原作者独家授权。未经原作者允许转载追究侵权责任。

**微信公众号｜import_bigdata**

**编辑** 《大数据技术与架构》

**插画**《大数据技术与架构》

**文章链接** [https://www.jianshu.com/u/b14730fd40bd](https://www.jianshu.com/u/b14730fd40bd)

**欢迎点赞 + 收藏 + 转发朋友圈素质三连**

\***\* 文章不错？\*\***点个【在看】吧！**\*\***👇**\*\*** 
 [https://mp.weixin.qq.com/s/uMxDaDDz7El3yY2tEsmccA](https://mp.weixin.qq.com/s/uMxDaDDz7El3yY2tEsmccA)
