# 你要悄悄学会HBase，然后惊艳所有人(1.7万字建议收藏)
**1**

**HBase 简介**

* * *

       HBASE 是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用 HBASE 技术可在廉价 PC Server 上搭建起大规模结构化存储集群。  

       HBASE 的目标是存储并处理大型的数据，更具体来说是仅需使用普通的硬件配置，就能够处理由成千上万的行和列所组成的大型数据。

       HBASE 是 Google Bigtable 的开源实现，但是也有很多不同之处。比如：Google Bigtable 利用 GFS 作为其文件存储系统，HBASE 利用 Hadoop HDFS 作为其文件存储系统；Google 运行 MAPREDUCE 来处理 Bigtable 中的海量数据，HBASE 同样利用 Hadoop MapReduce 来处理 HBASE 中的海量数据；Google Bigtable 利用 Chubby 作为协同服务，HBASE 利用 Zookeeper 作为对应。

      它介于 nosql 和 RDBMS 之间，仅能通过主键 (row key) 和主键的 range 来检索数据，仅支持单行事务(可通过 hive 支持来实现多表 join 等复杂操作)。主要用来存储非结构化和半结构化的松散数据。

![](https://mmbiz.qpic.cn/mmbiz_jpg/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omPpF4uHMw7dkdwV7OFsBQnoSicyNg6TlvxZ5VicoOVUJuTvhzqBfZF0VA/640?wx_fmt=jpeg)

       HBASE 与 mysql、oralce、db2、sqlserver 等关系型数据库不同，它是一个 NoSQL 数据库（非关系型数据库）

-   Hbase 的表模型与关系型数据库的表模型不同：
-   Hbase 的表没有固定的字段定义；
-   Hbase 的表中每行存储的都是一些 key-value 对
-   Hbase 的表中有列族的划分，用户可以指定将哪些 kv 插入哪个列族
-   Hbase 的表在物理存储上，是按照列族来分割的，不同列族的数据一定存储在不同的文件中
-   Hbase 的表中的每一行都固定有一个行键，而且每一行的行键在表中不能重复
-   Hbase 中的数据，包含行键，包含 key，包含 value，都是 byte\[ ]类型，hbase 不负责为用户维护数据类型
-   HBASE 对事务的支持很差

HBASE 相比于其他 nosql 数据库 (mongodb、redis、cassendra、hazelcast) 的特点：

Hbase 的表数据存储在 HDFS 文件系统中，所以，hbase 具备如下特性：

-   海量存储
-   列式存储  
-   数据存储的安全性可靠性极高！
-   支持高并发
-   存储容量可以线性扩展；  

**2**

**HBase 安装**

HBASE 是一个分布式系统

其中有一个管理角色：HMaster(一般 2 台，一台 active，一台 backup)

其他的数据节点角色：HRegionServer(很多台，看数据量)

1

安装准备

需要先有一个 java 环境

首先，要有一个 HDFS 集群，并正常运行；regionserver 应该跟 hdfs 中的 datanode 在一起

其次，还需要一个 zookeeper 集群，并正常运行，然后，安装 HBASE

角色分配如下：

Hdp01:  namenode  datanode  regionserver  hmaster  zookeeper

Hdp02:  datanode   regionserver  zookeeper

Hdp03:  datanode   regionserver  zookeeper

2

安装步骤

解压 hbase 安装包

修改 hbase-env.sh

```javascript
export JAVA_HOME=/root/apps/jdk1.7.0_67
export HBASE_MANAGES_ZK=false
```

修改 hbase-site.xml  

```xml
<configuration>

        <property>
                <name>hbase.rootdir</name>
                <value>hdfs://hdp01:9000/hbase</value>
        </property>

        <property>
                <name>hbase.cluster.distributed</name>
                <value>true</value>
        </property>

        <property>
                <name>hbase.zookeeper.quorum</name>
                <value>hdp01:2181,hdp02:2181,hdp03:2181</value>
        </property>
</configuration>
```

修改 regionservers  

```nginx
hdp01
hdp02
hdp03
```

3

启动 HBase 集群

```sql
bin/start-hbase.sh
```

启动完后，还可以在集群中找任意一台机器启动一个备用的 master

```sql
bin/hbase-daemon.sh start master
```

新启的这个 master 会处于 backup 状态

**3**

**HBase 初体验**

1

**启动 HBase 命令行客户端**

```cpp
bin/hbase shellHbase> list     
```

NameSpace 操作

```cpp
HBase系统默认定义了两个缺省的namespace
hbase：系统内建表，包括namespace和meta表
default：用户建表时未指定namespace的表都创建在此

创建：create_namespace 'lzc'
删除：drop_namespace 'lzc'
查看：describe_namespace 'lzc'
列出所有：list_namespace
在namespace下创建表：create 'lzc:user_info','id','name','age'
查看namespace下的表 ：list_namespace_tables  'lzc'
```

2

**HBase 表模型特点**

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyasZZsbnDXk98bJfYCKnLVVsuV0zSiaoibYVhP7yI5TdEspv7aicPtlV1afg/640?wx_fmt=png)

1.  一个表，有表名
2.  一个表可以分为多个列族（不同列族的数据会存储在不同文件中）
3.  表中的每一行有一个 “行键 rowkey”，而且行键在表中不能重复
4.  表中的每一对 kv 数据称作一个 cell
5.  hbase 可以对数据存储多个历史版本（历史版本数量可配置）
6.  整张表由于数据量过大，会被横向切分成若干个 region（用 rowkey 范围标识），不同 region 的数据也存储在不同文件中![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcS5k4PoawZSmtvBKZuibY8C60KjojzLmPajJZWyOr5iaDFXv7TEDIvuHMT2eGK2GNIXzxbMdBicTIhA/640?wx_fmt=png)
7.  hbase 会对插入的数据按顺序存储：

     要点一：首先会按行键排序

     要点二：同一行里面的 kv 会按列族排序，再按 k 排序

3

**HBase 数据类型**

hbase 中只支持 byte\[]

此处的 byte\[] 包括了：rowkey,key,value, 列族名, 表名

4

**HBase 命令行操作**

\| 

**名称**

 \| 

**命令表达式**

 \|
\| 

**创建表**

 \| 

create '表名', '列族名 1','列族名 2','列族名 N'

 \|
\| 

**查看所有表**

 \| 

list

 \|
\| 

**描述表**

 \| 

describe  ‘表名’

 \|
\| 

**判断表存在**

 \| 

exists  '表名'

 \|
\| 

**判断是否禁用启用表**

 \| 

is_enabled '表名'

is_disabled ‘表名’

 \|
\| 

**添加记录**

 \| 

put  ‘表名’, ‘rowKey’, ‘列族 : 列‘  ,  '值'

 \|
\| 

**查看记录 rowkey 下的所有数据**

 \| 

get  '表名' , 'rowKey'

 \|
\| 

**查看表中的记录总数**

 \| 

count  '表名'

 \|
\| 

**获取某个列族**

 \| 

get '表名','rowkey','列族'

 \|
\| 

**获取某个列族的某个列**

 \| 

get '表名','rowkey','列族：列’

 \|
\| 

**删除记录**

 \| 

delete  ‘表名’ ,‘行名’ , ‘列族：列'

 \|
\| 

**删除整行**

 \| 

deleteall '表名','rowkey'

 \|
\| 

**删除一张表**

 \| 

先要屏蔽该表，才能对该表进行删除

第一步 disable ‘表名’ ，第二步  drop '表名'

 \|
\| 

**清空表**

 \| 

truncate '表名'

 \|
\| 

**查看所有记录**

 \| 

scan "表名"  

 \|
\| 

**查看某个表某个列中所有数据**

 \| 

scan "表名" , {COLUMNS=>'列族名: 列名'}

 \|
\| 

**更新记录**

 \| 

就是重写一遍，进行覆盖，hbase 没有修改，都是追加

 \|

**建表**

```nginx
create 't_user_info','base_info','extra_info'           
        表名      列族名        列族名
```

**插入数据**

```ruby
hbase(main):011:0> put 't_user_info','001','base_info:username','zhangsan'
0 row(s) in 0.2420 seconds

hbase(main):012:0> put 't_user_info','001','base_info:age','18'
0 row(s) in 0.0140 seconds

hbase(main):013:0> put 't_user_info','001','base_info:sex','female'
0 row(s) in 0.0070 seconds

hbase(main):014:0> put 't_user_info','001','extra_info:career','it'
0 row(s) in 0.0090 seconds

hbase(main):015:0> put 't_user_info','002','extra_info:career','actoress'
0 row(s) in 0.0090 seconds

hbase(main):016:0> put 't_user_info','002','base_info:username','liuyifei'
0 row(s) in 0.0060 seconds
```

**查询方式一 scan 扫描**

```cs
hbase(main):017:0> scan 't_user_info'
ROW                               COLUMN+CELL                                                                                     
001                              column=base_info:age, timestamp=1496567924507, value=18                                         
001                              column=base_info:sex, timestamp=1496567934669, value=female                                     
001                              column=base_info:username, timestamp=1496567889554, value=zhangsan                              
001                              column=extra_info:career, timestamp=1496567963992, value=it                                     
002                              column=base_info:username, timestamp=1496568034187, value=liuyifei                              
002                              column=extra_info:career, timestamp=1496568008631, value=actoress

```

**查询方式二 get 单行数据**

```cs
hbase(main):020:0> get 't_user_info','001'
COLUMN                            CELL                                                                                            
 base_info:age                    timestamp=1496568160192, value=19                                                               
 base_info:sex                    timestamp=1496567934669, value=female                                                           
 base_info:username               timestamp=1496567889554, value=zhangsan                                                         
 extra_info:career                timestamp=1496567963992, value=it                                                               
4 row(s) in 0.0770 seconds
```

**删除一个 kv 数据**

```ruby
hbase(main):021:0> delete 't_user_info','001','base_info:sex'
0 row(s) in 0.0390 seconds
```

**删除整行数据**

```ruby
hbase(main):024:0> deleteall 't_user_info','001'
0 row(s) in 0.0090 seconds
hbase(main):025:0> get 't_user_info','001'
COLUMN                            CELL                                                                                            
0 row(s) in 0.0110 seconds
```

**删除整个表**

```ruby
hbase(main):028:0> disable 't_user_info'
0 row(s) in 2.3640 seconds
hbase(main):029:0> drop 't_user_info'
0 row(s) in 1.2950 seconds
hbase(main):030:0> list
TABLE                                                                                                                             
0 row(s) in 0.0130 seconds
=> []
```

5

**HBase 排序特性**

与 nosql 数据库们一样, row key 是用来检索记录的主键。访问 HBASE table 中的行，只有三种方式：

1.  通过单个 row key 访问
2.  通过 row key 的 range（正则）
3.  全表扫描

Row key 行键 (Row key)可以是任意字符串 (最大长度 是 64KB，实际应用中长度一般为 10-100bytes)，在 HBASE 内部，row key 保存为字节数组。存储时，数据按照 Row key 的字典序(byte order) 排序存储。设计 key 时，要充分排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性)

插入到 hbase 中去的数据，hbase 会自动排序存储：

排序规则： 首先看行键，然后看列族名，然后看列（key）名；按字典顺序

Hbase 的这个特性跟查询效率有极大的关系

比如：一张用来存储用户信息的表，有名字，户籍，年龄，职业.... 等信息

然后，在业务系统中经常需要：

查询某个省的所有用户

经常需要查询某个省的指定姓的所有用户

思路：如果能将相同省的用户在 hbase 的存储文件中连续存储，并且能将相同省中相同姓的用户连续存储，那么，上述两个查询需求的效率就会提高！！！

做法：将查询条件拼到 rowkey 内

![](https://mmbiz.qpic.cn/mmbiz_jpg/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omT8Zuz0EV3IuXPLkRwibuvHspiapJw7Dibk0L1sibSvsVTibybIINNmQEibzA/640?wx_fmt=jpeg)

**4**

**HBase 客户端 API 操作**

```swift
package com.wedoctor.hbase;

import java.util.ArrayList;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.CellUtil;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.MasterNotRunningException;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.ZooKeeperConnectionException;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Delete;
import org.apache.hadoop.hbase.client.Get;
import org.apache.hadoop.hbase.client.HBaseAdmin;
import org.apache.hadoop.hbase.client.HConnection;
import org.apache.hadoop.hbase.client.HConnectionManager;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.client.ResultScanner;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.filter.ColumnPrefixFilter;
import org.apache.hadoop.hbase.filter.CompareFilter;
import org.apache.hadoop.hbase.filter.FilterList;
import org.apache.hadoop.hbase.filter.FilterList.Operator;
import org.apache.hadoop.hbase.filter.RegexStringComparator;
import org.apache.hadoop.hbase.filter.RowFilter;
import org.apache.hadoop.hbase.filter.SingleColumnValueFilter;
import org.apache.hadoop.hbase.util.Bytes;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

public class HbaseTest {

  /**
   * 配置ss
   */
  static Configuration config = null;
  private Connection connection = null;
  private Table table = null;

  @Before
  public void init() throws Exception {
    config = HBaseConfiguration.create();// 配置
    config.set("hbase.zookeeper.quorum", "master,work1,work2");// zookeeper地址
    config.set("hbase.zookeeper.property.clientPort", "2181");// zookeeper端口
    connection = ConnectionFactory.createConnection(config);
    table = connection.getTable(TableName.valueOf("user"));
  }

  /**
   * 创建一个表
   *
   * @throws Exception
   */
  @Test
  public void createTable() throws Exception {
    // 创建表管理类
    HBaseAdmin admin = new HBaseAdmin(config); // hbase表管理
    // 创建表描述类
    TableName tableName = TableName.valueOf("test3"); // 表名称
    HTableDescriptor desc = new HTableDescriptor(tableName);
    // 创建列族的描述类
    HColumnDescriptor family = new HColumnDescriptor("info"); // 列族
    // 将列族添加到表中
    desc.addFamily(family);
    HColumnDescriptor family2 = new HColumnDescriptor("info2"); // 列族
    // 将列族添加到表中
    desc.addFamily(family2);
    // 创建表
    admin.createTable(desc); // 创建表
  }

  @Test
  @SuppressWarnings("deprecation")
  public void deleteTable() throws MasterNotRunningException,
      ZooKeeperConnectionException, Exception {
    HBaseAdmin admin = new HBaseAdmin(config);
    admin.disableTable("test3");
    admin.deleteTable("test3");
    admin.close();
  }

  /**
   * 向hbase中增加数据
   *
   * @throws Exception
   */
  @SuppressWarnings({ "deprecation", "resource" })
  @Test
  public void insertData() throws Exception {
    table.setAutoFlushTo(false);
    table.setWriteBufferSize(534534534);
    ArrayList<Put> arrayList = new ArrayList<Put>();
    for (int i = 21; i < 50; i++) {
      Put put = new Put(Bytes.toBytes("1234"+i));
      put.add(Bytes.toBytes("info"), Bytes.toBytes("name"), Bytes.toBytes("wangwu"+i));
      put.add(Bytes.toBytes("info"), Bytes.toBytes("password"), Bytes.toBytes(1234+i));
      arrayList.add(put);
    }

    //插入数据
    table.put(arrayList);
    //提交
    table.flushCommits();
  }

  /**
   * 修改数据
   *
   * @throws Exception
   */
  @Test
  public void uodateData() throws Exception {
    Put put = new Put(Bytes.toBytes("1234"));
    put.add(Bytes.toBytes("info"), Bytes.toBytes("namessss"), Bytes.toBytes("lisi1234"));
    put.add(Bytes.toBytes("info"), Bytes.toBytes("password"), Bytes.toBytes(1234));
    //插入数据
    table.put(put);
    //提交
    table.flushCommits();
  }

  /**
   * 删除数据
   *
   * @throws Exception
   */
  @Test
  public void deleteDate() throws Exception {
    Delete delete = new Delete(Bytes.toBytes("1234"));
    table.delete(delete);
    table.flushCommits();
  }

  /**
   * 单条查询
   *
   * @throws Exception
   */
  @Test
  public void queryData() throws Exception {
    Get get = new Get(Bytes.toBytes("1234"));
    Result result = table.get(get);
    System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("password"))));
    System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("namessss"))));
    System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("sex"))));
  }

  /**
   * 全表扫描
   *
   * @throws Exception
   */
  @Test
  public void scanData() throws Exception {
    Scan scan = new Scan();
    //scan.addFamily(Bytes.toBytes("info"));
    //scan.addColumn(Bytes.toBytes("info"), Bytes.toBytes("password"));
    scan.setStartRow(Bytes.toBytes("wangsf_0"));
    scan.setStopRow(Bytes.toBytes("wangwu"));
    ResultScanner scanner = table.getScanner(scan);
    for (Result result : scanner) {
      System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("password"))));
      System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name"))));
      //System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("password"))));
      //System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("name"))));
    }
  }

  /**
   * 全表扫描的过滤器
   * 列值过滤器
   *
   * @throws Exception
   */
  @Test
  public void scanDataByFilter1() throws Exception {

    // 创建全表扫描的scan
    Scan scan = new Scan();
    //过滤器：列值过滤器
    SingleColumnValueFilter filter = new SingleColumnValueFilter(Bytes.toBytes("info"),
        Bytes.toBytes("name"), CompareFilter.CompareOp.EQUAL,
        Bytes.toBytes("zhangsan2"));
    // 设置过滤器
    scan.setFilter(filter);

    // 打印结果集
    ResultScanner scanner = table.getScanner(scan);
    for (Result result : scanner) {
      System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("password"))));
      System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name"))));
    }

  }
  /**
   * rowkey过滤器
   * @throws Exception
   */
  @Test
  public void scanDataByFilter2() throws Exception {

    // 创建全表扫描的scan
    Scan scan = new Scan();
    //匹配rowkey以wangsenfeng开头的
    RowFilter filter = new RowFilter(CompareFilter.CompareOp.EQUAL, new RegexStringComparator("^12341"));
    // 设置过滤器
    scan.setFilter(filter);
    // 打印结果集
    ResultScanner scanner = table.getScanner(scan);
    for (Result result : scanner) {
      System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("password"))));
      System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name"))));
      //System.out.println(Bytes.toInt(result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("password"))));
      //System.out.println(Bytes.toString(result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("name"))));
    }


  }

  /**
   * 匹配列名前缀
   * @throws Exception
   */
  @Test
  public void scanDataByFilter3() throws Exception {

    // 创建全表扫描的scan
    Scan scan = new Scan();
    //匹配rowkey以wangsenfeng开头的
    ColumnPrefixFilter filter = new ColumnPrefixFilter(Bytes.toBytes("na"));
    // 设置过滤器
    scan.setFilter(filter);
    // 打印结果集
    ResultScanner scanner = table.getScanner(scan);
    for (Result result : scanner) {
      System.out.println("rowkey：" + Bytes.toString(result.getRow()));
      System.out.println("info:name："
          + Bytes.toString(result.getValue(Bytes.toBytes("info"),
              Bytes.toBytes("name"))));
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info"), Bytes.toBytes("age")) != null) {
        System.out.println("info:age："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info"),
                Bytes.toBytes("age"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info"), Bytes.toBytes("sex")) != null) {
        System.out.println("infi:sex："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info"),
                Bytes.toBytes("sex"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("name")) != null) {
        System.out
        .println("info2:name："
            + Bytes.toString(result.getValue(
                Bytes.toBytes("info2"),
                Bytes.toBytes("name"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("age")) != null) {
        System.out.println("info2:age："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info2"),
                Bytes.toBytes("age"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("sex")) != null) {
        System.out.println("info2:sex："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info2"),
                Bytes.toBytes("sex"))));
      }
    }

  }
  /**
   * 过滤器集合
   * @throws Exception
   */
  @Test
  public void scanDataByFilter4() throws Exception {

    // 创建全表扫描的scan
    Scan scan = new Scan();
    //过滤器集合：MUST_PASS_ALL（and）,MUST_PASS_ONE(or)
    FilterList filterList = new FilterList(Operator.MUST_PASS_ONE);
    //匹配rowkey以wangsenfeng开头的
    RowFilter filter = new RowFilter(CompareFilter.CompareOp.EQUAL, new RegexStringComparator("^wangsenfeng"));
    //匹配name的值等于wangsenfeng
    SingleColumnValueFilter filter2 = new SingleColumnValueFilter(Bytes.toBytes("info"),
        Bytes.toBytes("name"), CompareFilter.CompareOp.EQUAL,
        Bytes.toBytes("zhangsan"));
    filterList.addFilter(filter);
    filterList.addFilter(filter2);
    // 设置过滤器
    scan.setFilter(filterList);
    // 打印结果集
    ResultScanner scanner = table.getScanner(scan);
    for (Result result : scanner) {
      System.out.println("rowkey：" + Bytes.toString(result.getRow()));
      System.out.println("info:name："
          + Bytes.toString(result.getValue(Bytes.toBytes("info"),
              Bytes.toBytes("name"))));
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info"), Bytes.toBytes("age")) != null) {
        System.out.println("info:age："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info"),
                Bytes.toBytes("age"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info"), Bytes.toBytes("sex")) != null) {
        System.out.println("infi:sex："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info"),
                Bytes.toBytes("sex"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("name")) != null) {
        System.out
        .println("info2:name："
            + Bytes.toString(result.getValue(
                Bytes.toBytes("info2"),
                Bytes.toBytes("name"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("age")) != null) {
        System.out.println("info2:age："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info2"),
                Bytes.toBytes("age"))));
      }
      // 判断取出来的值是否为空
      if (result.getValue(Bytes.toBytes("info2"), Bytes.toBytes("sex")) != null) {
        System.out.println("info2:sex："
            + Bytes.toInt(result.getValue(Bytes.toBytes("info2"),
                Bytes.toBytes("sex"))));
      }
    }

  }

  @After
  public void close() throws Exception {
    table.close();
    connection.close();
  }

}
```

**5**

**HBase 运行原理**

1

**master**

职责

1\. 管理监控 HRegionServer，实现其负载均衡。

2\. 处理 region 的分配或转移，比如在 HRegion split 时分配新的 HRegion；在 HRegionServer 退出时迁移其负责的 HRegion 到其他 HRegionServer 上。

3\. 处理元数据的变更

4\. 管理 namespace 和 table 的元数据（实际存储在 HDFS 上）。

5\. 权限控制（ACL）。

6\. 监控集群中所有 HRegionServer 的状态 (通过 Heartbeat 和监听 ZooKeeper 中的状态)。

2

**Region Server**

1.  管理自己所负责的 region 数据的读写。
2.  读写 HDFS，管理 Table 中的数据。
3.  Client 直接通过 HRegionServer 读写数据（从 HMaster 中获取元数据，找到 RowKey 所在的 HRegion/HRegionServer 后）。
4.  刷新缓存到 HDFS
5.  维护 Hlog
6.  执行压缩
7.  负责分裂（region split）在运行过程中变得过大的 region

![](https://mmbiz.qpic.cn/mmbiz_jpg/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omTddLYiatNoA6wgIsgSDdqsSejN44oAbvOVbUibiaIT2dVCusrDVknPCPQ/640?wx_fmt=jpeg)

3

**zookeeper 作用**

1.  存放整个 HBase 集群的元数据以及集群的状态信息。
2.  实现 HMaster 主从节点的 failover。

注：HMaster 通过监听 ZooKeeper 中的 Ephemeral 节点 (默认：/hbase/rs/\*) 来监控 HRegionServer 的加入和宕机。

在第一个 HMaster 连接到 ZooKeeper 时会创建 Ephemeral 节点 (默认：/hbasae/master) 来表示 Active 的 HMaster，其后加进来的 HMaster 则监听该 Ephemeral 节点

如果当前 Active 的 HMaster 宕机，则该节点消失，因而其他 HMaster 得到通知，而将自身转换成 Active 的 HMaster，在变为 Active 的 HMaster 之前，它会在 / hbase/masters / 下创建自己的 Ephemeral 节点。

4

**HBase 读写流程**

### 写数据流程

客户端现在要插入一条数据，rowkey=r000001, 这条数据应该写入到 table 表中的那个 region 中呢？

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcS5k4PoawZSmtvBKZuibY8CoKLeBhXo0yGk8DVLjibJ65DjGdqamib4nf3I94COcNCp0qS8Fiavfcu2A/640?wx_fmt=png)

1.  客户端要连接 zookeeper, 从 zk 的 / hbase 节点找到 hbase:meta 表所在的 regionserver（host:port）;
2.  regionserver 扫描 hbase:meta 中的每个 region 的起始行健，对比 r000001 这条数据在那个 region 的范围内；
3.  从对应的 info:server key 中存储了 region 是有哪个 regionserver(host:port) 在负责的；
4.  客户端直接请求对应的 regionserver；
5.  regionserver 接收到客户端发来的请求之后，就会将数据写入到 region 中

### 读数据流程

客户端现在要查询 rowkey=r000001 这条数据，那么这个流程是什么样子的呢？

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcS5k4PoawZSmtvBKZuibY8CBABt0RBQhuONIfCeNzQCmhtKtCZunAp5wrkVvia0qojTq8rQ7WiaQmhw/640?wx_fmt=png)

1.  首先 Client 连接 zookeeper, 找到 hbase:meta 表所在的 regionserver;
2.  请求对应的 regionserver，扫描 hbase:meta 表，根据 namespace、表名和 rowkey 在 meta 表中找到 r00001 所在的 region 是由那个 regionserver 负责的；
3.  找到这个 region 对应的 regionserver
4.  regionserver 收到了请求之后，扫描对应的 region 返回数据到 Client

(先从 MemStore 找数据，如果没有，再到 BlockCache 里面读；BlockCache 还没有，再到 StoreFile 上读 (为了读取的效率)；

如果是从 StoreFile 里面读取的数据，不是直接返回给客户端，而是先写入 BlockCache，再返回给客户端。)

blockcache 逐渐满了之后，会采用 LRU 的淘汰策略

注：客户会缓存这些位置信息，然而第二步它只是缓存当前 RowKey 对应的 HRegion 的位置，因而如果下一个要查的 RowKey 不在同一个 HRegion 中，则需要继续查询 hbase:meta 所在的 HRegion，然而随着时间的推移，客户端缓存的位置信息越来越多，以至于不需要再次查找 hbase:meta Table 的信息，除非某个 HRegion 因为宕机或 Split 被移动，此时需要重新查询并且更新缓存。

5

HBase:meta 表

hbase:meta 表存储了所有用户 HRegion 的位置信息：

Rowkey：tableName,regionStartKey,regionId,replicaId 等；

info 列族：这个列族包含三个列，他们分别是：

-   info:regioninfo 列：regionId,tableName,startKey,endKey,offline,split,replicaId；
-   info:server 列：HRegionServer 对应的 server:port；
-   info:serverstartcode 列：HRegionServer 的启动时间戳。

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcS5k4PoawZSmtvBKZuibY8CJLqnmrKHXlHjwaDIXLHPgtwFccMibsqY2So2UDsVposA92caPPbbdbA/640?wx_fmt=png)

6

Region Server 内部机制

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyasVQJ9IpNIq9O8wTR9M4AyibOs2mQC4haib2lTwjFo0HhjaHaoCTckPFpg/640?wx_fmt=png)

-   WAL 即 Write Ahead Log，在早期版本中称为 HLog，它是 HDFS 上的一个文件，如其名字所表示的，所有写操作都会先保证将数据写入这个 Log 文件后，才会真正更新 MemStore，最后写入 HFile 中。WAL 文件存储在 / hbase/WALs/${HRegionServer_Name} 的目录中


-   BlockCache 是一个读缓存，即 “引用局部性” 原理（也应用于 CPU，分空间局部性和时间局部性，空间局部性是指 CPU 在某一时刻需要某个数据，那么有很大的概率在一下时刻它需要的数据在其附近；时间局部性是指某个数据在被访问过一次后，它有很大的概率在不久的将来会被再次的访问），将数据预读取到内存中，以提升读的性能。


-   HRegion 是一个 Table 中的一个 Region 在一个 HRegionServer 中的表达。一个 Table 可以有一个或多个 Region，他们可以在一个相同的 HRegionServer 上，也可以分布在不同的 HRegionServer 上，一个 HRegionServer 可以有多个 HRegion，他们分别属于不同的 Table。

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyasGz4Lzt1I2ibt2aezSz8Xea2Iq5KAvSbciaaF5I7tBkx5rzJU1RvWGR8Q/640?wx_fmt=png)

-    region 按大小分割的，每个表一开始只有一个 region，随着数据不断插入表，region 不断增大，当增大到一个阀值的时候，Hregion 就会等分会两个新的 Hregion。当 table 中的行不断增多，就会有越来越多的 Hregion。

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyasycVG4GZJ7sfKyy0o1FG8tMMljhbjISBHxiaLkstIag8Q5aMJyXFZstA/640?wx_fmt=png)

-   HRegion 由多个 Store(HStore) 构成，每个 HStore 对应了一个 Table 在这个 HRegion 中的一个 Column Family，即每个 Column Family 就是一个集中的存储单元，因而最好将具有相近 IO 特性的 Column 存储在一个 Column Family，以实现高效读取 (数据局部性原理，可以提高缓存的命中率)。HStore 是 HBase 中存储的核心，它实现了读写 HDFS 功能，一个 HStore 由一个 MemStore 和 0 个或多个 StoreFile 组成。

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyas3HCh7dOCwWyAud6ECrXtMJocC1QibFnj1iaqmEyQlFqtj6lMS7yp2KtQ/640?wx_fmt=png)

-   MemStore 是一个写缓存 (In Memory Sorted Buffer)，所有数据的写在完成 WAL 日志写后，会 写入 MemStore 中，由 MemStore 根据一定的算法将数据 Flush 到地层 HDFS 文件中 (HFile)，通常每个 HRegion 中的每个 Column Family 有一个自己的 MemStore。


-   HFile(StoreFile) 用于存储 HBase 的数据 (Cell/KeyValue)。在 HFile 中的数据是按 RowKey、Column Family、Column 排序，对相同的 Cell(即这三个值都一样)，则按 timestamp 倒序排列。

flush 触发条件

1.  每一次请求都是先写入到 MemStore 中，当 MemStore 满后会 Flush 成一个新的 StoreFile(底层实现是 HFile)，即一个 HStore(Column Family) 可以有 0 个或多个 StoreFile(HFile)。
2.  当一个 HRegion 中的所有 MemStore 的大小总和超过了 hbase.hregion.memstore.flush.size 的大小，默认 128MB。此时当前的 HRegion 中所有的 MemStore 会 Flush 到 HDFS 中。
3.  当全局 MemStore 的大小超过了 hbase.regionserver.global.memstore.upperLimit 的大小，默认 40％的内存使用量。此时当前 HRegionServer 中所有 HRegion 中的 MemStore 都会 Flush 到 HDFS 中，Flush 顺序是 MemStore 大小的倒序（一个 HRegion 中所有 MemStore 总和作为该 HRegion 的 MemStore 的大小还是选取最大的 MemStore 作为参考？有待考证），直到总体的 MemStore 使用量低于 hbase.regionserver.global.memstore.lowerLimit，默认 38% 的内存使用量。
4.  当前 HRegionServer 中 WAL 的大小超过了_hbase.regionserver.hlog.blocksize \* hbase.regionserver.max.logs_的数量，当前 HRegionServer 中所有 HRegion 中的 MemStore 都会 Flush 到 HDFS 中，Flush 使用时间顺序，最早的 MemStore 先 Flush 直到 WAL 的数量少于_hbase.regionserver.hlog.blocksize \* hbase.regionserver.max.logs_这里说这两个相乘的默认大小是 2GB，查代码，hbase.regionserver.max.logs 默认值是 32，而 hbase.regionserver.hlog.blocksize 默认是 32MB。但不管怎么样，因为这个大小超过限制引起的 Flush 不是一件好事，可能引起长时间的延迟
5.  手动触发

**6**

**Hive 表映射 HBase**

**建 HBase 表**

```ruby
hbase(main):018:0> create 'user_info','info'
```

**数据插入 HBase**

info:order_amt

info:order_id

info:user_id

info:user_name

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyassSicg8dYt71SQIGMS00jwA4Vtr65PFpr06DaIMdCeIicJ2KA4mrVPJDA/640?wx_fmt=png)

建 hive 映射表

```sql
create external table wedw_tmp.t_user_info
(
id        string
,order_id  string
,order_amt string
,user_id   string
,user_name string
)
STORED by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
with serdeproperties("hbase.columns.mapping"=":key,info:order_id,info:order_amt,info:user_id,info:user_name")
tblproperties("hbase.table.name"="user_info");
```

查询映射好的 hive 表

```sql
select * from wedw_tmp.t_user_info;
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcXIu50VIwhVjZMPvK5gyasZ4nUJL7OB13NfsGSAqpoaKa9G1gQhG8rMERYMya4XDccuPr9zHx2PA/640?wx_fmt=png)

**7**

**BuckLoader**

       如果我们离线计算好的 hive 数据需要同步到 hbase 中，大家会用什么方法呢？如果是明细数据，上千万乃至上亿行的数据，导入到 hbase 中肯定是需要考虑效率问题的。如果是直接使用 hbase 客户端的 API 进行数据插入，效率是非常低的

     所以我们选择了 bulkloader 工具进行操作 (原理：利用 hbase 之外的计算引擎将源数据加工成 hbase 的底层文件格式：Hfile，然后通知 hbase 导入即可)

1

测试数据

```sql
CREATE TABLE wedw_dw.t_user_order_info(                                
   user_id string                                   
  ,user_name string
  ,order_id  string
  ,order_amt decimal(16,2)
)
ROW FORMAT SERDE                                      
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (                                
  'field.delim'=',',                                  
  'serialization.format'=',')       
  ;
+----------+------------+-----------+------------+--+
| user_id  | user_name  | order_id  | order_amt  |
+----------+------------+-----------+------------+--+
| 1        | 小红         | 001       | 100.32     |
| 2        | 小明         | 002       | 34.76      |
| 3        | 小花         | 003       | 39.88      |
| 4        | 小牛         | 004       | 22.22      |
| 5        | 小刘         | 005       | 98765.34   |
+----------+------------+-----------+------------+--+
# /data/hive/warehouse/wedw/dw/t_user_order_info/
```

2

**利用 hbasse 自带程序导入**

\# hbase 建表

```javascript
hbase(main):009:0* create 'user_order_info','user_info','order_info'
```

\# 执行 hbase 自带的 importtsv 程序（mapreduce 程序），将原始文件转成 hfile

```ruby
/usr/local/hadoop-current/bin/yarn jar \/usr/local/hbase-current/lib/hbase-server-1.2.0-cdh5.8.2.jar  \importtsv -Dimporttsv.columns=HBASE_ROW_KEY,user_info:user_name,order_info:order_id,order_info:order_amt \'-Dimporttsv.separator=,' \-Dmapreduce.job.queuename='root.test' \-Dimporttsv.bulk.output=hdfs://cluster/data/hive/output1 user_order_info \hdfs://cluster/data/hive/warehouse/wedw/dw/t_user_order_info
```

完整参数：

-     \-Dimporttsv.bulk.output=/path/for/output   输出目录
-     \-Dimporttsv.skip.bad.lines=false   是否跳过脏数据行
-     \-Dimporttsv.separator=|'   指定分隔符
-     \-Dimporttsv.timestamp=currentTimeAsLong 是否指定时间戳
-     \-Dimporttsv.mapper.class=my.Mapper  替换默认的 Mapper 类

\# 移动数据到 hbase 表中

```javascript
hadoop jar hbase-server-1.2.0-cdh5.8.2.jar completebulkload  hdfs://cluster/data/hive/output1 user_order_info
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLdiczbcjCXp106hXaFXTud0ibSF1AefNQKNnE6tMjQrNaWJrNf6ia2wze5jRPNm23P3j4e6HtU1RIGJg/640?wx_fmt=png)

3

**编写代码导入**

hbase 建表  

```ruby
hbase(main):018:0> create 'user_info','info'
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.wedoctor.spark</groupId>
    <artifactId>spark-0708</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <scala.version>2.11.8</scala.version>
        <spark.version>2.2.0</spark.version>
        <hadoop.version>2.8.1</hadoop.version>
        <encoding>UTF-8</encoding>
    </properties>
    <dependencies>
        
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.41</version>
        </dependency>
        
        <dependency>
            <groupId>com.typesafe</groupId>
            <artifactId>config</artifactId>
            <version>1.3.0</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.8.1</version>
        </dependency>
        <dependency>
            <groupId>org.scalikejdbc</groupId>
            <artifactId>scalikejdbc_2.11</artifactId>
            <version>2.5.0</version>
        </dependency>
        
        <dependency>
            <groupId>org.scalikejdbc</groupId>
            <artifactId>scalikejdbc-config_2.11</artifactId>
            <version>2.5.0</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.10.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.2.0-cdh5.8.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.2.0-cdh5.8.2</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.httpcomponents</groupId>
                    <artifactId>httpclient</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.httpcomponents</groupId>
                    <artifactId>httpcore</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
    <repositories>
        <repository>
            <id>cloudera</id>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
        </repository>
    </repositories>
    <build>
        <plugins>
          
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
            </plugin>
            
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                        <configuration>
                            <args>
                                <arg>-dependencyfile</arg>
                                <arg>${project.build.directory}/.scala_dependencies</arg>
                            </args>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

spark 程序编写

```typescript
package com.hbase.bulkloader
import org.apache.hadoop.fs.{Path}
import org.apache.hadoop.hbase.client.ConnectionFactory
import org.apache.hadoop.hbase.{HBaseConfiguration, KeyValue, TableName}
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapreduce.{HFileOutputFormat2, LoadIncrementalHFiles}
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.mapreduce.Job
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{DataFrame, SparkSession}
object BulkLoader {
  //Logger.getLogger("org").setLevel(Level.ERROR)
  def main(args: Array[String]): Unit = {
    System.setProperty("HADOOP_USER_NAME", "pgxl")
    val spark: SparkSession = SparkSession.builder()
      .master("local[*]")
      .config("hive.metastore.uris", "thrift://10.11.3.44:9999")
      .appName("bulkloaderTest")
      .enableHiveSupport()
      .getOrCreate()
    val re: DataFrame = spark.sql("select * from wedw_dw.t_user_order_info")
    val dataRdd: RDD[(String, (String, String, String))] = re.rdd.flatMap(row => {
      val rowkey: String = row.getAs[String]("user_id").toString
      Array(
        (rowkey, ("info", "user_id", row.getAs[String]("user_id"))),
        (rowkey, ("info", "user_name", row.getAs[String]("user_name"))),
        (rowkey, ("info", "order_id", row.getAs[String]("order_id"))),
        (rowkey, ("info", "order_amt", row.get(3).toString))
      )
    })
    val output = dataRdd.filter(x=>x._1 != null).sortBy(x=>(x._1,x._2._1,x._2._2)).map {
      x => {
        val rowKey = Bytes.toBytes(x._1)
        val immutableRowKey = new ImmutableBytesWritable(rowKey)
        val colFam = x._2._1
        val colName = x._2._2
        val colValue = x._2._3
        val kv = new KeyValue(
          rowKey,
          Bytes.toBytes(colFam),
          Bytes.toBytes(colName),
          Bytes.toBytes(colValue.toString)
        )
        (immutableRowKey, kv)
      }
    }
    val conf = HBaseConfiguration.create()
    conf.set("fs.defaultFS", "hdfs://cluster")
    conf.set("hbase.zookeeper.quorum", "10.11.3.43")
    val job = Job.getInstance(conf)
    val conn = ConnectionFactory.createConnection(conf)
    val table = conn.getTable(TableName.valueOf("user_info"))
    val locator = conn.getRegionLocator(TableName.valueOf("user_info"))
    // 将我们自己的数据保存为HFile
    HFileOutputFormat2.configureIncrementalLoad(job, table, locator)
    output.saveAsNewAPIHadoopFile("/data/hive/test/", classOf[ImmutableBytesWritable], classOf[KeyValue], classOf[HFileOutputFormat2], job.getConfiguration)
    // 构造一个导入hfile的工具类
    new LoadIncrementalHFiles(job.getConfiguration).doBulkLoad(new Path("/data/hive/test/"),conn.getAdmin,table,locator)
    conn.close()
    spark.close()
  }
}
```

hbase 表结果：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLdB6Ine9YIhXj71NYTzGDKCcNysuTwaQic0BVxfDVXAEfO7W785ZibSiaDeW2TczNy5xaYPnHibua1qTA/640?wx_fmt=png)

**8**

**HBase 优化**

1

HBase 高可用

       在 HBase 中 Hmaster 负责监控 RegionServer 的生命周期，均衡 RegionServer 的负载，如果 Hmaster 挂掉了，那么整个 HBase 集群将陷入不健康的状态，此时的工作状态并不会维持太久。所以需要配置 hbase 的高可用

2

预分区

       每一个 region 维护着 startRow 与 endRowKey，如果加入的数据符合某个 region 维护的 rowKey 范围，则该数据交给这个 region 维护。那么依照这个原则，我们可以将数据所要投放的分区提前大致的规划好，以提高 HBase 性能。

手动设置预分区

```typescript
hbase> create 'user','info','partition1',SPLITS => ['a','c','f','h']
```

生成 16 进制序列预分区  

```typescript
create 'user2','info','partition2',{NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
```

按照文件中设置的规则预分区  

创建 splits.txt 文件内容如下：

然后执行：

```typescript
create 'user3','partition3',SPLITS_FILE => 'splits.txt'
```

使用 javaAPI 创建预分区

```cs
//自定义算法，产生一系列Hash散列值存储在二维数组中
byte[][] splitKeys = 某个散列值函数
//创建HBaseAdmin实例
HBaseAdmin hAdmin = new HBaseAdmin(HBaseConfiguration.create());
//创建HTableDescriptor实例
HTableDescriptor tableDesc = new HTableDescriptor(tableName);
//通过HTableDescriptor实例和散列值二维数组创建带有预分区的HBase表
hAdmin.createTable(tableDesc, splitKeys);
```

3

优化 RowKey 设计 (防止热点)

       Hbase 中一条数据的唯一标识就是 Rowkey，类似于关系型数据库中的主键，HBase 中的数据是根据 Rowkey 的字典顺序来排序的。

      那么这条数据存储于哪个分区，取决于 Rowkey 处于哪一个预分区的区间内，设计 Rowkey 的主要目的 ，就是让数据均匀的分布于所有的 Region 中，在一定程度上防止数据倾斜。尽量在访问的时候不会出现热点现象

**什么是热点**  

       因为 HBase 中的行是按照 Rowkey 的字典顺序排序的，这种设计使得 Scan 操作更为方便，但是也容易出现热点问题。  

       热点问题是大量的客户端只访问集群的一个或少数节点，大量访问请求会使该台机器的负载很高，直接导致性能下降，甚至 Region 不可用，而集群的其他节点却处于相对空闲的状态。

**长度**  

      Rowkey 可以使任意字符串，最大长度 64kb，建议越短越好，最好不要超过 16 个字节，原因如下:  

-   目前操作系统都是 64 位系统，内存 8 字节对齐，控制在 16 字节，8 字节的整数倍利用了操作系统的最佳特性。
-   Hbase 将部分数据加载到内存当中，如果 Rowkey 太长，内存的有效利用率就会下降。

**唯一**

       Rowkey 必须保证是唯一的，如果不唯一的话，同一版本同一个 Rowkey 插入 Hbase 中会更新之前的数据，与需求不符  

**散列**  

**加盐**

      在 Rowkey 的前面增加随机数，散列之后的 Rowkey 就会根据随机生成的前缀分散到各个 Region 上，可以有效的避免热点问题。

     加盐这种方式增加了写的吞吐，但是使得读数据更加困难  

**Hash**

     Hash 算法包含了 MD5 等算法，可以直接取 Rowkey 的 MD5 值作为 Rowkey，或者取 MD5 值拼接原始 Rowkey，组成新的 rowkey，由于 Rowkey 设计不应该太长，所以可以对 MD5 值进行截取拼接

**字符串反转**

-   时间戳反转  
-   手机号反转
-   ...

4

内存优化

       HBase 操作过程中需要大量的内存开销，毕竟 Table 是可以缓存在内存中的，一般会分配整个可用内存的 70% 给 HBase 的 Java 堆。

     但是不建议分配非常大的堆内存，因为 GC 过程持续太久会导致 RegionServer 处于长期不可用状态，一般 16~48G 内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。

5

压缩

生产系统应使用其 ColumnFamily 定义进行压缩。

### 注意

压缩会缩小磁盘上的数据。当它在内存中（例如，在 MemStore 中）或在线上（例如，在 RegionServer 和 Client 之间传输）时，它会膨胀。因此，虽然使用 ColumnFamily 压缩是最佳做法，但它不会完全消除过大的 Keys，过大的 ColumnFamily 名称或过大的列名称的影响。

6

column 数控制

     Hbase 中的每个列，都归属于某个列簇，列簇是表的 schema 的一部分 (列不是), 必须在使用之前定义  

        HBase 目前对于两列族或三列族以上的任何项目都不太合适，因此请将模式中的列族数量保持在较低水平。

        目前，flushing 和 compactions 是按照每个区域进行的，所以如果一个列族承载大量数据带来的 flushing，即使所携带的数据量很小，也会 flushing 相邻的列族。当许多列族存在时，flushing 和 compactions 相互作用可能会导致一堆不必要的 I/O（要通过更改 flushing 和 compactions 来针对每个列族进行处理）。

7

开启布隆过滤器

什么是布隆过滤器  

       布隆过滤器是一种多哈希函数映射的快速查找算法 (存储结构），可以实现用很小的空间和运算代价，来实现海量数据的存在与否的记录（黑白名单判断）。特点是高效的插入和查询，可以判断出一定不存在和可能存在

      相比于传统的 List、Set、Map 等数据结构，它更高效、占用空间更少，但是缺点是其返回的可能存在结果是概率性的，而不是确切的。

布隆过滤器是一个 bit 向量或者 bit 数组

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfC55OQ9nvScow4wPaZrjia8Vj8V6oUtV5v2o0IN8QG4BwWGHnAQaaBv8mzFyBWAKSBUSj7EiagHuoA/640?wx_fmt=png)

布隆过滤器操作及原理  

        如果我们要添加一个值到布隆过滤器中，需要使用多个 hash 函数生成多个 hash 值，每个 hash 值对应位数组上的一个点，然后将位数组对应的位置标记为 1。

      如下图，字符串'hello'就通过 3 种 hash 函数生成了哈希值 1,3,9，字符串‘word’就生成了 1,5,7

注：由于 hello 和 word 都返回了 bit 位 1，所以前面的 1 会被覆盖

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfC55OQ9nvScow4wPaZrjia8gLfKLueYp5NDg4a01kS1fHro778DyGsjhRQcYo2iciayMeUASaYHcJVA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLfC55OQ9nvScow4wPaZrjia8dlcdPvv6ZdfR5YicUJkGMevm2I2S85Fbic5VBicK8XbHeAEiarXaz4sZ8A/640?wx_fmt=png)

      查询元素 flink 是否存在集合中的时候，同样的方法将 flink 通过哈希映射到位数组上的 3 个点。如果 3 个点的其中有一个点不为 1，则可以判断该元素一定不存在集合中。

       如果 3 个点都为 1，则该元素可能存在集合中。注意：此处不能判断该元素是否一定存在，可能存在一定的误判率。

      因为新增的元素越来越多，被置为 1 的 bit 位也会越来越多，这样 “flink” 即使没有被存储过，但是万一哈希函数返回的三个 bit 位都被其他值置位了 1 ，那么程序还是会判断 “flink” 这个值存在。比如 flink 三个 hash 值为 1,3,7, 就能说明 flink 一定存在吗?

如何建设误差？

-   加大布隆过滤器的长度，否则很容易就所有的 bit 位都为 1 了  
-   哈希函数的个数要考虑，个数越多则布隆过滤器 bit 位置位 1 的速度越快，且布隆过滤器的效率越低；但是如果太少的话，那我们的误差会变高。

布隆过滤器在 HBase 中的应用  

       布隆过滤器是 hbase 中的高级功能，它能够减少特定访问模式（get/scan）下的查询时间。不过由于这种模式增加了内存和存储的负担，所以被默认为关闭状态。

hbase 支持如下类型的布隆过滤器：

-   NONE          不使用布隆过滤器
-   ROW           行键使用布隆过滤器
-   ROWCOL    列键使用布隆过滤器

至于选用什么模式，还是要看使用场景

        hbase 的实际存储结构是 HFile，它是位于 hdfs 系统中的，也就是在磁盘中。而加载到内存中的数据存储在 MemStore 中，当 MemStore 中的数据达到一定数量时，它会将数据存入 HFile 中。

       HFIle 是由一个个数据块与索引块组成，他们通常默认为 64KB。hbase 是通过块索引来访问这些数据块的。而索引是由每个数据块的第一行数据的 rowkey 组成的。当 hbase 打开一个 HFile 时，块索引信息会优先加载到内存当中。然后 hbase 会通过这些块索引来查询数据。

      当我们随机读 get 数据时，如果采用 hbase 的块索引机制，hbase 会加载很多块文件。

       采用布隆过滤器后，它能够准确判断该 HFile 的所有数据块中是否含有我们查询的数据，从而大大减少不必要的块加载，增加吞吐，降低内存消耗，提高性能

         在读取数据时，hbase 会首先在布隆过滤器中查询，根据布隆过滤器的结果，再在 MemStore 中查询，最后再在对应的 HFile 中查询。

**9**

**HBase 通过 springboot 提供接口**

1

简介

      Spring Boot 是由 Pivotal 团队提供的全新框架，其设计目的是用来简化新 Spring 应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。通过这种方式，Spring Boot 致力于在蓬勃发展的快速应用开发领域 (rapid application development) 成为领导者

数据开发人员应该是需要具备 springboot 开发提供接口服务能力的！！！  

2

Hbase 数据准备

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLdiczbcjCXp106hXaFXTud0ibSF1AefNQKNnE6tMjQrNaWJrNf6ia2wze5jRPNm23P3j4e6HtU1RIGJg/640?wx_fmt=png)

3

工程创建

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpbhxowK3NibS0HibkBLAYvKZ4tkVOQCPwwXm0ZvwicFfj0Il4WYaImrjoQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5Yp19NZFr9kKqH98EgTQr84aOZA66ckqTh0QPoQuibxhDIBNN47NytYBlw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpwLqfDic8IrPxXWLzj6c8YSy9KKibWFnAPvM23wajfqLnLTgq1OWCNWwQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YprMwq6C4sJDXQEIUvHdfrMWYeMTj2IShxBUV1P5icJb4ZqXiatFzGU7LQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5Yps8g5vSsN2s716dMjSNskj7rFEBrnyLnqiaQRfmoeAqyxzSAORQvgemg/640?wx_fmt=png)

运行 Application，出现以下界面，则表示创建工程成功

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5Ypo97l86avybicqw4uydc8Jo2icENgFeHjMq5IHXiaszicIKiaZ48VXbT5U2A/640?wx_fmt=png)

4

简单测试

UserInfo.java

```java
package com.wedoctor.demo.domain;

public class UserInfo {

    private String userId;
    private String UserName;
    private Integer age;

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getUserName() {
        return UserName;
    }

    public void setUserName(String userName) {
        UserName = userName;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public UserInfo(String userId, String userName, Integer age) {
        this.userId = userId;
        UserName = userName;
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserInfo{" +
                "userId='" + userId + '\'' +
                ", UserName='" + UserName + '\'' +
                ", age=" + age +
                '}';
    }
}

```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5Ypleic1amuBWFJzQt34cha8vX8jn3A3Tu54nd3zgvDN7kJL0hmQyfj9cA/640?wx_fmt=png)

测试 ok  

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5Ypn7jrZhsLPue16nfZtXF1uXoaJmBJRvybYQh4zsbxibq3h0HQefpUolg/640?wx_fmt=png)

5

读取 hbase 数据并提供接口

整体代码结构  

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpcH5KAWNIg9xY6xY8PwJAmvYgZpebZ9j6fMyG4HZqjzCYp5GrSyJBtA/640?wx_fmt=png)

Controller 层  

```kotlin
package com.wedoctor.demo.controller;

import com.wedoctor.demo.domain.UserInfo;
import com.wedoctor.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {


    @Autowired
    UserService userService;
    //http://localhost:8080/api/getName/2

    @RequestMapping("user")
    public UserInfo showUser(){
        UserInfo user = new UserInfo("1","小红",20);
        return user;
    }
    @RequestMapping("/api/getName/{rowkey}")
    public String getHbaseUserInfo(@PathVariable String rowkey) throws Exception{

        return userService.getHbaseUserInfo(rowkey);
    }
}

```

Service 层  

```java
package com.wedoctor.demo.service;


public interface UserService {

    String getHbaseUserInfo(String rowkey) throws  Exception;

}

```

```css
package com.wedoctor.demo.service.impl;

import com.wedoctor.demo.dao.UserDao;
import com.wedoctor.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;


@Service
public class UserServiceImpl implements UserService {

    @Autowired
    UserDao userDao;


    @Override
    public String getHbaseUserInfo(String rowkey) throws Exception{
        return userDao.getHbaseUserInfo(rowkey);
    }
}

```

Dao 层

```swift
package com.wedoctor.demo.dao;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;
import org.springframework.stereotype.Repository;

import java.io.IOException;

@Repository
public class UserDao {

    Table table = null;
    public UserDao() throws IOException {
        System.setProperty("HADOOP_USER_NAME", "root");
        Configuration conf = HBaseConfiguration.create();
        conf.set("fs.defaultFS", "hdfs://test");
        conf.set("hbase.zookeeper.quorum", "192.168.1.111,192.168.1.222,192.168.1.333");
        Connection conn = ConnectionFactory.createConnection(conf);
        table = conn.getTable(TableName.valueOf("user_info"));
    }

    public String getHbaseUserInfo(String rowkey) throws Exception{

        Get get = new Get(Bytes.toBytes(rowkey));
        Result result = table.get(get);

        /*CellScanner cellScanner = result.cellScanner();
        String res = null;
        while(cellScanner.advance()){
            Cell cell = cellScanner.current();
            byte[] valueBytes = CellUtil.cloneValue(cell);
            res = new String(valueBytes);
        }
        return res;*/

        // 从结果中取用户指定的某个key的value
        String name = Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("user_name")));
        System.out.println(name);
        return name;
    }

}

```

测试类  

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpYCiaWFQmyiaLPdvGSz2kId1eBMvBN9fuPmX0xXJ6BTCLtkgpjpCiazCpg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpDXu2icOq4Dj1iblTE1PQqSgic2lflRicgxebKjDuU6zA1QHcGo5EvzNhrg/640?wx_fmt=png)

```css
package com.wedoctor.demo.service;

import org.junit.jupiter.api.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import static org.junit.jupiter.api.Assertions.*;

@RunWith(SpringRunner.class)
@SpringBootTest
class UserServiceTest {
    @Autowired
    //@Qualifier("userInfo")
    UserService UserService;

    @Test
    public void test() throws Exception {
        UserService.getHbaseUserInfo("1");
    }
}
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpNEZJVBSib1Lf7DyzzVOyMBubzuiapwQfwaNjXgiaaosjpUic5lm4NbRvrg/640?wx_fmt=png)

接口访问

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLcx3icBl3QhuleLsVf3Io5YpmFEaFCgBDlsqSkYOC3Mfk0Rfiaa4g25msibnXIOGLcKbBR00V1bsrMBA/640?wx_fmt=png)

**10**

 **HBase 二级索引涉设计思想**

1

**为什么需要创建二级索引**

       HBase 对于多条件组合查询这种应用场景是非常不占优势的，甚至可以说就是其短板，一般情况下，我们有两种方式查询 Hbase 中的数据

-          通过 Rowkey 查询数据，Rowkey 里面会组合固定查询条件，但是需要把多组合查询的字段都拼接在 Rowkey 中，这是不可能的。  
-          通过 Scan 全部扫描符合条件的数据，这样的效率是非常低的

所以这时候我们就需要用建立二级索引的方法来解决这个问题

2

二级索引原理

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLexPQwxf7Kb1cLOkhdia4Y39f0MDl1iczTZbE7tSFpUMyv0mk5oAqNG3HhIIGvAqjXr8icEoZ24SdmGg/640?wx_fmt=png)

如上图所示，Hbase 表中的字段为 Rowkey，age，sex，username，phone，目前的需求是需要按照 age，sex，username，phone 随机组合查询符合条件的数据。  

这时候我们就需要用 ES 来建立二级索引了，原始数据存在 HBase 中，索引存在 ES 中，如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLexPQwxf7Kb1cLOkhdia4Y39Jiceh3KorRYpiauJvjtuDhXWeD3nhowaUNJXEblX2gtGQ2OV9zyQc1sg/640?wx_fmt=png)

原理流程  

1.  将原始数据存入 HBase
2.  将需要查询的条件字段及 Rowkey 存入 ES  
3.  客户端发送请求会根据组合查询条件去 ES 中查找到对应的 RowKey  
4.  ES 返回 RowKey 给客户端  
5.  客户端根据 ES 返回的结果 (RowKey) 查询 HBase 数据
6.  HBase 返回符合条件的数据给客户端

**11**

**phoenix 操作 HBase**

Phoenix，由 saleforce.com 开源的一个项目，后又捐给了 Apache。它相当于一个 Java 中间件，帮助开发者，像  
使用 jdbc 访问关系型数据库一样，访问 NoSql 数据库 HBase。  
Apache Phoenix 与其他 Hadoop 产品完全集成，如 Spark，Hive，Pig，Flume 和 MapReduce。

1

## 1.1 下载 pheonix

## [http://phoenix.apache.org/download.html](http://phoenix.apache.org/download.html)

注意：下载 Phoenix 的时候，请注意对应的版本，其中 4.14 版本可以运行在 HBase0.98、1.1、1.2、1.3、1.4 上。  
下载时也可以直接使用：  

```ruby
wget http://mirrors.shu.edu.cn/apache/phoenix/apache-phoenix-4.14.0-HBase-1.2/bin/apache-phoenix-4.14.0-HBase-1.2-bin.tar.gz
```

## 1.2 解压 pheonix

```css
tar -zxvf apache-phoenix-4.14.0-HBase-1.2-bin.tar.gz
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omyI4xsiaY4vib8dAeZcE5iciaXmqRXrZneQuMG8j3Z6trVotO6YVMUdgfSA/640?wx_fmt=png)

## 1.3 整合 phoenix 到 hbase

查看 Phoenix 下的所有的文件，将 phoenix-4.14.0-HBase-1.2-server.jar 拷贝到所有 HBase 节点 (包括 Hmaster 以及 HregionServer) 的 lib 目录下：

```sql
重启HBase：
bin/stop-hbase.sh
bin/start-hbase.sh
```

## 1.4 使用 phoenix SQL 命令行

进入 Phoenix 的安装包，执行：

    bin/sqlline.py bigdata1:2181

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omS9zwDn2g6M8bMMSnMpU8pH4TBaXDhcgDD0Qu3CfyuBiacRS0Ng71ulQ/640?wx_fmt=png)

### 1.4.1 创建表

在 Phoenix 终端下创建 us_population 表：  

```properties
>> CREATE TABLE IF NOT EXISTS us_population (
state CHAR(2) NOT NULL,
city VARCHAR NOT NULL,
population BIGINT
CONSTRAINT my_pk PRIMARY KEY (state, city));
```

使用! tables 查看创建的表：  

```ruby
>> !tables
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omCRs9bws6VBicptnnhwZ2eb0dsnf5gg7mCNMibf9MwH3rgMuTIlZWz2Bg/640?wx_fmt=png)

### 1.4.2 编辑并导入数据

在 Phoenix 目录下创建一个 data 目录，在 data 目录下创建：  
vi us_population.csv

```php
NY,New York,8143197
CA,Los Angeles,3844829
IL,Chicago,2842518
TX,Houston,2016582
PA,Philadelphia,1463281
AZ,Phoenix,1461575
TX,San Antonio,1256509
CA,San Diego,1255540
TX,Dallas,1213825
CA,San Jose,912332
```

执行 bin/psql.py data/us_population.csv 导入数据。

\# 除了导入数据外，还可以使用 Phoenix 的语法插入数据：upsert into us_population values('NY','NewYork',8143197);

### 1.4.3 查询数据

方式一：在 data 目录下创建 us_population_queries.sql 文件：

```sql
SELECT state as "State",count(city) as "City Count",sum(population) as "Population Sum"
FROM us_population
GROUP BY state
ORDER BY sum(population) DESC;
```

执行 bin/psql.py data/us_population_queries.sql 检索数据。

方式二：使用命令行终端  

```cs
bin/sqlline.py bigdata1:2181
>> select * from us_populcation;
```

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omRhJfK0MLibSnZ6rVS4SWp520RmICjDtsGIWh6wpz1rUhbgKJ8qdrdicg/640?wx_fmt=png)

2

## 2.1 下载 Squirrel-sql

[http://www.squirrelsql.org/#installation](http://www.squirrelsql.org/#installation)

## 2.2 设置 Squirrel-sql 连接 Phoenix

-   拷贝 Phoenix Client jar【phoenix-4.14.0-HBase-1.2-client.jar】到 Squirrel-sql 的 lib 目录；

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0ombY7gscPAfFGInZzsx8bMEh8dKXDLfsepZLiaaF2bicic0UFuL6B5bufXw/640?wx_fmt=png)

-   设置 Phoenix 连接的 Driver 信息，其中 localhost 为 zookeeper 所在的主机地址，填写一个即可。

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omHXlQSjbssFs2CtgYvWiczYpPjoPwrLBPEhR9uJxqW4ziaLBBTCViamlag/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0om90fH67WyYIKgQef0d3wE85E5GcjnZHBvOMM84Zb0GEZPqEjriaLbreg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omPwF5Z0VdeXphxehHS7sC8ia3HXmU44yGBcGmicEXKczx6gaTL8X3CN8Q/640?wx_fmt=png)

3

进入 Hbase 命令行终端 bin/hbase shell  
创建 Hbase 表'phoenix'：

\-- 创建 Hbase 表 Phoenix, 列族 info  

```nginx
create 'phoenix','info'
```

\-- 添加数据  

```nginx
put 'phoenix', 'row001','info:name','phoenix'
put 'phoenix', 'row002','info:name','hbase'
```

映射 HBase 表的方式有两种，一直是视图映射，一种是表映射。  
两者的区别就是对 HBase 的物理表有没有影响；  
删除 Phoenix 视图映射不会对 Hbase 的表造成影响；  
删除 Phoenix 表映射会将 Hbase 的表也删除；  
非必要情况下一般创建视图映射。

## 3.1 视图映射

在 Phoenix 下创建视图映射 HBase 表：

```sql
-- 创建视图关联映射Hbase 表
create view "phoenix" (
pk VARCHAR primary key,
"info"."name" VARCHAR
);
```

查询创建好的 Phoenix 视图：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omuO1ILmIYaLTKtTuiaric6q6vfWIqQrJJUmU1bGdfFRGGQP38LZJnvgxw/640?wx_fmt=png)

\-- 删除视图后，在 hbase shell 终端下查看 phoenix 依然存在  

```sql
drop view "phoenix";
```

## 3.2 表映射

在 Phoenix 下创建表映射 HBase 表：

\-- 创建表关联映射 Hbase 表, 4.10 以后 Phoenix 优化了列映射，COLUMN_ENCODED_BYTES=0 禁用列映射。  

```sql
create table "phoenix" (
pk VARCHAR primary key,
"info"."name" VARCHAR
) COLUMN_ENCODED_BYTES = 0;
```

查询数据：

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLd1uiaQF7GXwfaRNYsNfJ0omtLKhIWIoiavAORuuEwRlzoIEx4wYEvHBfGIzm3ia8upAdmyhBibw6pyibQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_gif/dXCnejTRMLfaPKxaibsZ7cCiaozWvibvo25R8yoqsvvTmiaG2PAMap0daZ8F31icEtXsianE5bA67SQ5Lh1fAbT9yLvA/640?wx_fmt=gif)

[2020 大数据面试题真题总结 (附答案)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485055&idx=1&sn=4e74131939bb203865fd6219df82516c&chksm=ea68ecb3dd1f65a5c0f55066118e1ea1de73b40b428ec52ae5589b72cca612c5e825a242b696&scene=21#wechat_redirect)  

[一文探究数据仓库体系 (2.7 万字建议收藏)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485691&idx=1&sn=d6cb1353031e07e4b02cd903d8b57911&chksm=ea68e237dd1f6b210f65f25ef42dabf4453d3bfa36fe8f33b149c0ff5329f77b9b792eef7882&scene=21#wechat_redirect)

[数据建模知多少？](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485443&idx=1&sn=8307b0922a3ee002446936f83153bcc6&chksm=ea68e2cfdd1f6bd96cd1de5fa46a3191c0a70e4e3ca570ed57c1c647abd624fcf8a0901f08cc&scene=21#wechat_redirect)  

如何写好一篇数据部门规范文档  

[如何优化整个数仓的执行时长 (比如 7 点所有任务跑完，如何优化到 5 点)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485234&idx=1&sn=8b90724bcc9250eae4d790853579d7e8&chksm=ea68edfedd1f64e83df7595e2d52f070a54004d73e69131cd23ed20458a662be9bb4aa92a1d9&scene=21#wechat_redirect)  

[从 0-1 建设数仓遇到什么问题？怎么解决的？](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485205&idx=1&sn=cb08ded2967531570be2d758f5e3c5f2&chksm=ea68edd9dd1f64cf9c8831fa9a7acdc2ca47e6d943fbda104356bb9067fd93c6c0fb9c942470&scene=21#wechat_redirect)

[Hive 调优，数据工程师成神之路](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485048&idx=1&sn=5fc1219f4947bea9743cd938cec510c7&chksm=ea68ecb4dd1f65a2df364d79272e0e472a394c5b13b5d55d848c89d9498ccb7ea78a933fbdea&scene=21#wechat_redirect)  

[数据质量那点事](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485039&idx=1&sn=140c3bc720da51765292fe3f5082fe38&chksm=ea68eca3dd1f65b5aef4d6f7ab0c33d3d3033bcc0eead1650be079687e0b4e898562bfe4d25b&scene=21#wechat_redirect)  

[简述元数据管理](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247485186&idx=1&sn=85fbe5703c56aa2dcfd2980fccbab4f6&chksm=ea68edcedd1f64d8e2d8c3da6b456fcaa4b105f2216a2bddb2393a7380498166225de5e855b4&scene=21#wechat_redirect)

[Sqoop or Datax](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484752&idx=1&sn=567442111447f2a7cac5379b694205f7&chksm=ea68ef9cdd1f668ad81e435e8c42a622f0ccfda773ff03485fad3cabeb1bd9aeea2e8cb40428&scene=21#wechat_redirect)  

[left join(on&where)](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484805&idx=1&sn=109966c92bb55de84287c78810970f81&chksm=ea68ef49dd1f665f6224fbef09fa0a11d9d8fc9a12a7302eb28c63bcf1798780c2f25653a51c&scene=21#wechat_redirect)

[大厂高频面试题 - 连续登录问题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484649&idx=1&sn=3070c0d82b45157b86885c2f6a9b4dd8&chksm=ea68ee25dd1f6733df5f3e734f06fd3bb4e07ce91fa129763994c67b4572f60e8afb274be7ce&scene=21#wechat_redirect)  

[朋友面试数据研发岗遇到的面试题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484533&idx=1&sn=e817ea44f364f00d44fbcd3b321cc2c3&chksm=ea68eeb9dd1f67af148952b86c7d4911f788f9943992f339d5804f261f8c55908f4237b86d64&scene=21#wechat_redirect)

[简单聊一聊大数据学习之路](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484074&idx=1&sn=f1b74ec65a9d0101cafb7741046e29f9&chksm=ea68e866dd1f61703304e4f01fb97d3b1f98aba6ea3fc5a9fb258b4c5e4e317d79edd7f15267&scene=21#wechat_redirect)

[朋友面试数据专家岗遇到的面试题](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484277&idx=1&sn=12568aaf7f50f259ff1fd1764dd17ad0&chksm=ea68e9b9dd1f60af84a49c9653849fb215b92566fc6ded9ce0d056d481cea2855654bc72fbd7&scene=21#wechat_redirect)

[HADOOP 快速入门](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484163&idx=1&sn=0741c076904f81133cb2524033c5993f&chksm=ea68e9cfdd1f60d9254ef6bd1128dacf0b379d7fb1409f356177e2b8a9577ea8319b7619c141&scene=21#wechat_redirect)

[数仓工程师的利器 - HIVE 详解](http://mp.weixin.qq.com/s?__biz=MzI2MDQzOTk3MQ==&mid=2247484172&idx=1&sn=8bdb336229dc8aed89edbe1ab9dce771&chksm=ea68e9c0dd1f60d6f1d864bef3c531d0f42163beb764d9d6d1a7febf15f6f541f9709a2fa80c&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/dXCnejTRMLeibmoxSpRXUevnVIz4RyV0AUrglXsIgLOkFMNuffPAh5lZbwOnSoRHbZqT12VyKibm0EmnTicXBSRTg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/156dHv6J3MmdaB1xcHT2Ug](https://mp.weixin.qq.com/s/156dHv6J3MmdaB1xcHT2Ug)
