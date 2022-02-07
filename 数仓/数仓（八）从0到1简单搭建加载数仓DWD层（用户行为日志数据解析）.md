# 数仓（八）从0到1简单搭建加载数仓DWD层（用户行为日志数据解析）
[数仓（一）简介数仓，OLTP 和 OLAP](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503438&idx=1&sn=3d249b38099cdb4f3d17a7d4676f93de&chksm=f9ed3766ce9abe70d7ae724c1a83ad7669f19c4148b16bfc16f03dbc23f77b23465ca845a47d&scene=21#wechat_redirect)  

[数仓（二）关系建模和维度建模](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503465&idx=1&sn=fee932bd60e96b2814f7a257399d8170&chksm=f9ed3741ce9abe57203c7cee8a0207d9c48b53cf261aa77ff6dd9b2d65ec85543f985f01acb7&scene=21#wechat_redirect)  

[数仓（三）简析阿里、美团、网易、恒丰银行、马蜂窝 5 家数仓分层架构](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503466&idx=2&sn=aa9ab1dd24fccdb41d9e805a937c12c4&chksm=f9ed3742ce9abe54f73b86a4db0125fe36dbc8c7889cd7452456b1ed058e2c13a9f4e26eb6d7&scene=21#wechat_redirect)  

[数仓（四）数据仓库分层](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503496&idx=2&sn=3f0c6f34fe307932efeaf4c3f19b5503&chksm=f9ed37a0ce9abeb60a5417821d94f7d1bfb9ef26844b0cf44aecf17aa5e901a91f85cda7ab91&scene=21#wechat_redirect)  

[数仓 (五) 元数据管理系统解析](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503532&idx=1&sn=c996c431e25199496e464270cf9aa8eb&chksm=f9ed3784ce9abe92a7b09b535b08e8fabacdc51807e801dc4bb7806de36288c39d7ac4111e03&scene=21#wechat_redirect)  

[数仓（六）从 0 到 1 简单搭建数仓 ODS 层 (埋点日志 + 业务数据)](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503555&idx=1&sn=9718725ec437cc7e9fe78167193bb958&chksm=f9ed37ebce9abefd9c68cdf5905be86e9718bab479e1a83868d17f6159b512b64052ecff1818&scene=21#wechat_redirect)  

[数仓（七）从 0 到 1 简单搭建加载数仓 DIM 层以及拉链表处理](http://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247503579&idx=2&sn=4eef68828d5b616631dff19319ec2b99&chksm=f9ed37f3ce9abee5023f1385c608ad67cd10b7a026e7009416c8bfa0f401e8a207a01a901395&scene=21#wechat_redirect)  

上一节我们讲解了数仓 DIM 维度层的搭建和使用，并且讲解了拉链表的概念和使用。这节我们讲解 DWD 层关于用户行为日志数据的搭建和使用。下一次分享 DWD 层关于业务数据的搭建和使用。

**一、DWD 层用户行为日志解析结构**

DWD 层是对用户的日志行为进行解析，以及对业务数据采用维度模型的方式重新建模（维度退化）。本节我们先来回顾一下用户行为日志的结构。

**1、前端埋点日志信息**

前端埋点日志信息都是 JSON 格式形式，主要包括两方面：

（1）启动日志；（2）事件日志；

我们之前已经把前端埋点的日志信息，写到 ODS 层 ods_log 表了，传入的参数是一个 String 类型字符串即一条日志信息一个 String 类型字符串。  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiaclb5aVu6PjAsYuXsb1OJ6ENJ6EckoXwTXIRDt3oTjSYFcmnr29gAY7Hza3qog4fKRrrrfYudNkw/640?wx_fmt=png)

**2、日志解析思路**

我们根据 ODS 层日志数据内容来解析到 DWD 层分为 5 个表，也可以根据启动日志和事件日志来解析到 DWD 层分为 2 个表。前者是根据内容抽象来解析，颗粒度更细，一般大公司使用。后者比较简单就是根据数据的类型来解析。我们这里采用第一种方式，把页面日志和启动日志信息的数据需要装载到 DWD 层里面的五张表。所以 DWD 层就是解析这五张表。  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAwdsiarHekUX8Uue23CaK6wSG5N53ZOiaXklwibbCpf7bictMyhxkyHCeuw/640?wx_fmt=png)

我们下面来依次解析每一张表。

**二、DWD 层 - 启动日志表**

启动日志表中每一行数据对应一个启动记录，一个启动记录应该包含日志中的公共信息和启动信息。

常规解析思路是：可以先将所有包含 start 字段的日志过滤出来，然后使用 get_json_object 函数解析每个字段。

**1、启动日志表结构**

-   **分区：** 

        dt = 2020-06-14

-   **过滤条件：** 

        利用 get_json_object 函数，解析 start 内容不为空说明是启动日志信息；

-   **范围**

包括：公共信息 common、启动信息 start、启动 app 时间 ts;

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAywO9BHlYqoOXhqexE70tnGZxgZGarIhBcowEHEdZUENhSUHVnyUFXg/640?wx_fmt=png)

**2、创建表结构**

```sql
DROP TABLE IF EXISTS dwd_start_log;
CREATE EXTERNAL TABLE dwd_start_log(
    `area_code` STRING COMMENT '地区编码',
    `brand` STRING COMMENT '手机品牌',
    `channel` STRING COMMENT '渠道',
    `is_new` STRING COMMENT '是否首次启动',
    `model` STRING COMMENT '手机型号',
    `mid_id` STRING COMMENT '设备id',
    `os` STRING COMMENT '操作系统',
    `user_id` STRING COMMENT '会员id',
    `version_code` STRING COMMENT 'app版本号',
    `entry` STRING COMMENT 'icon手机图标 notice 通知 install 安装后启动',
    `loading_time` BIGINT COMMENT '启动加载时间',
    `open_ad_id` STRING COMMENT '广告页ID ',
    `open_ad_ms` BIGINT COMMENT '广告总共播放时间',
    `open_ad_skip_ms` BIGINT COMMENT '用户跳过广告时点',
    `ts` BIGINT COMMENT '时间'
) COMMENT '启动日志表'
PARTITIONED BY (`dt` STRING) -- 按照时间创建分区
STORED AS PARQUET -- 采用parquet列式存储
LOCATION '/warehouse/gmall/dwd/dwd_start_log' -- 指定在HDFS上存储位置
TBLPROPERTIES('parquet.compression'='lzo') -- 采用LZO压缩;
```

**3、装载数据**

首日和每日加载数据分区都是一样的策略，每天 DWD 层从 ODS 层获取数据后加载。  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAg18aRh98icIC2Bwrm2nMZiaGw8WZ58jvnYgEMnuyg5ss870s2ylCkMOA/640?wx_fmt=png)

**SQL 实现**  

```cs
insert overwrite table dwd_start_log partition(dt='2020-06-14')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.is_new'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.ts')
from ods_log
where dt='2020-06-14'
and get_json_object(line,'$.start') is not null;
```

**注意：** 这里 hive 需要使用 HiveInputFormat，而不是 CombineHiveInputFormat  

```sql
SET hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
```

后者不能识别 lzo.index 的索引文件，会把索引文件当做普通文件来处理，并且导致 lzo 文件无法切片。而 ODS 层数据我们处理的时候是带 lzo 索引文件的。

**三、DWD 层 - 页面日志表**

页面日志表和启动日志表的处理逻辑一样。页面日志表中每一行数据对应一个页面的访问记录，一个页面访问记录应该包含日志中的公共信息和页面信息。

常规解析思路是：也是先将所有包含 page 字段的日志过滤出来，然后使用 get_json_object 函数解析每个字段。

**1、页面日志表结构**

-   **分区：** 

        dt = 2020-06-14

-   **过滤条件：** 

        利用 get_json_object 函数，解析 page 内容不为空说明是页面日志信息；

-   **范围**

包括：公共信息 common、页面信息 page、启动 app 时间 ts;

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LATCvSZ8ur16SIbCPpyc7fv50AYoyy78ibO90RdS7uZiaBCjD7jAUtibG7A/640?wx_fmt=png)

**3、创建表结构**

```sql
drop table if exists dwd_page_log;
CREATE EXTERNAL TABLE dwd_page_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `during_time` bigint COMMENT '持续时间毫秒',
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面ID ',
    `source_type` string COMMENT '来源类型', 
    `ts` bigint
) COMMENT '页面日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_page_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

**4、装载数据**

和启动日志表一样，首日和每日加载数据分区都是一样的策略，每天 DWD 层从 ODS 层获取数据后加载。

```sql
insert overwrite table dwd_page_log partition(dt='2020-06-14')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.ts')
from ods_log
where dt='2020-06-14'
and get_json_object(line,'$.page') is not null;
```

**四、DWD 层 - 动作日志表**

动作日志表，比前面启动日志和页面信息表要复杂的多。动作日志表中每行数据对应的是用户的一个动作记录，一个动作记录应当包含公共信息、页面信息以及动作信息。

常规解析思路是：先将包含 action 字段的日志过滤出来，然后通过 UDF、**UDTF 函数**（用户定义表生成函数 user-defined table-generating function）将 action 数组 “炸裂”（类似于 explode 函数的效果），然后使用 get_json_object 函数解析每个字段。

**1、页面日志表结构**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LA3ZNVEIKILRZmfTtX8lPHjdrlL5JiaO2U5sc5M13yx14a4iaQJRT6wlMQ/640?wx_fmt=png)

根据图示我们可以看到：  

(1) 需要自定义创建 UDTF 函数

来完成对 actions 动作数组的 “炸裂”，实现“一行输入，多行输出” 的需求。即输入 JSON 数组字符串 actions, 输出每一个 JSON 数组元素 action。

(2) 然后通过 get_json_object(action,$action 元素字段) 获取信息；

-   **分区：** 

        dt = 2020-06-14

-   **过滤条件：** 

      利用 get_json_object 函数，解析 actions 内容不为空；

-   **范围**

包括：公共信息 common、页面信息 page、动作信息、启动 app 时间 ts;

**2、创建表结构**

```sql
drop table if exists dwd_action_log;
CREATE EXTERNAL TABLE dwd_action_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `during_time` bigint COMMENT '持续时间毫秒', 
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面id ',
    `source_type` string COMMENT '来源类型', 
    `action_id` string COMMENT '动作id',
    `item` string COMMENT '目标id ',
    `item_type` string COMMENT '目标类型', 
    `ts` bigint COMMENT '时间'
) COMMENT '动作日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_action_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

**3、创建 UDTF 函数**

我们通过 java 代码来实现功能，并且通过 MAVEN 来打成 jar。然后上传到 HDFS 集群，给 hive 做关联调用。当然也可以使用 hive 自带的 UDTF 函数含来完成解析。但是后者必须是在 hive 的客户端完成，必须带 hive 库名，不够灵活。

**3.1、编写 java 代码实现 UDTF 功能**

**1、pom.xml 文件中 hive 版本依赖**

我这里 hive 的版本是 3.1.2

```xml
<dependencies>
    <!--添加hive依赖-->
    <dependency>
        <groupId>org.apache.hive</groupId>
        <artifactId>hive-exec</artifactId>
        <version>3.1.2</version>
    </dependency>
</dependencies>
```

**2、项目工程结构**  

这个比较简单。

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAUibG24BLc7a5UGVIDQMvkNgLUjgJsVTZzQU3KSts0Us9am5vEvFpXSQ/640?wx_fmt=png)

**3、ExplodeJSONArray 类编写**  

**思路是：** 

(1)ExplodeJSONArray 继承 hive 的 GenericUDTF 类；

(2) 实现 initialize、process、close 三个方法；

initialize 方法：主要对解析的变量名、变量类型做校验

process 方法：完成对 “JSON 数组炸裂成单一元素” 的功能。这里是一行 in，多行 out，并且 forward 到元素里面

```java
package com.qiusheng.hive.udtf;

import org.apache.hadoop.hive.ql.exec.MapredContext;
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
import org.json.JSONArray;

import java.util.ArrayList;
import java.util.List;

/**
 * @author qiusheng
 * @date 2021年11月04日
 *
 */
public class ExplodeJSONArray extends GenericUDTF
{

    /**
     *  1、第一步需要自定义的类继承GenericUDTF类
     *      并且重写initialize方法；
     *      返回StructObjectInspector对象
     *
     * @qiusheng
     * @param argOIs
     * @return
     * @throws UDFArgumentException
     */
    @Override
    public StructObjectInspector initialize(StructObjectInspector argOIs) throws UDFArgumentException
{
        //1、判断参数的合法性argOI实际就是actions
        //(1)判断参数的个数（需要传递一个参数）

        //如果传递的参数个数不是1个，则抛异常；
        if(argOIs.getAllStructFieldRefs().size() != 1)
        {
            throw new UDFArgumentException("参数个数错误！只需要1个参数......");

        }
        //(2)判断传递的参数类型，必须是String类型
        //如果不是string类型，则抛异常；
        if(!"String".equals(argOIs.getAllStructFieldRefs().get(0).getFieldObjectInspector().getTypeName()))
        {
            throw new UDFArgumentException("参数类型不对！应该是String类型......");
        }


        //2、返回StructObjectInspector对象
        //第一个参数:变量名，类型是List<String>数组;
        //第二个参数:检验变量，类型是List<ObjectInspector>;

        //定义返回值名称
        List<String> fieldNames = new ArrayList<String>();

        //定义返回值类型
        List<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();

        //把items添加到返回值名称
        fieldNames.add("items");

        //调用检验方法对items
        fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);

        //3、返回类型
        return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames,fieldOIs);
    }

    /**
     * 2、第二步重写process方法
     *
     * function：炸裂功能
     * @author qiusheng
     * @param objects
     * @throws HiveException
     */
    @Override
    public void process(Object[] objects) throws HiveException
{
        //1、获取传入的数据就是actions
        String jsonArray = objects.toString();

        //2、把传入的数据（string）类型转化为JSON数组JSONArray类型
        JSONArray actions = new JSONArray(jsonArray);

        //3、循环取出JSONArray对象中的JSON，并且写出来
        for (int i = 0;i < actions.length();i++)
        {
            //定义一个字符串数组，长度就是1
            String[] result = new String[1];
            //getString(i)把元素取出来，添加到这个数组中
            result[0] = actions.getString(i);
            //写到String数组里面
            forward(result);
        }
    }

    /**
     * @author qiusheng
     * @throws HiveException
     */
    @Override
    public void close() throws HiveException
{

  }
}
```

**3.2、通过 maven 打成 jar 包**  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAIvJZ09uZRCjC3R9V7LZLpp3GoibvmkiaN5IMjgLOMF1AGelQTZ5lGjWA/640?wx_fmt=png)

**3.3、上传 jar 包到 hadoop 集群路径下**  

(1) 先上传 jar 包到 CentOS 集群的 node1 节点

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiaclb5aVu6PjAsYuXsb1OJ6q7h30icnrtcMptKibT8f5JscicyebxBBOVrVXrryIuQEot9hjnQ5WUZLQ/640?wx_fmt=png)

(2) 上传到 HDFS 集群系统

先创建目录 functions/hive/jars

```ruby
[qiusheng@node01 module]$ hadoop fs -mkdir -p /functions/hive/jars
```

检查查看目录是否已经创建；  

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiaclb5aVu6PjAsYuXsb1OJ6hdmv1U0bDtTlg2tibAaic89LNcbkRQySMPiaicVL4GjaIZ9R3ZKND13kKw/640?wx_fmt=png)

上传 jar 包到这个目录 / functions/hive/jars

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiaclb5aVu6PjAsYuXsb1OJ6sGUBsv4Xy19FexNiaIR6ic1kFicptGcGpD9lmL7D5cmuaQMnLr3jRgUCQ/640?wx_fmt=png)

**3.4、创建 UDTF 和 jar 包关联**  

在 hive 客户端, 创建 function 关联 jar 包  

```sql
hive (gmall)>
create function explode_json_array as 'com.qiusheng.hive.udtf.ExplodeJSONArray' using jar 'hdfs://mode01:8020/functions/hive/jars/HiveDWDActionLog-1.0-SNAPSHOT.jar';
```

**4、装载数据**  

所需要的字段都拼接齐全了，我们来写装载语句。

```sql
insert overwrite table dwd_action_log partition(dt='2020-06-14')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(action,'$.action_id'),
    get_json_object(action,'$.item'),
    get_json_object(action,'$.item_type'),
    get_json_object(action,'$.ts')
from ods_log lateral view explode_json_array(get_json_object(line,'$.actions')) tmp as action
where dt='2020-06-14'
and get_json_object(line,'$.actions') is not null;
```

这里 lateral view 意思是：为原始表的每行调用 UDTF，UTDF 会把一行拆分成一或者多行，lateral view 再把结果组合，产生一个支持别名表的虚拟表 action。

```cs
from ods_log lateral view explode_json_array(get_json_object(line,'$.actions')) tmp as action
```

这样我们动作日志表就完成了。

**五、DWD 层 - 曝光日志表**

曝光日志表和动作日志表，解析一样。曝光日志表中每行数据对应一个曝光记录，一个曝光记录应当包含公共信息、页面信息以及曝光信息。

常规思路：先将包含 display 字段的日志过滤出来，然后通过 UDTF 函数，将 display 数组 “炸开”（类似于 explode 函数的效果），然后使用 get_json_object 函数解析每个字段。

**1、页面日志表结构**

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbgGHYhR7gJ6GCA44eWxj3LAjXX2ibNL7BVeHLwRUIEngeKwqDBkhbVg06qPZvOXqHJnNSF7KXeLfiaw/640?wx_fmt=png)

**后面建表、做 UDTF、一系列操作、装载数据和动作日志表一样。留给读者自行实践一下。** 

**六、DWD 层 - 错误日志表**

动作日志表，比前面启动日志和页面信息表要复杂一些。错误日志表中每行数据对应一个错误记录，为方便定位错误，一个错误记录应当包含与之对应的公共信息、页面信息、曝光信息、动作信息、启动信息以及错误信息。先将包含 err 字段的日志过滤出来，然后使用 get_json_object 函数解析所有字段。

\***\*1、页面日志表结构 \*\***

![](https://mmbiz.qpic.cn/mmbiz_png/EGZ1ibxgFrbiaclb5aVu6PjAsYuXsb1OJ67FepZPOZYqXmd8sXvlSFm0XqwQ4C0pybHM7xJbToPVxzTxFkhtmYhw/640?wx_fmt=png)

含有 2 个 actions 、displays 数组的 UDTF 炸裂。  

**\*\*\*\***2、创建表结构**\*\*\*\***

```sql
drop table if exists dwd_error_log;
CREATE EXTERNAL TABLE dwd_error_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面ID ',
    `source_type` string COMMENT '来源类型', 
    `entry` string COMMENT ' icon手机图标  notice 通知 install 安装后启动',
    `loading_time` string COMMENT '启动加载时间',
    `open_ad_id` string COMMENT '广告页ID ',
    `open_ad_ms` string COMMENT '广告总共播放时间', 
    `open_ad_skip_ms` string COMMENT '用户跳过广告时点',
    `actions` string COMMENT '动作',
    `displays` string COMMENT '曝光',
    `ts` string COMMENT '时间',
    `error_code` string COMMENT '错误码',
    `msg` string COMMENT '错误信息'
) COMMENT '错误日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_error_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

**注意：**   

对动作数组和曝光数组做处理，如需分析错误与单个动作或曝光的关联，可先使用 explode_json_array 函数将数组 “炸开”，再使用 get_json_object 函数获取具体字

****\*\*\*\*****3、装载表数据****\*\*\*\*****

```sql
insert overwrite table dwd_error_log partition(dt='2020-06-14')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.actions'),
    get_json_object(line,'$.displays'),
    get_json_object(line,'$.ts'),
    get_json_object(line,'$.err.error_code'),
    get_json_object(line,'$.err.msg')
from ods_log 
where dt='2020-06-14'
and get_json_object(line,'$.err') is not null;
```

这样我们就完成了最后一张表从 ODS 层到 DWD 层的解析。

![](https://mmbiz.qpic.cn/mmbiz_png/TwK74MzofXd78W49nBaME6TkGc8gv8DBzMJvytIYy9Dibfsl7qq5ibATfYh9BN1xQO5qU1OejK3Gic6dfl8iafXwGg/640?wx_fmt=png) 
 [https://mp.weixin.qq.com/s/7qtjPFB0OpqMm3AdYB7Www](https://mp.weixin.qq.com/s/7qtjPFB0OpqMm3AdYB7Www)
