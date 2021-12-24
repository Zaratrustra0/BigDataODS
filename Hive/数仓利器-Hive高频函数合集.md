# 数仓利器-Hive高频函数合集
点击上方蓝字关注我们, 更多惊喜等着你

每日一句

错误的开始，未必不能走到完美的结束，人生没有什么事是一定的。都是在碰，在等，在慢慢寻找。——《流苏与娜拉》

前言数据准备数据集建表语句窗口函数 row_number：使用频率 ★★★★★rank ：使用频率 ★★★★dense_rank：使用频率 ★★★★rank/dense_rank/row_number 对比 first_value：使用频率 ★★★last_value：使用频率 ★lead：使用频率 ★★lag：使用频率 ★★集合相关 collect_set：使用频率 ★★★★★collect_list：使用频率 ★★★★★sort_array：使用频率 ★★★URL 相关 parse_url：使用频率 ★★★★reflect：使用频率 ★★JSON 相关 get_json_object：使用频率 ★★★★★列转行相关 explode：使用频率 ★★★★★Cube 相关 GROUPING SETS：使用频率 ★字符相关 concat：使用频率 ★★★★★concat_ws：使用频率 ★★★★★instr：使用频率 ★★★★length：使用频率 ★★★★★size: 使用频率 ★★★★★trim：使用频率 ★★★★★regexp_replace：使用频率 ★★★★★regexp_extract：使用频率 ★★★★substring_index：使用频率 ★★条件判断 if：使用频率 ★★★★★case when ：使用频率 ★★★★★coalesce：使用频率 ★★★★★数值相关 round：使用频率 ★★ceil：使用频率 ★★★floor：使用频率 ★★★hex：使用频率 ★时间相关 (比较简单)from_unxitime：使用频率 ★★★★★unix_timestamp：使用频率 ★★★★★to_date：使用频率 ★★★★★year：使用频率 ★★★★★month：使用频率 ★★★★★day：使用频率 ★★★★★date_add：使用频率 ★★★★★date_sub：使用频率 ★★★★★

### 前言

> Hive 是数仓建设使用频率最高的一项技术，基于各种业务需求，使用功能函数会为我们的开发提高了很多效率。本篇是基于笔者在日常开发中使用频率较高的函数做一次总结 (同时也会给出一些业务场景帮助读者理解)，同时也是面试中经常会被问到的函数。
>
> 如有遗漏，欢迎各位读者一起交流沟通并补充进来～；
>
> 另关注公众号后台回复 “大数据” 可获取更多关于大数据资料

### 数据准备

#### 数据集

     1user1,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,10,2020-09-12 02:20:02,2020-09-12 2user1,https://blog.csdn.net/qq_28680977/article/details/108298276?k1=v1&k2=v2#Ref1,2,2020-09-11 11:20:12,2020-09-11 3user1,https://blog.csdn.net/qq_28680977/article/details/108295053?k1=v1&k2=v2#Ref1,4,2020-09-10 08:19:22,2020-09-10 4user1,https://blog.csdn.net/qq_28680977/article/details/108460523?k1=v1&k2=v2#Ref1,5,2020-08-12 19:20:22,2020-08-12 5user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,29,2020-04-04 12:23:22,2020-04-04 6user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,30,2020-05-15 12:34:23,2020-05-15 7user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,30,2020-05-15 13:34:23,2020-05-15 8user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,19,2020-05-16 19:03:32,2020-05-16 9user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,10,2020-05-17 06:20:22,2020-05-1710user3,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,43,2020-04-12 08:02:22,2020-04-1211user3,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,5,2020-08-02 08:10:22,2020-08-0212user3,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,6,2020-08-02 10:10:22,2020-08-0213user3,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,50,2020-08-12 12:23:22,2020-08-1214user4,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,10,2020-04-12 11:20:22,2020-04-1215user4,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,30,2020-03-12 10:20:22,2020-03-1216user4,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,20,2020-02-12 20:26:43,2020-02-1217user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,10,2020-04-12 19:12:36,2020-04-1218user2,https://blog.csdn.net/qq_28680977/article/details/108161655?k1=v1&k2=v2#Ref1,40,2020-05-12 18:24:31,2020-05-12

#### 建表语句

     1create table wedw_tmp.tmp_url_info( 2 user_id string comment "用户id", 3 visit_url string comment "访问url", 4 visit_cnt int comment "浏览次数/pv", 5visit_time timestamp comment "浏览时间", 6 visit_date string comment "浏览日期" 7) 8row format delimited 9fields terminated by ','10stored as textfile;

### 窗口函数

#### row_number：使用频率 ★★★★★

> row_number 函数通常用于分组统计组内的排名，然后进行后续的逻辑处理。
>
> **注意：当遇到相同排名的时候，不会生成同样的序号，且中间不会空位**
>
> **该函数经常被大厂问到，可以参考**

     1-- 统计每个用户每天最近一次访问记录 2select  3  user_id, 4  visit_time, 5  visit_cnt 6from  7( 8  select 9    *,10   row_number() over(partition by user_id,visit_date order by visit_time desc) as rank11  from wedw_tmp.tmp_url_info12)t13where rank=114order by user_id,visit_time15+----------+------------------------+------------+--+16| user_id  |       visit_time       | visit_cnt  |17+----------+------------------------+------------+--+18| user1    | 2020-08-12 19:20:22.0  | 5          |19| user1    | 2020-09-10 08:19:22.0  | 4          |20| user1    | 2020-09-11 11:20:12.0  | 2          |21| user1    | 2020-09-12 02:20:02.0  | 10         |22| user2    | 2020-04-04 12:23:22.0  | 29         |23| user2    | 2020-04-12 19:12:36.0  | 10         |24| user2    | 2020-05-12 18:24:31.0  | 40         |25| user2    | 2020-05-15 13:34:23.0  | 30         |  --该用户同一天访问了多次,但只取了最新一次访问记录26| user2    | 2020-05-16 19:03:32.0  | 19         |27| user2    | 2020-05-17 06:20:22.0  | 10         |28| user3    | 2020-04-12 08:02:22.0  | 43         |29| user3    | 2020-08-02 10:10:22.0  | 6          |30| user3    | 2020-08-12 12:23:22.0  | 50         |31| user4    | 2020-02-12 20:26:43.0  | 20         |32| user4    | 2020-03-12 10:20:22.0  | 30         |33| user4    | 2020-04-12 11:20:22.0  | 10         |34+----------+------------------------+------------+--+

#### rank ：使用频率 ★★★★

> 和 row_number 功能一样，都是分组内统计排名，但是当出现同样排名的时候，中间会出现空位。这里给一个例子就可以很容易理解了

     1select  2  user_id, 3  visit_time, 4  visit_date, 5  rank() over(partition by user_id order by visit_date desc) as rank --每个用户按照访问时间倒排，通常用于统计用户最近一天的访问记录 6from wedw_tmp.tmp_url_info 7order by user_id,rank 8+----------+------------------------+-------------+-------+--+ 9| user_id  |       visit_time       | visit_date  | rank  |10+----------+------------------------+-------------+-------+--+11| user1    | 2020-09-12 02:20:02.0  | 2020-09-12  | 1     |12| user1    | 2020-09-12 02:20:02.0  | 2020-09-12  | 1     | --同一天访问了两次，9月11号访问排名第三13| user1    | 2020-09-11 11:20:12.0  | 2020-09-11  | 3     |14| user1    | 2020-09-10 08:19:22.0  | 2020-09-10  | 4     |15| user1    | 2020-08-12 19:20:22.0  | 2020-08-12  | 5     |16| user2    | 2020-05-17 06:20:22.0  | 2020-05-17  | 1     |17| user2    | 2020-05-16 19:03:32.0  | 2020-05-16  | 2     |18| user2    | 2020-05-15 12:34:23.0  | 2020-05-15  | 3     |19| user2    | 2020-05-15 13:34:23.0  | 2020-05-15  | 3     |20| user2    | 2020-05-12 18:24:31.0  | 2020-05-12  | 5     |21| user2    | 2020-04-12 19:12:36.0  | 2020-04-12  | 6     |22| user2    | 2020-04-04 12:23:22.0  | 2020-04-04  | 7     |23| user3    | 2020-08-12 12:23:22.0  | 2020-08-12  | 1     |24| user3    | 2020-08-02 08:10:22.0  | 2020-08-02  | 2     |25| user3    | 2020-08-02 10:10:22.0  | 2020-08-02  | 2     |26| user3    | 2020-04-12 08:02:22.0  | 2020-04-12  | 4     |27| user4    | 2020-04-12 11:20:22.0  | 2020-04-12  | 1     |28| user4    | 2020-03-12 10:20:22.0  | 2020-03-12  | 2     |29| user4    | 2020-02-12 20:26:43.0  | 2020-02-12  | 3     |30+----------+------------------------+-------------+-------+--+

#### dense_rank：使用频率 ★★★★

> 和 row_number 以及 rank 功能一样，都是分组排名，但是该排名如果出现同次序的话，中间不会留下空位

     1--还是以rank的sql为例子 2select  3  user_id, 4  visit_time, 5  visit_date, 6  dense_rank() over(partition by user_id order by visit_date desc) as rank  7from wedw_tmp.tmp_url_info 8order by user_id,rank 9+----------+------------------------+-------------+-------+--+10| user_id  |       visit_time       | visit_date  | rank  |11+----------+------------------------+-------------+-------+--+12| user1    | 2020-09-12 02:20:02.0  | 2020-09-12  | 1     |13| user1    | 2020-09-12 02:20:02.0  | 2020-09-12  | 1     |14| user1    | 2020-09-11 11:20:12.0  | 2020-09-11  | 2     |--中间不会留下空缺15| user1    | 2020-09-10 08:19:22.0  | 2020-09-10  | 3     | 16| user1    | 2020-08-12 19:20:22.0  | 2020-08-12  | 4     |17| user2    | 2020-05-17 06:20:22.0  | 2020-05-17  | 1     |18| user2    | 2020-05-16 19:03:32.0  | 2020-05-16  | 2     |19| user2    | 2020-05-15 12:34:23.0  | 2020-05-15  | 3     |20| user2    | 2020-05-15 13:34:23.0  | 2020-05-15  | 3     |21| user2    | 2020-05-12 18:24:31.0  | 2020-05-12  | 4     |22| user2    | 2020-04-12 19:12:36.0  | 2020-04-12  | 5     |23| user2    | 2020-04-04 12:23:22.0  | 2020-04-04  | 6     |24| user3    | 2020-08-12 12:23:22.0  | 2020-08-12  | 1     |25| user3    | 2020-08-02 08:10:22.0  | 2020-08-02  | 2     |26| user3    | 2020-08-02 10:10:22.0  | 2020-08-02  | 2     |27| user3    | 2020-04-12 08:02:22.0  | 2020-04-12  | 3     |28| user4    | 2020-04-12 11:20:22.0  | 2020-04-12  | 1     |29| user4    | 2020-03-12 10:20:22.0  | 2020-03-12  | 2     |30| user4    | 2020-02-12 20:26:43.0  | 2020-02-12  | 3     |31+----------+------------------------+-------------+-------+--+

#### rank/dense_rank/row_number 对比

> 相同点：都是分组排序
>
> 不同点：
>
> 1.  Row_number: 即便出现相同的排序，排名也不会一致，只会进行累加；**即排序次序连续，但不会出现同一排名**
> 2.  rank: 当出现相同的排序时，中间会出现一个空缺，**即分组内会出现同一个排名，但是排名次序是不连续的**
> 3.  Dense_rank: 当出现相同排序时，中间不会出现空缺，**即分组内可能会出现同样的次序，且排序名次是连续的**

#### first_value：使用频率 ★★★

> 按照分组排序取截止到当前行的第一个值; 通常用于取最新记录或者最早的记录 (根据排序字段进行变通即可)

    1--仍然使用row_number的例子;方便读者理解2select3user_id,4visit_time,5visit_cnt,6first_value(visit_time) over(partition by user_id order by visit_date desc) as first_value_time,7row_number() over(partition by user_id order by visit_date desc) as rank8from  wedw_tmp.tmp_url_info9order by user_id,rank

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK2AxiaykpfFeEViafyarcLzB3kAnOOnLPu5EozTxibhd5biabs1iaYgGXzhA/640?wx_fmt=png)

#### last_value：使用频率 ★

> 按照分组排序取当前行的最后一个值；这个函数好像没啥卵用

    1--仍然使用row_number的例子;方便读者理解2select3user_id,4visit_time,5visit_cnt,6last_value(visit_time) over(partition by user_id order by visit_date desc) as first_value_time,7row_number() over(partition by user_id order by visit_date desc) as rank8from  wedw_tmp.tmp_url_info9order by user_id,rank

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKqlYwjAvibvQicoYzol35L8aKvBGia8MKq5TERYMao5HOtqQSiaIVPYI54g/640?wx_fmt=png)

#### lead：使用频率 ★★

> LEAD(col,n,DEFAULT) 用于取窗口内往下第 n 行值；通常用于行值填充；或者和指定行进行差值比较
>
> 第一个参数为列名
>
> 第二个参数为往下第 n 行 (可选),
>
> 第三个参数为默认值 (当往下第 n 行为 NULL 时候, 取默认值, 如不指定, 则为 NULL)

     1select 2user_id, 3visit_time, 4visit_cnt, 5row_number() over(partition by user_id order by visit_date desc) as rank, 6lead(visit_time,1,'1700-01-01') over(partition by user_id order by visit_date desc) as lead_time 7from  wedw_tmp.tmp_url_info 8order by user_id 9+----------+------------------------+------------+-------+------------------------+--+10| user_id  |       visit_time       | visit_cnt  | rank  |       lead_time        |11+----------+------------------------+------------+-------+------------------------+--+12| user1    | 2020-09-12 02:20:02.0  | 10         | 1     | 2020-09-12 02:20:02.0  | --取下一行的值作为当前值13| user1    | 2020-09-12 02:20:02.0  | 10         | 2     | 2020-09-11 11:20:12.0  |14| user1    | 2020-09-11 11:20:12.0  | 2          | 3     | 2020-09-10 08:19:22.0  |15| user1    | 2020-09-10 08:19:22.0  | 4          | 4     | 2020-08-12 19:20:22.0  |16| user1    | 2020-08-12 19:20:22.0  | 5          | 5     | 1700-01-01 00:00:00.0  | --这里是最后一条记录，则取默认值17| user2    | 2020-05-17 06:20:22.0  | 10         | 1     | 2020-05-16 19:03:32.0  |18| user2    | 2020-05-16 19:03:32.0  | 19         | 2     | 2020-05-15 12:34:23.0  |19| user2    | 2020-05-15 12:34:23.0  | 30         | 3     | 2020-05-15 13:34:23.0  |20| user2    | 2020-05-15 13:34:23.0  | 30         | 4     | 2020-05-12 18:24:31.0  |21| user2    | 2020-05-12 18:24:31.0  | 40         | 5     | 2020-04-12 19:12:36.0  |22| user2    | 2020-04-12 19:12:36.0  | 10         | 6     | 2020-04-04 12:23:22.0  |23| user2    | 2020-04-04 12:23:22.0  | 29         | 7     | 1700-01-01 00:00:00.0  |24| user3    | 2020-08-12 12:23:22.0  | 50         | 1     | 2020-08-02 08:10:22.0  |25| user3    | 2020-08-02 08:10:22.0  | 5          | 2     | 2020-08-02 10:10:22.0  |26| user3    | 2020-08-02 10:10:22.0  | 6          | 3     | 2020-04-12 08:02:22.0  |27| user3    | 2020-04-12 08:02:22.0  | 43         | 4     | 1700-01-01 00:00:00.0  |28| user4    | 2020-04-12 11:20:22.0  | 10         | 1     | 2020-03-12 10:20:22.0  |29| user4    | 2020-03-12 10:20:22.0  | 30         | 2     | 2020-02-12 20:26:43.0  |30| user4    | 2020-02-12 20:26:43.0  | 20         | 3     | 1700-01-01 00:00:00.0  |31+----------+------------------------+------------+-------+------------------------+--+

#### lag：使用频率 ★★

> 和 lead 功能一样，但是是取上 n 行的值作为当前行值

    1select2user_id,3visit_time,4visit_cnt,5row_number() over(partition by user_id order by visit_date desc) as rank,6lag(visit_time,1,'1700-01-01') over(partition by user_id order by visit_date desc) as lead_time7from  wedw_tmp.tmp_url_info8order by user_id

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKzyTvQCyicibTE8aviaOfO8zD9uicffaAm5oBGcvtYcG1F05dm3gfjff8jQ/640?wx_fmt=png)

### 集合相关

#### collect_set：使用频率 ★★★★★

> 将分组内的数据放入到一个集合中，具有去重的功能；

    1--统计每个用户具体哪些天访问过2select3  user_id,4  collect_set(visit_date) over(partition by user_id) as visit_date_set 5from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK8CTgHicz8MV07uIs4CEpo55kicHKeWedd39h0qUdT4vN49uq7zLdz45w/640?wx_fmt=png)

#### collect_list：使用频率 ★★★★★

> 和 collect_set 一样，但是没有去重功能

    1select2  user_id,3  collect_set(visit_date) over(partition by user_id) as visit_date_set 4from wedw_tmp.tmp_url_info56--如下图可见，user2在2020-05-15号多次访问，这里也算进去了

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKqBEG2XuZcwNQ0L3y5oZVuxia6DuTSLkzGQzFQC87gqMqwyPL6pLc8gg/640?wx_fmt=png)

#### sort_array：使用频率 ★★★

> 数组内排序；通常结合 collect_set 或者 collect_list 使用；
>
> 如 collect_list 为例子，可以发现日期并不是按照顺序组合的，这里有需求需要按照时间升序的方式来组合

    1--按照时间升序来组合2select3  user_id,4  sort_array(collect_list(visit_date) over(partition by user_id)) as visit_date_set 5from wedw_tmp.tmp_url_info6--结果如下图所示；

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK51Bv25YuCEMMo4WFYK1HuiaEWR55T9f6icFGgiaPslIHEia1AXysCol9Kw/640?wx_fmt=png)

> 如果突然业务方改需求了，想要按照时间降序来组合，那基于上面的 sql 该如何变通呢？哈哈哈哈，其实没那么复杂，这里根据没必要按照 sort_array 来实现，在 collect_list 中的分组函数内直接按照 visit_date 降序即可，这里只是为了演示 sort_array 如何使用

    1--按照时间降序排序2select3  user_id,4  collect_list(visit_date) over(partition by user_id order by visit_date desc) as visit_date_set 5from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK6ib9Q57Me6gLaIdPtH3btNG43LE5brfhXicJWKicriajwdwsWzNY2FkPdQ/640?wx_fmt=png)

> 这里还有一个小技巧，对于数值类型统计多列或者数组内的最大值，可以使用 sort_array 来实现

     1--具体思路就是先把数值变成负数，然后升序排序即可 2select -sort_array(array(-a,-b,-c))[0] as max_value 3from ( 4    select 1 as a, 3 as b, 2 as c 5) as data 6 7+------------+--+ 8| max_value  | 9+------------+--+10| 3          |11+------------+--+

### URL 相关

#### parse_url：使用频率 ★★★★

> 用于解析 url 相关的参数, 直接上 sql

     1select  2visit_url, 3parse_url(visit_url, 'HOST') as url_host, --解析host 4parse_url(visit_url, 'PATH') as url_path, --解析path 5parse_url(visit_url, 'QUERY') as url_query,--解析请求参数 6parse_url(visit_url, 'REF') as url_ref, --解析ref 7parse_url(visit_url, 'PROTOCOL') as url_protocol, --解析协议 8parse_url(visit_url, 'AUTHORITY') as url_authority,--解析author 9parse_url(visit_url, 'FILE') as url_file, --解析filepath10parse_url(visit_url, 'USERINFO') as url_user_info --解析userinfo11from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKd3Iq15YFbdWicWOfV4CluxInib5CvrUsQpRzQbmulS0J9KffWGBmNw2w/640?wx_fmt=png)

#### reflect：使用频率 ★★

> 该函数是利用 java 的反射来实现一些功能，目前笔者只用到了关于 url 编解码

    1--url编码2select 3visit_url,4reflect("java.net.URLEncoder", "encode", visit_url, "UTF-8") as visit_url_encode5from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKdlrOLgPfEkWcOXNqmAOhxFbsIaLBYRTLVJx1OA4UIThpMwc0B2yC2w/640?wx_fmt=png)

     1--url解码 2select  3  visit_url, 4 reflect("java.net.URLDecoder", "decode", visit_url_encode, "UTF-8") as visit_url_decode 5from  6( 7  select  8  visit_url, 9  reflect("java.net.URLEncoder", "encode", visit_url, "UTF-8") as visit_url_encode10  from wedw_tmp.tmp_url_info11)t

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKCiaZnicMprOkiacFLc1LWFJmmyIm2cy5ZoTXvX7lNLzibiblpVwFBcNXeDw/640?wx_fmt=png)

### JSON 相关

#### get_json_object：使用频率 ★★★★★

> 通常用于获取 json 字符串中的 key, 如果不存在则返回 null

    1select 2  get_json_object(json_data,'$.user_id') as user_id,3  get_json_object(json_data,'$.age') as age --不存在age，则返回null4from 5(6  select 7     concat('{"user_id":"',user_id,'"}') as json_data8  from wedw_tmp.tmp_url_info9)t

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKJTiaedrDLHVYNCOMoggwt3gfHNbtfuF17GJNhrriacbB6JvHJoQIEerA/640?wx_fmt=png)

### 列转行相关

#### explode：使用频率 ★★★★★

> 列转行，通常是将一个数组内的元素打开，拆成多行

     1--简单例子 2select  explode(array(1,2,3,4,5)) 3+------+--+ 4| col  | 5+------+--+ 6| 1    | 7| 2    | 8| 3    | 9| 4    |10| 5    |11+------+-12--结合lateral view 使用13select 14  get_json_object(user,'$.user_id')15from 16(17  select 18   distinct collect_set(concat('{"user_id":"',user_id,'"}')) over(partition by year(visit_date)) as user_list19  from wedw_tmp.tmp_url_info20)t21lateral view explode(user_list) user_list as user

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKQ0PRIcQR9Q4Uz5rUB4lezFEE1Hcgiaen4cep1ObiatlUfx1mC0FQOEow/640?wx_fmt=png)

### Cube 相关

#### GROUPING SETS：使用频率 ★

> 类似于 kylin 中的 cube，将多种维度进行组合统计；在一个 GROUP BY 查询中, 根据不同维度组合进行聚合，等价于将不同维度的 GROUP BY 结果集进行 UNION ALL

     1--按照用户+访问日期统计统计次数 2select  3  user_id,  4  visit_date, 5  sum(visit_cnt) as visit_cnt 6from wedw_tmp.tmp_url_info 7group by user_id,visit_date 8grouping sets(user_id,visit_date) 910--下图的结果类似于以下sql11select 12  user_id, 13  NULL as visit_date,14  sum(visit_cnt) as visit_cnt15from wedw_tmp.tmp_url_info16union all 17select 18  NULL as user_id, 19  visit_date,20  sum(visit_cnt) as visit_cnt21from wedw_tmp.tmp_url_info22union all 23select 24  user_id, 25  visit_date,26  sum(visit_cnt) as visit_cnt27from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK4L9bzonKVFxEhnT5rlSmVxKPBpbOiafJKfuq2HOoJ5akdcDsv7qKYzQ/640?wx_fmt=png)

### 字符相关

#### concat：使用频率 ★★★★★

> 字符拼接，concat(string|binary A, string|binary B…); 该函数比较简单

    1select concat('a','b','c') 2--最后结果就是abc

#### concat_ws：使用频率 ★★★★★

> 按照指定分隔符将字符或者数组进行拼接；concat_ws(string SEP, array)/concat_ws(string SEP, string A, string B…)

    1--还是concat使用的例子，这里可以写成2select concat_ws('','a','b','c')34--将数组列表元素按照指定分隔符拼接，类似于python中的join方法5select concat_ws('',array('a','b','c'))

#### instr：使用频率 ★★★★

> 查找字符串 str 中子字符串 substr 出现的位置，如果查找失败将返回 0，如果任一参数为 Null 将返回 null，注意位置为从 1 开始的；通常笔者用这个函数作为模糊查询来查询

    1--查询vist_time包含10的记录2select 3 user_id,4 visit_time,5 visit_date,6 visit_cnt7from wedw_tmp.tmp_url_info8where instr(visit_time,'10')>0

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKJWSWjBJW7w37spd3Za2oxb4uNLNzypNbodrARiaJjnK1fThgubias8HQ/640?wx_fmt=png)

#### length：使用频率 ★★★★★

> 统计字符串的长度

    1select length('abc')

#### size: 使用频率 ★★★★★

> 是用来统计数组或者 map 的元素，通常笔者用该函数用来统计去重数（一般都是通过 distinct，然后 count 统计，但是这种方式效率较慢）

     1--使用size 2select  3   distinct size(collect_set(user_id) over(partition by year(visit_date))) 4from wedw_tmp.tmp_url_info 5+-----------+--+ 6| user_cnt  | 7+-----------+--+ 8| 4         | 9+-----------+--+101 row selected (0.268 seconds)1112--使用通过distinct，然后count统计的方式13select 14  count(1)15from 16(17  select 18    distinct user_id19  from wedw_tmp.tmp_url_info 20)t21+-----------+--+22| count(1)  |23+-----------+--+24| 4         |25+-----------+--+261 row selected (0.661 seconds)2728--笔者这里只用到了19条记录数，就可以明显观察到耗时差异，这里涉及到shuffle问题，后续将会有单独的文章来讲解hive的数据倾斜问题

#### trim：使用频率 ★★★★★

> 将字符串前后的空格去掉，和 java 中的 trim 方法一样，这里还有 ltrim 和 rtrim, 不再讲述了

    1--最后会得到sfssf sdf sdfds2select trim(' sfssf sdf sdfds ') 

#### regexp_replace：使用频率 ★★★★★

> regexp_replace(string INITIAL_STRING, string PATTERN, string REPLACEMENT)
>
> 按照 Java 正则表达式 PATTERN 将字符串中符合条件的部分成 REPLACEMENT 所指定的字符串，如里 REPLACEMENT 空的话，抽符合正则的部分将被去掉

    1--将url中？参数后面的内容全部剔除2  select 3    distinct regexp_replace(visit_url,'\\?(.*)','') as visit_url4  from wedw_tmp.tmp_url_info

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKVmNia92Ua7kf4bGtQwMsbNrXicSGxqoKplyBY9PnMFia2PglDIibic0LcicQ/640?wx_fmt=png)

#### regexp_extract：使用频率 ★★★★

> regexp_extract(string subject, string pattern, int index)
>
> 抽取字符串 subject 中符合正则表达式 pattern 的第 index 个部分的子字符串，注意些预定义字符的使用
>
> 类型于 python 爬虫中的 xpath，用于提取指定的内容

    1--提取csdn文章编号2select 3    distinct regexp_extract(visit_url,'/details/([0-9]+)',1) as visit_url4  from wedw_tmp.tmp_url_info 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK7wZiabQgcLhxn5VFrC8fE2EXRYzPcN3rLxicZwT1ho9XyFmDwdqhkYOA/640?wx_fmt=png)

#### substring_index：使用频率 ★★

> substring_index(string A, string delim, int count)
>
> 截取第 count 分隔符之前的字符串，如 count 为正则从左边开始截取，如果为负则从右边开始截取

     1--比如将2020年的用户组合获取前2个用户,下面的sql将上面讲解的函数都结合在一起使用了 2select  3  user_set, 4  substring_index(user_set,',',2) as user_id 5from   6( 7  select  8    distinct concat_ws(',',collect_set(user_id) over(partition by year(visit_date))) as user_set 9  from wedw_tmp.tmp_url_info 10)t

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKsBOMBeftC1rldoJOaEtdRuLoO2Na5WytaKGHEJw7OYhp7Wd9iauPBicw/640?wx_fmt=png)

### 条件判断

#### if：使用频率 ★★★★★

> if(boolean testCondition, T valueTrue, T valueFalseOrNull): 判断函数，很简单
>
> 如果 testCondition 为 true 就返回 valueTrue, 否则返回 valueFalseOrNull

    1--判断是否为user1用户2select 3  distinct user_id,4  if(user_id='user1',true,false) as flag5from wedw_tmp.tmp_url_info 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKgNbbm6ARvTqNgPQHe5TMnE4EDTMcsrW3gJiaZzQylYOL202aUU4lowg/640?wx_fmt=png)

#### case when ：使用频率 ★★★★★

> CASE a WHEN b THEN c \[WHEN d THEN e]  \[ELSE f] END
>
> 如果 a=b 就返回 c,a=d 就返回 e，否则返回 f  如 CASE 4 WHEN 5  THEN 5 WHEN 4 THEN 4 ELSE 3 END 将返回 4
>
> 相比 if，个人更倾向于使用 case when

    1--仍然以if上面的列子2select 3  distinct user_id,4  case when user_id='user1' then 'true'5     when user_id='user2' then 'test'6  else 'false' end  as flag7from wedw_tmp.tmp_url_info 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlK6xyzmMjv2GMcefK56tIsLsfTamxpeG9yRcvTYRWTiae7Nr3p3aLt9SA/640?wx_fmt=png)

#### coalesce：使用频率 ★★★★★

> COALESCE(T v1, T v2, …)
>
> 返回第一非 null 的值，如果全部都为 NULL 就返回 NULL

     1--该函数结合lead或者lag更容易贴近实际业务需求,这里使用lead，并取后3行的值作为当前行值 2select  3  user_id, 4  visit_time, 5  rank, 6  lead_time, 7  coalesce(visit_time,lead_time) as has_time 8from  9(10  select11  user_id,12  visit_time,13  visit_cnt,14  row_number() over(partition by user_id order by visit_date desc) as rank,15  lead(visit_time,3) over(partition by user_id order by visit_date desc) as lead_time16  from  wedw_tmp.tmp_url_info17  order by user_id18)t

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKSua3FVXLmibVfnb1SxSYcvtG76In9I6Mdd4TJ0RnPWic9xIxFaU6uH4g/640?wx_fmt=png)

### 数值相关

#### round：使用频率 ★★

> round(DOUBLE a): 返回对 a 四舍五入的 BIGINT 值,
>
> round(DOUBLE a, INT d): 返回 DOUBLE 型 d 的保留 n 位小数的 DOUBLW 型的近似值
>
> 该函数没什么可以讲解的

    1select round(4/3),round(4/3,2);2+------+-------+--+3| _c0  |  _c1  |4+------+-------+--+5| 1.0  | 1.33  |6+------+-------+--+

#### ceil：使用频率 ★★★

> ceil(DOUBLE a), ceiling(DOUBLE a)
>
> 求其不小于小给定实数的最小整数;**向上取整**

    1select ceil(4/3),ceiling(4/3)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKKPXGB1MzIxNxtK6fy64wmS6QBGjWqlXaTEWPGicgfoFVNraWJ7bckSA/640?wx_fmt=png)

#### floor：使用频率 ★★★

> floor(DOUBLE a):**向下取整**''

    1select floor(4/3);

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKcGm7YXW7te4fFslxt6Dkj1bDbtfUsl3LYy7SZjDVM7ad6c0y7rG0EA/640?wx_fmt=png)

#### hex：使用频率 ★

> hex(BIGINT a)/ hex(STRING a)/ hex(BINARY a)
>
> 计算十六进制 a 的 STRING 类型，如果 a 为 STRING 类型就转换成字符相对应的十六进制
>
> 该函数很少使用，主要是因为曾经遇到过关于 emoj 表情符脏数据，故使用该函数进行处理

### 时间相关 (比较简单)

#### from_unxitime：使用频率 ★★★★★

> from_unixtime(bigint unixtime\[, string format])
>
> 将时间的秒值转换成 format 格式（format 可为 “yyyy-MM-dd hh:mm:ss”,“yyyy-MM-dd hh”,“yyyy-MM-dd hh:mm” 等等）

    1select from_unixtime(1599898989,'yyyy-MM-dd') as current_time2+---------------+--+3| current_time  |4+---------------+--+5| 2020-09-12    |6+---------------+--+

#### unix_timestamp：使用频率 ★★★★★

> unix_timestamp(): 获取当前时间戳
>
> unix_timestamp(string date)：获取指定时间对应的时间戳
>
> 通过该函数结合 from_unixtime 使用，或者可计算两个时间差等

    1select 2 unix_timestamp() as current_timestamp,--获取当前时间戳3 unix_timestamp('2020-09-01 12:03:22') as speical_timestamp,--指定时间对于的时间戳4 from_unixtime(unix_timestamp(),'yyyy-MM-dd')  as current_date --获取当前日期

![](https://mmbiz.qpic.cn/sz_mmbiz_png/fBHtXf4CBicJicl5pVl6dJDhmr8YwerrlKaeyFL0CywArY61cicMNiaHHib4naTqHQZz5qSfny89sjI1HN8sdMvUJ6A/640?wx_fmt=png)

#### to_date：使用频率 ★★★★★

> to_date(string timestamp)
>
> 返回时间字符串的日期部分

    1--最后得到2020-09-102select to_date('2020-09-10 10:31:31') 

#### year：使用频率 ★★★★★

> year(string date)
>
> 返回时间字符串的年份部分

    1--最后得到20202select year('2020-09-02')

#### month：使用频率 ★★★★★

> month(string date)
>
> 返回时间字符串的月份部分

    1--最后得到092select month('2020-09-10')

#### day：使用频率 ★★★★★

> day(string date)
>
> 返回时间字符串的天

    1--最后得到102select day('2002-09-10')

#### date_add：使用频率 ★★★★★

> date_add(string startdate, int days)
>
> 从开始时间 startdate 加上 days

    1--获取当前时间下未来一周的时间2select date_add(now(),7) 3--获取上周的时间4select date_add(now(),-7)

#### date_sub：使用频率 ★★★★★

> date_sub(string startdate, int days)
>
> 从开始时间 startdate 减去 days

    1--获取当前时间下未来一周的时间2select date_sub(now(),-7) 3--获取上周的时间4select date_sub(now(),7)

**往期回顾**

◀[数据同步神器 - Datax 源码重构](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247484079&idx=1&sn=6793e2912a69cb0a402d8ea7421a74f6&chksm=fd4e1d4bca39945da6f5b391b3cfd1852af3509660a87a982c9502faab4a0eee4493616c52de&scene=21#wechat_redirect)

◀[2020 年大厂面试题 - 数据仓库篇](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247484057&idx=1&sn=9df87f8a364825246625242df5b41e00&chksm=fd4e1d7dca39946b66e25a83a13a3ae662101e9fcb5c556a25fcace3228a811088021bf09d2f&scene=21#wechat_redirect)

◀[0003-01-Hive 常见问题汇总](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247484057&idx=2&sn=74695c1d9d5e0e9a0be67a370cd29516&chksm=fd4e1d7dca39946bf739ec17d80694e4b2430e8d0159b427f7b3fa12b44ea90461a09d27b33b&scene=21#wechat_redirect)

◀[zookeeper 源码解读之 - DataTree 模型构建 + Leader 选举](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247484002&idx=1&sn=aa366400c6a344bef86e25e40892c0cb&chksm=fd4e1d86ca39949040611fc4f2d6e3b0ea54d3f7019a2e727a70c9257fca7509edc1628fbacb&scene=21#wechat_redirect)

◀[zookeeper 源码解读之 - 服务端启动流程](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247484001&idx=1&sn=343bdeae32355d665d2097ebfbe27074&chksm=fd4e1d85ca39949376bb8b12d360095967b1258dd6e3a6477c5729148bb35e52e91ed710fe47&scene=21#wechat_redirect)

◀[Spark 数据倾斜之骚操作解决方案](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247483826&idx=1&sn=8164022d80a8317cc374204a0840392d&chksm=fd4e1e56ca399740cba4e486c331db5926cdb269f93bff2d0460ed1b91192034db94c4cca3bb&scene=21#wechat_redirect)

◀[0004-01-03 Livy REST 提交 Spark 作业](http://mp.weixin.qq.com/s?__biz=MzU4MDkzMDE4Nw==&mid=2247483785&idx=1&sn=4d40f3178d2ece4b985cbefac1401167&chksm=fd4e1e6dca39977bc82d399ebbeff133361fe88a4a917b5758e2c531b4e26fcee051534e02ce&scene=21#wechat_redirect) 
 [https://mp.weixin.qq.com/s/PPpcE4NsOQQsdH9rfpxnGQ](https://mp.weixin.qq.com/s/PPpcE4NsOQQsdH9rfpxnGQ)
