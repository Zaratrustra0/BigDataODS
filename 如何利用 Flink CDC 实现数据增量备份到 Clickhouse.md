# 如何利用 Flink CDC 实现数据增量备份到 Clickhouse
![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcfg3df4PNapwOxjhCD1t73xxuKCfKFVld3Bs722HibrjsrN9RicAkeLibg/640?wx_fmt=png)

挖了很久的 CDC 坑，今天打算填一填了。本文我们首先来介绍什么是 CDC，以及 CDC 工具选型，接下来我们来介绍如何通过 Flink CDC 抓取 mysql 中的数据，并把他汇入 Clickhouse 里，最后我们还将介绍 Flink SQL CDC 的方式。

首先什么是 CDC ？它是 Change Data Capture 的缩写, 即变更数据捕捉的简称，使用 CDC 我们可以从数据库中获取已提交的更改并将这些更改发送到下游，供下游使用。这些变更可以包括 INSERT,DELETE,UPDATE 等操作。

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcfxQ2VYayibticMadBpaOkiciaESdVibbjVLwWtK1qYPgm6kIfmJUribucwyw/640?wx_fmt=png)

其主要的应用场景：

-   异构数据库之间的数据同步或备份 / 建立数据分析计算平台
-   微服务之间共享数据状态
-   更新缓存 / CQRS 的 Query 视图更新

CDC 它是一个比较广义的概念，只要能捕获变更的数据，我们都可以称为 CDC 。业界主要有基于查询的 CDC 和基于日志的 CDC ，可以从下面表格对比他们功能和差异点。

\|  
 | 基于查询的 CDC | 基于日志的 CDC |
\| --- \| --- \| --- \|
| 概念 | 每次捕获变更发起 Select 查询进行全表扫描，过滤出查询之间变更的数据 | 读取数据存储系统的 log ，例如 MySQL 里面的 binlog 持续监控 |
| 开源产品 | Sqoop, Kafka JDBC Source | Canal, Maxwell, Debezium |
| 执行模式 | Batch | Streaming |
| 捕获所有数据的变化 | ❌ | ✅ |
| 低延迟，不增加数据库负载 | ❌ | ✅ |
| 不侵入业务（LastUpdated 字段） | ❌ | ✅ |
| 捕获删除事件和旧记录的状态 | ❌ | ✅ |
| 捕获旧记录的状态 | ❌ | ✅ |

Debezium 是一个开源项目，为捕获数据更改 (change data capture,CDC) 提供了一个低延迟的流式处理平台。你可以安装并且配置 Debezium 去监控你的数据库，然后你的应用就可以消费对数据库的每一个行级别 (row-level) 的更改。只有已提交的更改才是可见的，所以你的应用不用担心事务 (transaction) 或者更改被回滚(roll back)。Debezium 为所有的数据库更改事件提供了一个统一的模型，所以你的应用不用担心每一种数据库管理系统的错综复杂性。另外，由于 Debezium 用持久化的、有副本备份的日志来记录数据库数据变化的历史，因此，你的应用可以随时停止再重启，而不会错过它停止运行时发生的事件，保证了所有的事件都能被正确地、完全地处理掉。

> Debezium is an open source distributed platform for change data capture. Start it up, point it at your databases, and your apps can start responding to all of the inserts, updates, and deletes that other apps commit to your databases. Debezium is durable and fast, so your apps can respond quickly and never miss an event, even when things go wrong

Why debezium?  这里就放一张和网易大佬的聊天截图，说明吧

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcAEAlS45FZODav6VMliazOAdQTnzGl8qsBG40R17qf5XY8t2tdbYneLQ/640?wx_fmt=png)

**_鸣谢，简佬，同意出镜_**  

实时数据分析数据库，俄罗斯的谷歌开发的，推荐 OLAP 场景使用

**Clickhouse 的优点.**

1.  真正的面向列的 DBMS

    ClickHouse 是一个 DBMS，而不是一个单一的数据库。它允许在运行时创建表和数据库、加载数据和运行

    查询，而无需重新配置和重新启动服务器。
2.  数据压缩

    一些面向列的 DBMS（InfiniDB CE 和 MonetDB）不使用数据压缩。但是，数据压缩确实提高了性能。
3.  磁盘存储的数据
4.  在多个服务器上分布式处理
5.  SQL 支持
6.  数据不仅按列存储，而且由矢量 - 列的部分进行处理，这使开发者能够实现高 CPU 性能

**Clickhouse 的缺点**

1.  没有完整的事务支持，
2.  缺少完整的 Update/Delete 操作，缺少高频率、低延迟的修改或删除已存在数据的能力，仅能用于批量删

    除或修改数据
3.  聚合结果必须小于一台机器的内存大小：
4.  不适合 key-value 存储，

**什么时候不可以用 Clickhouse?**

1.  事物性工作（OLTP）
2.  高并发的键值访问
3.  Blob 或者文档存储
4.  超标准化的数据

Flink cdc connector 消费 Debezium 里的数据，经过处理再 sink 出来，这个流程还是相对比较简单的

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcVCgAHpA3z2xKIQvTT2IozrQ7M7zhNl2FXKRia5XWtACHe00TI8klXoQ/640?wx_fmt=png)

首先创建 Source 和 Sink（对应的依赖引用，在文末）  

 SourceFunction<String> sourceFunction = MySQLSource.<String>builder()  
 .hostname("localhost")  
 .port(3306)  
 .databaseList("test")  
 .username("flinkcdc")  
 .password("dafei1288")  
 .deserializer(new JsonDebeziumDeserializationSchema())  
 .build();  

 // 添加 source  
 env.addSource(sourceFunction)  
 // 添加 sink  
 .addSink(new ClickhouseSink());

这里用到的 JsonDebeziumDeserializationSchema，是我们自定义的一个序列化类，用于将 Debezium 输出的数据，序列化

// 将 cdc 数据反序列化  
 public static class JsonDebeziumDeserializationSchema implements DebeziumDeserializationSchema {  
 @Override  
 public void deserialize(SourceRecord sourceRecord, Collector collector) throws Exception {  

 Gson jsstr = new Gson();  
 HashMap&lt;String, Object> hs = new HashMap&lt;>();  

 String topic = sourceRecord.topic();  
 String\[] split = topic.split("\[.]");  
 String database = split\[1];  
 String table = split\[2];  
 hs.put("database",database);  
 hs.put("table",table);  
 // 获取操作类型  
 Envelope.Operation operation = Envelope.operationFor(sourceRecord);  
 // 获取数据本身  
 Struct struct = (Struct)sourceRecord.value();  
 Struct after = struct.getStruct("after");  

 if (after != null) {  
 Schema schema = after.schema();  
 HashMap&lt;String, Object> afhs = new HashMap&lt;>();  
 for (Field field : schema.fields()) {  
 afhs.put(field.name(), after.get(field.name()));  
 }  
 hs.put("data",afhs);  
 }  

 String type = operation.toString().toLowerCase();  
 if ("create".equals(type)) {  
 type = "insert";  
 }  
 hs.put("type",type);  

 collector.collect(jsstr.toJson(hs));  
 }  

 @Override  
 public TypeInformation<String> getProducedType() {  
 return BasicTypeInfo.STRING_TYPE_INFO;  
 }  
 }

这里是将数据序列化成如下 Json 格式

{"database":"test","data":{"name":"jacky","description":"fffff","id":8},"type":"insert","table":"test_cdc"}

接下来就是要创建 Sink，将数据变化存入 Clickhouse 中，这里我们仅以 insert 为例

public static class ClickhouseSink extends RichSinkFunction<String>{  
 Connection connection;  
 PreparedStatement pstmt;  
 private Connection getConnection() {  
 Connection conn = null;  
 try {  
 Class.forName("ru.yandex.clickhouse.ClickHouseDriver");  
 String url = "jdbc:clickhouse://localhost:8123/default";  
 conn = DriverManager.getConnection(url,"default","dafei1288");  

 } catch (Exception e) {  
 e.printStackTrace();  
 }  
 return conn;  
 }  

 @Override  
 public void open(Configuration parameters) throws Exception {  
 super.open(parameters);  
 connection = getConnection();  
 String sql = "insert into sink_ch_test(id,name,description) values (?,?,?)";  
 pstmt = connection.prepareStatement(sql);  
 }  

 // 每条记录插入时调用一次  
 public void invoke(String value, Context context) throws Exception {  
 //{"database":"test","data":{"name":"jacky","description":"fffff","id":8},"type":"insert","table":"test_cdc"}  
 Gson t = new Gson();  
 HashMap&lt;String,Object> hs = t.fromJson(value,HashMap.class);  
 String database = (String)hs.get("database");  
 String table = (String)hs.get("table");  
 String type = (String)hs.get("type");  

 if("test".equals(database) && "test_cdc".equals(table)){  
 if("insert".equals(type)){  
 System.out.println("insert =>"+value);  
 LinkedTreeMap&lt;String,Object> data = (LinkedTreeMap&lt;String,Object>)hs.get("data");  
 String name = (String)data.get("name");  
 String description = (String)data.get("description");  
 Double id = (Double)data.get("id");  
 // 未前面的占位符赋值  
 pstmt.setInt(1, id.intValue());  
 pstmt.setString(2, name);  
 pstmt.setString(3, description);  

 pstmt.executeUpdate();  
 }  
 }  
 }  

 @Override  
 public void close() throws Exception {  
 super.close();  

 if(pstmt != null) {  
 pstmt.close();  
 }  

 if(connection != null) {  
 connection.close();  
 }  
 }  
 }

完整代码案例：

package name.lijiaqi.cdc;  

import com.alibaba.ververica.cdc.debezium.DebeziumDeserializationSchema;  
import com.google.gson.Gson;  
import com.google.gson.internal.LinkedTreeMap;  
import io.debezium.data.Envelope;  
import org.apache.flink.api.common.typeinfo.BasicTypeInfo;  
import org.apache.flink.api.common.typeinfo.TypeInformation;  
import org.apache.flink.configuration.Configuration;  
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;  
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;  
import org.apache.flink.streaming.api.functions.source.SourceFunction;  
import com.alibaba.ververica.cdc.connectors.mysql.MySQLSource;  
import org.apache.flink.util.Collector;  
import org.apache.kafka.connect.source.SourceRecord;  

import org.apache.kafka.connect.data.Field;  
import org.apache.kafka.connect.data.Schema;  
import org.apache.kafka.connect.data.Struct;  

import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.PreparedStatement;  
import java.util.HashMap;  

public class MySqlBinlogSourceExample {  
 public static void main(String\[] args) throws Exception {  
 SourceFunction<String> sourceFunction = MySQLSource.<String>builder()  
 .hostname("localhost")  
 .port(3306)  
 .databaseList("test")  
 .username("flinkcdc")  
 .password("dafei1288")  
 .deserializer(new JsonDebeziumDeserializationSchema())  
 .build();  

 StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  

 // 添加 source  
 env.addSource(sourceFunction)  
 // 添加 sink  
 .addSink(new ClickhouseSink());  

 env.execute("mysql2clickhouse");  
 }  

 // 将 cdc 数据反序列化  
 public static class JsonDebeziumDeserializationSchema implements DebeziumDeserializationSchema {  
 @Override  
 public void deserialize(SourceRecord sourceRecord, Collector collector) throws Exception {  

 Gson jsstr = new Gson();  
 HashMap&lt;String, Object> hs = new HashMap&lt;>();  

 String topic = sourceRecord.topic();  
 String\[] split = topic.split("\[.]");  
 String database = split\[1];  
 String table = split\[2];  
 hs.put("database",database);  
 hs.put("table",table);  
 // 获取操作类型  
 Envelope.Operation operation = Envelope.operationFor(sourceRecord);  
 // 获取数据本身  
 Struct struct = (Struct)sourceRecord.value();  
 Struct after = struct.getStruct("after");  

 if (after != null) {  
 Schema schema = after.schema();  
 HashMap&lt;String, Object> afhs = new HashMap&lt;>();  
 for (Field field : schema.fields()) {  
 afhs.put(field.name(), after.get(field.name()));  
 }  
 hs.put("data",afhs);  
 }  

 String type = operation.toString().toLowerCase();  
 if ("create".equals(type)) {  
 type = "insert";  
 }  
 hs.put("type",type);  

 collector.collect(jsstr.toJson(hs));  
 }  

 @Override  
 public TypeInformation<String> getProducedType() {  
 return BasicTypeInfo.STRING_TYPE_INFO;  
 }  
 }  

 public static class ClickhouseSink extends RichSinkFunction<String>{  
 Connection connection;  
 PreparedStatement pstmt;  
 private Connection getConnection() {  
 Connection conn = null;  
 try {  
 Class.forName("ru.yandex.clickhouse.ClickHouseDriver");  
 String url = "jdbc:clickhouse://localhost:8123/default";  
 conn = DriverManager.getConnection(url,"default","dafei1288");  

 } catch (Exception e) {  
 e.printStackTrace();  
 }  
 return conn;  
 }  

 @Override  
 public void open(Configuration parameters) throws Exception {  
 super.open(parameters);  
 connection = getConnection();  
 String sql = "insert into sink_ch_test(id,name,description) values (?,?,?)";  
 pstmt = connection.prepareStatement(sql);  
 }  

 // 每条记录插入时调用一次  
 public void invoke(String value, Context context) throws Exception {  
 //{"database":"test","data":{"name":"jacky","description":"fffff","id":8},"type":"insert","table":"test_cdc"}  
 Gson t = new Gson();  
 HashMap&lt;String,Object> hs = t.fromJson(value,HashMap.class);  
 String database = (String)hs.get("database");  
 String table = (String)hs.get("table");  
 String type = (String)hs.get("type");  

 if("test".equals(database) && "test_cdc".equals(table)){  
 if("insert".equals(type)){  
 System.out.println("insert =>"+value);  
 LinkedTreeMap&lt;String,Object> data = (LinkedTreeMap&lt;String,Object>)hs.get("data");  
 String name = (String)data.get("name");  
 String description = (String)data.get("description");  
 Double id = (Double)data.get("id");  
 // 未前面的占位符赋值  
 pstmt.setInt(1, id.intValue());  
 pstmt.setString(2, name);  
 pstmt.setString(3, description);  

 pstmt.executeUpdate();  
 }  
 }  
 }  

 @Override  
 public void close() throws Exception {  
 super.close();  

 if(pstmt != null) {  
 pstmt.close();  
 }  

 if(connection != null) {  
 connection.close();  
 }  
 }  
 }  
}

执行查看结果

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcpgsAqKUcENpA1ZJBUs3uO0icvPAIZUZk1bfU3rNIRIPzBCWD6FL8Lzw/640?wx_fmt=png)

数据成功汇入

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOciaFTHYl1vOkhbtx0QfPFVaAicmXsDVxZ0hsX8QJtl2ABolO7bmibgWpZg/640?wx_fmt=png)

接下来，我们看一下如何通过 Flink SQL 实现 CDC ，只需 3 条 SQL 语句即可。

创建数据源表

 // 数据源表  
 String sourceDDL =  
 "CREATE TABLE mysql_binlog (\\n" +  
 " id INT NOT NULL,\\n" +  
 " name STRING,\\n" +  
 " description STRING\\n" +  
 ") WITH (\\n" +  
 "'connector' = 'mysql-cdc',\\n" +  
 "'hostname' = 'localhost',\\n" +  
 "'port' = '3306',\\n" +  
 "'username' = 'flinkcdc',\\n" +  
 "'password' = 'dafei1288',\\n" +  
 "'database-name' = 'test',\\n" +  
 "'table-name' = 'test_cdc'\\n" +  
 ")";

创建输出表

 // 输出目标表  
 String sinkDDL =  
 "CREATE TABLE test_cdc_sink (\\n" +  
 " id INT NOT NULL,\\n" +  
 " name STRING,\\n" +  
 " description STRING,\\n" +  
 " PRIMARY KEY (id) NOT ENFORCED \\n " +  
 ") WITH (\\n" +  
 "'connector' = 'jdbc',\\n" +  
 "'driver' = 'com.mysql.jdbc.Driver',\\n" +  
 "'url' = '"+ url +"',\\n" +  
 "'username' = '"+ userName +"',\\n" +  
 "'password' = '"+ password +"',\\n" +  
 "'table-name' = '"+ mysqlSinkTable +"'\\n" +  
 ")";

这里我们直接将数据汇入

// 简单的聚合处理  
 String transformSQL =  
 "insert into test_cdc_sink select \* from mysql_binlog";

完整参考代码

package name.lijiaqi.cdc;  

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;  
import org.apache.flink.table.api.EnvironmentSettings;  
import org.apache.flink.table.api.SqlDialect;  
import org.apache.flink.table.api.TableResult;  
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;  

public class MysqlToMysqlMain {  
 public static void main(String\[] args) throws Exception {  
 EnvironmentSettings fsSettings = EnvironmentSettings.newInstance()  
 .useBlinkPlanner()  
 .inStreamingMode()  
 .build();  
 StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
 env.setParallelism(1);  
 StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, fsSettings);  

 tableEnv.getConfig().setSqlDialect(SqlDialect.DEFAULT);  

 // 数据源表  
 String sourceDDL =  
 "CREATE TABLE mysql_binlog (\\n" +  
 " id INT NOT NULL,\\n" +  
 " name STRING,\\n" +  
 " description STRING\\n" +  
 ") WITH (\\n" +  
 "'connector' = 'mysql-cdc',\\n" +  
 "'hostname' = 'localhost',\\n" +  
 "'port' = '3306',\\n" +  
 "'username' = 'flinkcdc',\\n" +  
 "'password' = 'dafei1288',\\n" +  
 "'database-name' = 'test',\\n" +  
 "'table-name' = 'test_cdc'\\n" +  
 ")";  

 String url = "jdbc:mysql://127.0.0.1:3306/test";  
 String userName = "root";  
 String password = "dafei1288";  
 String mysqlSinkTable = "test_cdc_sink";  
 // 输出目标表  
 String sinkDDL =  
 "CREATE TABLE test_cdc_sink (\\n" +  
 " id INT NOT NULL,\\n" +  
 " name STRING,\\n" +  
 " description STRING,\\n" +  
 " PRIMARY KEY (id) NOT ENFORCED \\n " +  
 ") WITH (\\n" +  
 "'connector' = 'jdbc',\\n" +  
 "'driver' = 'com.mysql.jdbc.Driver',\\n" +  
 "'url' = '"+ url +"',\\n" +  
 "'username' = '"+ userName +"',\\n" +  
 "'password' = '"+ password +"',\\n" +  
 "'table-name' = '"+ mysqlSinkTable +"'\\n" +  
 ")";  
 // 简单的聚合处理  
 String transformSQL =  
 "insert into test_cdc_sink select \* from mysql_binlog";  

 tableEnv.executeSql(sourceDDL);  
 tableEnv.executeSql(sinkDDL);  
 TableResult result = tableEnv.executeSql(transformSQL);  

 // 等待 flink-cdc 完成快照  
 result.print();  
 env.execute("sync-flink-cdc");  
 }  

}

查看执行结果

![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagibib6zAzLyTNZa5TCIZn3mOcw3dKBbKwc8ZA73s8s30tBiaibsM9iaeKGWicicgj8qicbicdTz1UpsHQWJVjA/640?wx_fmt=png)

添加依赖

 <dependencies>  
 <!\-\- https://mvnrepository.com/artifact/org.apache.flink/flink-core -->  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-core</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-streaming-java_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
  
<!\-\-        <dependency>-->  
<!\-\-            <groupId>org.apache.flink</groupId>-->  
<!\-\-            <artifactId>flink-jdbc_2.12</artifactId>-->  
<!\-\-            <version>1.10.3</version>-->  
<!\-\-        </dependency>-->  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-connector-jdbc_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-java</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-clients_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-table-api-java-bridge_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-table-common</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-table-planner_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-table-planner-blink_2.12</artifactId>  
 <version>1.13.0</version>  
 </dependency>  
 <dependency>  
 <groupId>org.apache.flink</groupId>  
 <artifactId>flink-table-planner-blink_2.12</artifactId>  
 <version>1.13.0</version>  
 <type>test-jar</type>  
 </dependency>  
  
 <dependency>  
 <groupId>com.alibaba.ververica</groupId>  
 <artifactId>flink-connector-mysql-cdc</artifactId>  
 <version>1.4.0</version>  
 </dependency>  
  
  
 <dependency>  
 <groupId>com.aliyun</groupId>  
 <artifactId>flink-connector-clickhouse</artifactId>  
 <version>1.12.0</version>  
 </dependency>  
 <dependency>  
 <groupId>ru.yandex.clickhouse</groupId>  
 <artifactId>clickhouse-jdbc</artifactId>  
 <version>0.2.6</version>  
 </dependency>  
 <dependency>  
 <groupId>com.google.code.gson</groupId>  
 <artifactId>gson</artifactId>  
 <version>2.8.6</version>  
 </dependency>  
 </dependencies>

参考链接：

[https://blog.csdn.net/zhangjun5965/article/details/107605396](https://blog.csdn.net/zhangjun5965/article/details/107605396)

[https://cloud.tencent.com/developer/article/1745233?from=article.detail.1747773](https://cloud.tencent.com/developer/article/1745233?from=article.detail.1747773)

[https://segmentfault.com/a/1190000039662261](https://segmentfault.com/a/1190000039662261)

[https://www.cnblogs.com/weijiqian/p/13994870.html](https://www.cnblogs.com/weijiqian/p/13994870.html)

关注 【麒思妙想】解锁更多硬核。

**历史文章导读**：  

-   [抽象语法树为什么抽象](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247485009&idx=1&sn=8c489fd8a2dbab1c1a61d315f25a345d&chksm=eacfa713ddb82e050e34c62093a66a518c6dad85b6fcad7172b931ba153ab1f666b3e6e6b48a&scene=21#wechat_redirect)  
-   [基于 Calcite 自定义 SQL 解析器](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484462&idx=1&sn=797a6e5bb250566c06da7a4036757427&chksm=eacfa56cddb82c7a26e9bd21479b49f8788fe6655c6ea10bb260181d17e0f9836d4952f22815&scene=21#wechat_redirect)  
-   [基于 JDBC 实现 VPD：SQL 解析篇](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484584&idx=1&sn=c8c968799bb01fec4102a9c2ab234839&chksm=eacfa5eaddb82cfc2b79a6535a689210846e764273caa443da1a04c534ea0d9c3c1c68146300&scene=21#wechat_redirect)  
-   [如何成为一个成功的首席数据官](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484854&idx=1&sn=6f0cf1741f6e90157de9ada408e3087f&chksm=eacfa4f4ddb82de2e912635d76fb17d9878a9621ee9819b28753392c3e7f489719c19f0e7016&scene=21#wechat_redirect)  
-   [数据库深度研究（100 页 PPT）](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484986&idx=1&sn=84b56c8946024c8b01e77eda7742e989&chksm=eacfa778ddb82e6e29dc7171dd2caea8403706d3240735bd1ce9264098d90f1898bea601a396&scene=21#wechat_redirect)  
-   [基于 Win10 单机部署 kubernetes 应用](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484674&idx=1&sn=6bb43c185b79388fc455bdc1e281e8a2&chksm=eacfa440ddb82d56617bba7e0931bea369da0a1959fe998ec80f35c68fdcc735a26b354668fd&scene=21#wechat_redirect)  
-   [浅谈基于 JDBC 实现虚拟专用数据库（VPD）](http://mp.weixin.qq.com/s?__biz=MzI3MDU3OTc1Nw==&mid=2247484563&idx=1&sn=9d17a488ddc11bc76aa0eb3fb1652ec1&chksm=eacfa5d1ddb82cc703861f6193c1aeaf3825ae6c9d91fd8ce5e6aff3754e1c140cea16cc1be2&scene=21#wechat_redirect)  

如果文章对您有那么一点点帮助，我将倍感荣幸

欢迎  **关注、在看、点赞、转发**  

![](https://mmbiz.qpic.cn/mmbiz_jpg/8uJ0ic6nAag8EItgzqIEhe3GbK3ibibrSC3kGNLaCYoEXEEEV8vatdHqibkazrs7oLJERAG1cldW9pbVmcTKvXL3fA/640?wx_fmt=jpeg) 
 [https://mp.weixin.qq.com/s/0wbG-u964UVAubKnzkqtxw](https://mp.weixin.qq.com/s/0wbG-u964UVAubKnzkqtxw)
