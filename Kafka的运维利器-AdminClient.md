# Kafka的运维利器-AdminClient
点击上方**蓝色字体**，选择 “设为星标”

回复”**面试**“获取更多惊喜

![](https://mmbiz.qpic.cn/mmbiz_png/sq2uE6cicHYxoldXibjHyWbvjJfI6ibEm5Kw715uVJTBLdX1gkVpExwlFh22TMnLIpBq96wT1ibdccSSd3LVdSE3LQ/640?wx_fmt=png)

## 前言

一般情况下，我们都习惯使用 kafka-topics.sh 脚本来管理主题，但有些时候我们希望将主题管理类的功能集成到公司内部的系统中，打造集管理、监控、运维、告警为一体的生态平台，那么就需要以程序调用 API 的方式去实现。

Kafka 社区于 0.11 版本正式推出了 Java 客户端版的 AdminClient，并不断地在后续的版本中对它进行完善。

本文主要介绍 KafkaAdminClient 的基本使用方式，以及采用这种调用 API 方式下的创建主题时的合法性验证。

## 功能

鉴于社区还在不断地完善 AdminClient 的功能，AdminClient 提供的功能有以下几个大类。

-   主题管理：包括主题的创建、删除和查询。
-   权限管理：包括具体权限的配置与删除。
-   配置参数管理：包括 Kafka 各种资源的参数设置、详情查询。所谓的 Kafka 资源，主要有 Broker、主题、用户、Client-id 等。
-   副本日志管理：包括副本底层日志路径的变更和详情查询。
-   分区管理：即创建额外的主题分区。
-   消息删除：即删除指定位移之前的分区消息。
-   Delegation Token 管理：包括 Delegation Token 的创建、更新、过期和详情查询。
-   消费者组管理：包括消费者组的查询、位移查询和删除。
-   Preferred 领导者选举：推选指定主题分区的 Preferred Broker 为领导者。

## 工作原理

AdminClient 是一个双线程的设计：前端主线程和后端 I/O 线程。

前端线程负责将用户要执行的操作转换成对应的请求，然后再将请求发送到后端 I/O 线程的队列中。

而后端 I/O 线程（kafka-admin-client-thread）从队列中读取相应的请求，然后发送到对应的 Broker 节点上，之后把执行结果保存起来，以便等待前端线程的获取。

## 使用

如果你使用的是 Maven，需要增加以下依赖项：

`<dependency>  
    <groupId>org.apache.kafka</groupId>  
    <artifactId>kafka-clients</artifactId>  
    <version>2.6.5</version>  
</dependency>  
`

### 构建 AdminClient

\`/\*\*  
 _ 创建 AdminClient 客户端对象  
 _/  
public static AdminClient createAdminClientByProperties() {

  Properties prop = new Properties();

  // 配置 Kafka 服务的访问地址及端口号  
  prop.setProperty(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG,"127.0.0.1:9092");

  // 创建 AdminClient 实例  
  return AdminClient.create(prop);  
}

/\*\*  
 _ 创建 AdminClient 的第二种方式  
 _/  
public static AdminClient createAdminClientByMap(){

  Map&lt;String, Object> conf = Maps.newHashMap();

  // 配置 Kafka 服务的访问地址及端口号  
  conf.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG,"127.0.0.1:9092");

  // 创建 AdminClient 实例  
  return AdminClient.create(conf);  
}

\`

### 创建 Topic 实例

\`private static final String TOPIC_NAME = "test_topic";

/\*\*  
 _ 创建 Topic 实例  
 _/  
public static void createTopic(){  
    AdminClient adminClient = AdminSample.adminClient();  
    // 副本因子  
    Short re = 1;  
    NewTopic newTopic = new NewTopic(TOPIC_NAME,1,re);  
    CreateTopicsResult createTopicsResult = adminClient.createTopics(Arrays.asList(newTopic));  
    System.out.println("CreateTopicsResult :" + createTopicsResult);  
    adminClient.close();  
}

\`

### 查询 Topic 列表

\`private static final String TOPIC_NAME = "test_topic";

/\*\*  
 _ 获取 topic 列表  
 _/  
public static void topicList() throws Exception {  
    AdminClient adminClient = adminClient();

    // 是否查看 Internal 选项  
    ListTopicsOptions options = new ListTopicsOptions();  
    options.listInternal(true);

    //ListTopicsResult listTopicsResult = adminClient.listTopics();  
    ListTopicsResult listTopicsResult = adminClient.listTopics(options);  
    Set<String> names = listTopicsResult.names().get();

    // 打印 names  
    names.stream().forEach(System.out::println);

    Collection<TopicListing> topicListings = listTopicsResult.listings().get();  
    // 打印 TopicListing  
    topicListings.stream().forEach((topicList) -> {  
        System.out.println(topicList.toString());  
    });  
    adminClient.close();  
}

\`

### 删除 topic

\`private static final String TOPIC_NAME = "test_topic";

/\*\*  
 _ 删除 topic  
 _/  
public static void delTopic() throws Exception {  
    AdminClient adminClient = adminClient();  
    DeleteTopicsResult deleteTopicsResult = adminClient.deleteTopics(Arrays.asList(TOPIC_NAME));  
    deleteTopicsResult.all().get();  
}

\`

### 描述 topic

\`/\*\*  
 _ 获取 topic 的描述信息  
 _/  
public static void describeTopics(List<String> topics) throws Exception {  
    // 创建 AdminClient 客户端对象  
    AdminClient adminClient = BuildAdminClient.createAdminClientByProperties();

    // 获取 Topic 的描述信息  
    DescribeTopicsResult result = adminClient.describeTopics(topics);

    // 解析描述信息结果, Map&lt;String, TopicDescription> ==> topicName:topicDescription  
    Map&lt;String, TopicDescription> topicDescriptionMap = result.all().get();  
    topicDescriptionMap.forEach((topicName, description) -> System.out.printf("topic name = %s, desc = %s \\n", topicName, description));

    // 关闭资源  
    adminClient.close();  
}

\`

### 查看 Topic 的配置信息

除了 Kafka 自身的配置项外，其内部的 Topic 也会有非常多的配置项，我们可以通过 describeConfigs 方法来获取某个 Topic 中的配置项信息。代码示例：

\`/\*\*  
 _ 获取 topic 的配置信息  
 _/  
public static void describeConfigTopics(List<String> topicNames) throws Exception {  
    // 创建 AdminClient 客户端对象  
    AdminClient adminClient = BuildAdminClient.createAdminClientByMap();

    List<ConfigResource> configResources = Lists.newArrayListWithCapacity(64);  
    topicNames.forEach(topicName -> configResources.add(  
            // 指定要获取的源  
            new ConfigResource(ConfigResource.Type.TOPIC, topicName)));

    // 获取 topic 的配置信息  
    DescribeConfigsResult result = adminClient.describeConfigs(configResources);

    // 解析 topic 的配置信息  
    Map&lt;ConfigResource, Config> resourceConfigMap = result.all().get();  
    resourceConfigMap.forEach((configResource, config) -> System.out.printf("topic config ConfigResource = %s, Config = %s \\n", configResource, config));

    // 关闭资源  
    adminClient.close();  
}

\`

### 修改 Topic 的分区数量

在创建 Topic 时我们需要设定 Partition 的数量，但如果觉得初始设置的 Partition 数量太少了，那么就可以使用 createPartitions 方法来调整 Topic 的 Partition 数量，但是需要注意在 Kafka 中 Partition 只能增加不能减少。代码示例：

\`/\*\*  
 _ 修改 topic 的分区数量  
 _ 只能增加不能减少  
 \*/  
public static void updateTopicPartition(List<String> topicNames, Integer partitionNum) throws Exception {  
    // 创建 AdminClient 客户端对象  
    AdminClient adminClient = BuildAdminClient.createAdminClientByMap();

    // 构建修改分区的 topic 请求参数  
    Map&lt;String, NewPartitions> newPartitions = Maps.newHashMap();  
    topicNames.forEach(topicName -> newPartitions.put(topicName, NewPartitions.increaseTo(partitionNum)));

    // 执行修改  
    CreatePartitionsResult result = adminClient.createPartitions(newPartitions);

    // get 方法是一个阻塞方法，一定要等到 createPartitions 完成之后才进行下一步操作  
    result.all().get();

    // 关闭资源  
    adminClient.close();  
}

\`

社区于 0.11 版本正式推出了 Java 客户端版的 AdminClient 工具，该工具提供了几十种运维操作，而且它还在不断地演进着。如果可以的话，你最好统一使用 AdminClient 来执行各种 Kafka 集群管理操作，摒弃掉连接 ZooKeeper 的那些工具。另外，建议时刻关注该工具的功能完善情况，毕竟，目前社区对 AdminClient 的变更频率很高。

![](https://mmbiz.qpic.cn/mmbiz_jpg/BSBqCXrZtzAicMToibKuIysLrB62M5A5YaLhZg6z86tI7ZeEZqTLLYyNrmlzrkyKUN5kNeUFicVC3bMP1GEqKz1OQ/640?wx_fmt=jpeg)

[八千里路云和月 | 从零到大数据专家学习路径指南](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504680&idx=1&sn=9f2547b40316b8dc8a748bbc8b6b6015&chksm=fd3e95bdca491cab9e0badbd7a1144f7511638ae80db684fd5383c771129a16bb08ba5e72515&scene=21#wechat_redirect)

[我们在学习 Flink 的时候，到底在学习什么？](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247499604&idx=1&sn=d938dfb30d221774704982d2938b30c1&chksm=fd3eb9c1ca4930d76a391241333de461ca22d2aa27472eab3cffab1564872ae37f1b48fe2d3c&scene=21#wechat_redirect)

[193 篇文章暴揍 Flink，这个合集你需要关注一下](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504856&idx=2&sn=6f62e2a0c756ce56773eed253d2f41ee&chksm=fd3e954dca491c5b732b7e2aa46db32efbc3f690e528522098659bb3c6e78e1582ab4529b5dc&scene=21#wechat_redirect)

[Flink 生产环境 TOP 难题与优化，阿里巴巴藏经阁 YYDS](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504742&idx=1&sn=8765c198d8ad66219a7bcabb221e4a23&chksm=fd3e95f3ca491ce52d0724b9e4154a47af0f5ea349e1e184c8fbc65aa17c0000242aa9b58429&scene=21#wechat_redirect)

[Flink CDC 我吃定了耶稣也留不住他！| Flink CDC 线上问题小盘点](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504813&idx=1&sn=f5cd6ae2aa2b1e30f87a5ae55971c514&chksm=fd3e9538ca491c2eb191677d070f2c7e4f1098eece00e256d6205b8a5d21b21c1e74f875f16f&scene=21#wechat_redirect)

[我们在学习 Spark 的时候，到底在学习什么？](http://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247492567&idx=1&sn=1f693e549f76622725b936041ff8896e&chksm=e9bff2fbdec87bedecbdeed4547d7d612c72fc8d4918614a26a4e31666cf0128fd57b537f347&scene=21#wechat_redirect)

[在所有 Spark 模块中，我愿称 SparkSQL 为最强！](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504834&idx=1&sn=46ca1a3924b8fdb89ac1c2acb0319d6c&chksm=fd3e9557ca491c41d837b917639ea62007ea16d3c2cc6c0c02d1f8a8c6e44318f5183b787c46&scene=21#wechat_redirect)

[硬刚 Hive | 4 万字基础调优面试小总结](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247502750&idx=1&sn=bd9a9173d060dc4e4ebd49c8efc6acfe&chksm=fd3e8d0bca49041dea84da93910e5efdc4935e520525c09887c986691377aeb48e5cf7fb5667&scene=21#wechat_redirect)

[数据治理方法论和实践小百科全书](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504382&idx=2&sn=550b78802acfe727e0e77cd9195f8784&chksm=fd3e976bca491e7db2b8b2446d231736df01bbf13653804d680ac4390597ec17fa466ad4ae83&scene=21#wechat_redirect)  

[标签体系下的用户画像建设小指南](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503741&idx=1&sn=e5039be93123f2e337013756a818bfc3&chksm=fd3e89e8ca4900fe603b63c5722a6fb8a32bd63d6ba23e0028851948a71b877eb1f742d95087&scene=21#wechat_redirect)  

[4 万字长文 | ClickHouse 基础 & 实践 & 调优全视角解析](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247503675&idx=1&sn=3ee6af64d0126c78b48cad219308f81e&chksm=fd3e89aeca4900b8b8954e9569ee3c0877881fac8c792bfafc22e7e9d3e8524da8eb860d33d8&scene=21#wechat_redirect)  

[【面试 & 个人成长】2021 年过半，社招和校招的经验之谈](http://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&mid=2247492567&idx=2&sn=57ecc77718f1b6f2f262d62e8318dcc9&chksm=e9bff2fbdec87bedb1986765bfae7dbabb9aece45b0b2af147bbad1ac5d79ebc934c64df40ea&scene=21#wechat_redirect)

[大数据方向另一个十年开启 |《硬刚系列》第一版完结](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504478&idx=1&sn=14efef3868ba42bd044f9618745a7fdc&chksm=fd3e94cbca491dddb961b7b8b93105b2869c5bcf4c03f9f0dc83ad62be0dce4c124b52ed0ba3&scene=21#wechat_redirect)

[我写过的关于成长 / 面试 / 职场进阶的文章](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504410&idx=1&sn=7e81ab5a324395eb0f12397c40247ca1&chksm=fd3e948fca491d9946456acbd93b2d651ae4d7a7d0127a211981e08f50f377e60d1b7c0d7b3a&scene=21#wechat_redirect)

[当我们在学习 Hive 的时候在学习什么？「硬刚 Hive 续集」](http://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247504783&idx=1&sn=72aed147a459368ed934a007a2df3bc9&chksm=fd3e951aca491c0cade212390011eca7d8a68951fee6aa2844b8f4d14bcbd5b3ed26305d69dc&scene=21#wechat_redirect) 
 [https://mp.weixin.qq.com/s/vchFozDTO9GMiP6Ug7YSjQ](https://mp.weixin.qq.com/s/vchFozDTO9GMiP6Ug7YSjQ)
