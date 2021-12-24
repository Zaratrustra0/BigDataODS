# 九个最容易出错的 Hive sql 详解及使用注意事项
五分钟学大数据，致力于大数据技术研究，如果你有任何问题或建议，可添加底部小编微信或直接后台留言

**阅读本文小建议：本文适合细嚼慢咽，不要一目十行，不然会错过很多有价值的细节。**   

#### 前言

在进行数仓搭建和数据分析时最常用的就是 sql，其语法简洁明了，易于理解，目前大数据领域的几大主流框架全部都支持 sql 语法，包括 hive，spark，flink 等，所以 sql 在大数据领域有着不可替代的作用，需要我们重点掌握。

在使用 sql 时如果不熟悉或不仔细，那么在进行查询分析时极容易出错，接下来我们就来看下几个容易出错的 sql 语句及使用注意事项。

#### 正文开始

#### 1. decimal

hive 除了支持 int,double,string 等常用类型，也支持 decimal 类型，用于在数据库中存储精确的数值，常用在表示金额的字段上

**注意事项：** 

如：decimal(11,2) 代表最多有 11 位数字，其中后 2 位是小数，整数部分是 9 位；   
如果**整数部分超过 9 位，则这个字段就会变成 null，如果整数部分不超过 9 位，则原字段显示**；   
如果**小数部分不足 2 位，则后面用 0 补齐两位，如果小数部分超过两位，则超出部分四舍五入**；   
也可直接写 decimal，后面不指定位数，默认是 decimal(10,0) 整数 10 位，没有小数

#### 2. location

    表创建的时候可以用 location 指定一个文件或者文件夹create  table stu(id int ,name string)  location '/user/stu2';

**注意事项：** 

创建表时使用 location，  
当**指定文件夹时，hive 会加载文件夹下的所有文件，当表中无分区时，这个文件夹下不能再有文件夹，否则报错。**         
当表是分区表时，比如 partitioned by (day string)， 则这个文件夹下的每一个文件夹就是一个分区，且文件夹名为 day=20201123  
这种格式，然后使用：**msck  repair   table  score**; 修复表结构，成功之后即可看到数据已经全部加载到表当中去了

#### 3. load data 和 load data local

    从hdfs上加载文件load data inpath '/hivedatas/techer.csv' into table techer;从本地系统加载文件load data local inpath '/user/test/techer.csv' into table techer;

**注意事项：** 

1.  使用 load data local 表示**从本地文件系统加载，文件会拷贝到 hdfs 上**
2.  使用 load data 表示**从 hdfs 文件系统加载，文件会直接移动到 hive 相关目录下**，注意不是拷贝过去，因为 hive 认为 hdfs 文件已经有 3 副本了，没必要再次拷贝了
3.  如果表是分区表，load 时不指定分区会报错  
4.  如果加载相同文件名的文件，会被自动重命名

#### 4. drop 和 truncate

    删除表操作drop table score1;清空表操作truncate table score2;

**注意事项：** 

如果 **hdfs 开启了回收站，drop 删除的表数据是可以从回收站恢复的**，表结构恢复不了，需要自己重新创建；**truncate 清空的表是不进回收站的，所以无法恢复 truncate 清空的表。**     
所以 truncate 一定慎用，一旦清空除物理恢复外将无力回天

#### 5. join 连接

    INNER JOIN 内连接：只有进行连接的两个表中都存在与连接条件相匹配的数据才会被保留下来select * from techer t [inner] join course c on t.t_id = c.t_id; -- inner 可省略LEFT OUTER JOIN 左外连接：左边所有数据会被返回，右边符合条件的被返回select * from techer t left join course c on t.t_id = c.t_id; -- outer可省略RIGHT OUTER JOIN 右外连接：右边所有数据会被返回，左边符合条件的被返回、select * from techer t right join course c on t.t_id = c.t_id;FULL OUTER JOIN 满外(全外)连接: 将会返回所有表中符合条件的所有记录。如果任一表的指定字段没有符合条件的值的话，那么就使用NULL值替代。SELECT * FROM techer t FULL JOIN course c ON t.t_id = c.t_id ;

**注意事项：** 

1.  hive2 版本已经支持不等值连接，就是 **join on 条件后面可以使用大于小于符号; 并且也支持 join on 条件后跟 or** (早前版本 on 后只支持 = 和 and，不支持> &lt; 和 or)
2.  如 hive 执行引擎使用 MapReduce，一个 join 就会启动一个 job，一条 sql 语句中如有多个 join，则会启动多个 job

**注意**：表之间用逗号 (,) 连接和 inner join 是一样的，例：

    select tableA.id, tableB.name from tableA , tableB where tableA.id=tableB.id;   和   select tableA.id, tableB.name from tableA join tableB on tableA.id=tableB.id;   

**它们的执行效率没有区别，只是书写方式不同**，用逗号是 sql 89 标准，join 是 sql 92 标准。用逗号连接后面过滤条件用 where ，用 join 连接后面过滤条件是 on。

#### 6. left semi join

    为什么把这个单独拿出来说，因为它和其他的 join 语句不太一样，这个语句的作用和 in/exists 作用是一样的，是 in/exists 更高效的实现SELECT A.* FROM A where id in (select id from B)SELECT A.* FROM A left semi join B ON A.id=B.id上述两个 sql 语句执行结果完全一样，只不过第二个执行效率高

**注意事项：** 

1.  left semi join 的限制是：join 子句中右边的表**只能在 on 子句中设置过滤条件**，在 where 子句、select 子句或其他地方过滤都不行。
2.  left semi join 中 on 后面的过滤条件**只能是等于号**，不能是其他的。
3.  left semi join 是只传递表的 join key 给 map 阶段，因此 left semi join 中最后 select 的**结果只许出现左表**。
4.  因为 left semi join 是 in(keySet) 的关系，遇到**右表重复记录，左表会跳过**

#### 7. 聚合函数中 null 值

    hive支持 count(),max(),min(),sum(),avg() 等常用的聚合函数

**注意事项：** 

**聚合操作时要注意 null 值**：

count(\*) 包含 null 值，统计所有行数；   
count(id) 不包含 id 为 null 的值；   
min **求最小值是不包含 null**，除非所有值都是 null；   
avg **求平均值也是不包含 null**。

**以上需要特别注意，null 值最容易导致算出错误的结果**

#### 8. 运算符中 null 值

    hive 中支持常用的算术运算符(+,-,*,/)  比较运算符(>, <, =)逻辑运算符(in, not in)以上运算符计算时要特别注意 null 值

**注意事项：** 

1.  **每行中的列字段相加或相减，如果含有 null 值，则结果为 null**    
    例：有一张商品表（product）

| id  | price | dis_amount |
| --- | ----- | ---------- |
| 1   | 100   | 20         |
| 2   | 120   | null       |

各字段含义：id (商品 id)、price (价格)、dis_amount (优惠金额)

我想算**每个商品优惠后实际的价格**，sql 如下：

    select id, price - dis_amount as real_amount from product;

得到结果如下：

| id  | real_amount |
| --- | ----------- |
| 1   | 80          |
| 2   | null        |

id=2 的商品价格为 null，结果是错误的。

我们可以**对 null 值进行处理**，sql 如下：

    select id, price - coalesce(dis_amount,0) as real_amount from product;使用 coalesce 函数进行 null 值处理下，得到的结果就是准确的coalesce 函数是返回第一个不为空的值如上sql：如果dis_amount不为空，则返回dis_amount，如果为空，则返回0

2.  **小于是不包含 null 值**，如 id \\&lt; 10；是不包含 id 为 null 值的。
3.  **not in 是不包含 null 值的**，如 city not in ('北京','上海')，这个条件得出的结果是 city 中不包含 北京，上海和 null 的城市。

#### 9. and 和 or

在 sql 语句的过滤条件或运算中，如果有多个条件或多个运算，我们都会考虑优先级，如乘除优先级高于加减，乘除或者加减它们之间优先级平等，谁在前就先算谁。那 and 和 or 呢，看似 and 和 or 优先级平等，谁在前先算谁，但是，**and 的优先级高于 or**。

**注意事项：** 

例：   
还是一张商品表（product）

| id  | classify | price |
| --- | -------- | ----- |
| 1   | 电器       | 70    |
| 2   | 电器       | 130   |
| 3   | 电器       | 80    |
| 4   | 家具       | 150   |
| 5   | 家具       | 60    |
| 6   | 食品       | 120   |

我想要统计下电器或者家具这两类中价格大于 100 的商品，sql 如下：

    select * from product where classify = '电器' or classify = '家具' and price>100

得到结果

| id  | classify | price |
| --- | -------- | ----- |
| 1   | 电器       | 70    |
| 2   | 电器       | 130   |
| 3   | 电器       | 80    |
| 4   | 家具       | 150   |

结果是错误的，把所有的电器类型都查询出来了，原因就是 and 优先级高于 or，上面的 sql 语句实际执行的是，先找出 classify = '家具' and price>100 的，然后在找出 classify = '电器' 的

正确的 sql 就是加个括号，先计算括号里面的：

    select * from product where (classify = '电器' or classify = '家具') and price>100

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)

文: 园陌

扫码收获更多技术

![](https://mmbiz.qpic.cn/mmbiz_gif/c6gqmhWiafyobvdkCN1GNh88geK5jbWxJ16lNcEJicicNgoRJDre75yTz7LicJBqaZMmNrojibsmOxWcia5Dk7qKC0eg/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ZubDbBye0zFXFP8Nqo5j6gk7icneiciaibL6PKich0rZzoNRPrlEJGZCqUyC8Uia5GLY9CJj97iaSphibTruU79jxE9uLQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_gif/TN05MmJLxMrNiboff6rhmPUYUNXdWTXZfCTse4EDYOEoRVrlpgoZF6t00NhnrdePKAibI1zaap7wibR0iclbJZFS1A/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/Df9ts2-W0y08HhPX3g99AA](https://mp.weixin.qq.com/s/Df9ts2-W0y08HhPX3g99AA)
