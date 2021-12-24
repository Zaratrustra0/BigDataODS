# 数据处理奇技巧（二）——SQL数据处理小技巧
![](https://mmbiz.qpic.cn/mmbiz_jpg/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyr5y93UqTKCEciapddbEdMYyRPynQtxrVEWibhdXhW8PbukwDF4CfrDAw/640?wx_fmt=jpeg)

**livandata**

数据 EDTA 创始人，没有之一

现担任数据 EDTA 个人公众号董事长兼 CEO 兼财务兼创作人

口号是：让大数据赋能每一个人。

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

这篇文章是基于 hive 库编写的 SQL，处理大数据的各个小哥哥们或许有些用处，函数上与 mysql、Oracle 有一些差异，但是语法大同小异。

欢迎关注。

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**1、格式转换**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）格式转换方法：

```cs
pmod(int a, int b)：返回a除以b的余数的绝对值；
cast(aaa as int):将string转化成int；
cast(aaa as decimal(10, 2)):将string转化成float,保留两位小数；
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**2、去除空格**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）去除空格的方法：

```javascript
trim(String A)：去除A两侧的空格；
ltrim(String A)：去除左边空格；
rtrim(String A)：去除右边空格
select trim('abc') from lxw_dual;
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**3、字符串连接**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）concat_ws (separator,str1,str2,...) ：根据固定的分隔符连接后侧字符串；

concat_ws 第一个参数是其它参数的分隔符, 分隔符的位置放在要连接的两个字符串之间，分隔符可以是一个字符串，也可以是其他参数。

```sql
select concat_ws(',','11','22','33');   　
11,22,33
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

**![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)
4、collect_list/collect_set 函数**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）collect_list/collect_set 列转行函数：

在本地文件系统创建测试文件：

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyIX8ic6LwtkWDcFRLuVdibdCnrhVjIxvcscfly6EvNtyDkekthJ5zo6icQ/640?wx_fmt=png)

存储在 hive 表中：  

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyJZe1cyRLQEJZEFcFCiciatHlgRyFpSibU8n8KN4cWfx4ibJJKMffj7xR6w/640?wx_fmt=png)

按用户分组, 取出每个用户每天看过的所有视频的名字：  

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyzDhzEc9CadFcSszTibBVzknd83zp8RYK6nNYXCh7cqo8s4T6aaRSicMw/640?wx_fmt=png)

上面结果中，由于霸王别姬李四看了两遍，所以列表中存在重复，去重处理 collect_set()

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewy4ibVTmkKtUrnPvWdn2gwfuJsRwIqWjaTOzkjRs79WGnhXKugiajESW9Q/640?wx_fmt=png)

突破 group by 限制:  

还可以利用 collect 来突破 group by 的限制, hive 中在 group by 查询的时候要求出现在 select 后面的列都必须出现在 group by 的后面

即 select 列必须作为分组依据的列，但有的时候我们想根据 A 分组然后随便取出每个分组中的一个 B，带入到这个实验中就是按照

用户进行分组，然后随便拿出一个他看过的视频名称即可:

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyP9IJcVPVtMAYDicmv62JejLic2tLtCD7u82ovicGT9xfyfMxVfVeLeMicw/640?wx_fmt=png)

eg：  

collect_list：该函数功能等同于 collect_set，唯一的区别在于 collect_set 会去除重复元素，collect_list 不去除重复元素，示例 sql 如下：

```sql
with t as (
select 1 id,123 value
  union all
select 1 id,234 value
  union all
select 2 id,124 value
  union all
select 2 id,124 value
)
select t.id,collect_set(t.value),collect_list(t.value)
from t
group by t.id
```

collect_set:  

该函数只接受基本的数据类型，主要作用是将某字段的值进行去重汇总，返回值是 array 类型字段：

```sql
with t as (
select 1 id,123 valu   union all
select 1 id,234 value
  union all
select 2 id,124 value
)
select t.id,collect_set(t.value)
from t
group by t.id
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**5、get_json_object(string json_string,string path)**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

该函数的第一个参数是 json 对象变量，第二个参数使用 $ 表示 json 变量标识，然后用. 或\[]读取对象或数组

```cs
select get_json_object(pricecount,'$.buyoutRoomRequest') new_id,pricecount
from table_sample a
where d='2018-08-31' limit 100
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**6、json_tuple(string json_string,string k1,...)**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

该函数的第一个参数是 json 对象变量，之后的参数是不定长参数，是一组键 k1,k2...，返回值是元组，该方法比 get_json_object 高效，因为可以在一次调用中输入多个键值

```sql
select m.*,n.pricecount
from (select 
      from table_sample a 
      where d='2018-08-31' limit 100)n
lateral view json_tuple(pricecount,'paymentType','complete') m as f1,f2
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**7、split(str,regex)：数据切分**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

数据切分：

该函数第一个参数是字符串，第二个参数是设定的分隔符，通过第二个参数把第一个参数做拆分，返回一个数组  

```perl
select split('123,3455,2568',',')
select split('sfas:sdfs:sf',':')
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**8\*\***、explode()：列转行 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

该函数接收一个参数，参数类型需是 array 或者 map 类型，该函数的输出是把输入参数的每个元素拆成独立的一行记录。

```sql
select explode(split('123,3455,2568',','))
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**9\*\***、lateral view 行转列函数 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

Lateral View 一般与用户自定义表生成函数（如 explode()）结合使用。UDTF 为每个输入行生成零个或多个输出行，Lateral view 首先将 UDTF 应用于基表的每一行，然后将结果输出行连接到输入行，已形成具有提供的表别名的虚拟表。

```sql
select j.nf,p.* from (
    select m.*,n.pricecount
    from (select * from ods_htl_htlinfogoverndb.buyout_appraise a where d =
          '${zdt.format("yyyy-MM-dd")}' limit 100)n
    lateral view json_tuple(pricecount,'paymentType','complete') m as f1,f2 )p
lateral view explode(split(regexp_replace(p.f1,'\\[|\\]',''),',')) j as nf
```

结合：hive 中行变列和列变行的函数：

collect_list() 与 collect_set() 是将数据列变行；

explode() 与 lateral view 是将数据行变列；

eg:

```sql
select items_page, num
from data_limit
lateral view explode(items_pages) good as items_page
```

其中：

1）explode 是将 items_pages 行变列；

2）good 是虚拟表，用来存储 explode 之后的数据，并定义列名为 items_page;

3）lateral view 是将 items_page 与上面的字段 num 进行笛卡尔积，呈现出多列表示；

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

**![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)
10\*\***、K_V 转化函数 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

使用两个分隔符将文本拆分成键值对。Delimiter1 将文本分成 k-v 对，Delimiter2 分割每个 k-v 对。对于 delimiter1 的默认值是','，delimiter2 的默认值是'='.

```cs
select str_to_map('abc:11&bcd:22', '&', ':')
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**11\*\***、\***\*array_contains(Array<T>,value)**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

该函数的用来判断 Arrary<T>中是否包含元素 value，返回值是 boolean

```sql
select array_contains(array(1,2,3,4,5),3)
true
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**12\*\***、\***\* 常见小函数**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）percentile(expr,pc)：

该函数用来计算参数 expr 的百分位数，expr：字段类型必须是 INT，否则报错；pc: 百分位数，已数值形式传入

2）percentile_approx(expr,pc,\[nb])

该函数也是用来计算参数 expr 的百分位数，但数据类型要求没有 percentile 严格，该函数数值类似类型都可以；pc: 百分位数，可以以数组形式传入，因此可以一次性查看多个指定百分位数；\[nb]：控制内存消耗的精度，选填

3）数学函数：

（1）round：四舍五入 

```sql
select round(数值,小数点位数);
```

（2）ceil：向上取整

```sql
select ceil(45.6);
```

（3）floor：向下取整 

```sql
select floor(45.6);
```

4）字符函数：

（1）lower：转成小写

```sql
 select lower('Hive'); 
```

（2）upper：转成大写

```sql
select lower('Hive'); 
```

（3）length：长度

```sql
  select length('Hive'); 
```

（4）concat：拼接字符串

```sql
  select concat('hello','Hive'); 
```

（5）substr：求子串

```sql
 select substr('hive',2); 
     select substr('hive',2,1); 
```

（6）trim：去掉前后的空格

```sql
select trim('  hive   '); -hive
```

（7）lpad：左填充

      对 hive 填充到 10 位，补位用#

```sql
 select lpad('hive',10,'#'); 
```

（8）rpad：右填充

```cs
  select rpad('hive',10,'#'); --hive######
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**13\*\***、\***\* 日期函数应用**

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

（1）to_date:

select  to_date('2015-06-01 15:34:23');

（2）year  

select  year('2015-05-22 15:34:23');

 （3）month

select  month('2015-05-22 15:34:23');

（4）day

select  day('2015-05-22 15:34:23');

（5）weekofyear

select  weekofyear('2015-05-22 15:34:23');

（6）datediff(date,date1)：日期差，即两个日期差几天，  

         months_between(date,date1)，两个日期差几月，用法一致。

select  datediff('2015-05-22 15:34:23','2015-05-29 15:34:23');

 （7）date_add：在现在日期上增加天数，add_months 是增加月份，用法一致。

  select  date_add('2015-05-22 15:34:23',2);

（8）date_sub：在现在日期上减少天数

select  date_sub('2015-05-22 15:34:23',2);

9）date_format(string/date,dateformate): 把字符串或者日期转成指定格式的日期。

```sql
select date_format('2018-09-12','yyyy-MM-dd HH:mm:ss') as date_time,date_format('2018-09-12','yyyyMMdd') as date_time1 from dual;
```

（10）unix_timestamp(date,dateformat)：日期格式转化为时间戳，如果括号内没有参数则表示返回当前的时间戳。

```sql
select unix_timestamp() as time_stamp,unix_timestamp('2018-09-26 9:13:26','yyyy-MM-ddHH:mm:ss') as time_stamp1 from dual;
```

（11）from_unixtime(timestamp,dateformat)：将时间戳转化为日期格式，格式必须是 10 位，毫秒级的时间戳需要用 cast 转化成秒级。

```cs
select from_unixtime(unix_timestamp(),'yyyy-MM-dd HH:mm:ss') asdate_time,from_unixtime(1537924406,'yyyy-MM-dd') as date_time1 from dual;
from_unixtime(cast(unix_timestamp()/1000 as int),'yyyy-MM-dd HH:mm:ss') as date_time
```

（12）last_day(date)：获取月末最后一天：

```sql
select last_day('2018-09-30') as date_time,last_day('2018-09-27 21:16:13') as date_time1 from dual;
```

（13）trunc(date,format)  format:MONTH/MON/MM, YEAR/YYYY/YY：返回月初、年初

```sql
select trunc('2018-09-27','YY') as date_time,trunc('2018-09-27 21:16:13','MM') as date_time1 from dual;

```

（14）next_day(date,formate) format: 英文星期几的缩写或者全拼：当前日期下个星期 X 的日期。

```sql
select next_day('2018-09-27','TH') as date_time,next_day('2018-09-27 21:16:13','TU') as date_time1 from dual;

```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**14\*\***、条件函数 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

```sql
select nvl(T v1, T default_value);     
select if(boolean testCondition, T valueTrue, T valueFalseOrNull);  
select coalesce(T v1, T v2, T v3, ...);  
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**15\*\***、库表操作 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

\--  删除库

```sql
drop database if exists db_name;
```

\--  强制删除库

```sql
drop database if exists db_name cascade;
```

\--  删除表

```sql
drop table if exists employee;
```

\-  清空表

```sql
truncate table employee;
```

\-  清空表，第二种方式

```sql
insert overwrite table employee select * from employee where 1=0;
```

\-  删除分区

```sql
alter table employee_table drop partition (stat_year_month>='2018-01');
```

\-  按条件删除数据

```sql
insert overwrite table employee_table select * from employee_table where id>'180203a15f';
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

**16\*\***、高阶操作 \*\*

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyhu7NkL2KMLicjFAXAFI8MEeXdz5yMpwYumhuJjYpaSrzBib1PbJsxajg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

1）row_number() 函数：

```cs
row_number() over(distribute by col1 sort by clo2 desc)
```

2）正则表达式的应用：

regexp_extract(string subject, string pattern, int index) 函数的应用；

将字符串 subject 按照 pattern 正则表达式的规则拆分，返回 index 指定的字符。

```cs
select clientcode
,regexp_extract(filterlist,'(filtertype"\\:")(\\d+)(",)',2)               as filtertype
,regexp_extract(filterlist,'(filtername"\\:")((\\W*\\w*)|(\\W*))(",)',2)  as filtername
,regexp_extract(filterlist,'(filtertitle"\\:")((\\W*\\w*)|(\\W*))(",)',2) as filtertitle
,regexp_extract(filterlist,'(filterid"\\:")(\\d+\\|\\d+)(",)',2)          as filterid
from tmp_action_click
```

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyibUMyCjay4ChSsszowx8le3Wl8PlKVDzZZRiam2arIOcX38Fs96wSXcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3EwyEVQVDahjeZGZOMB4sQzO6byQsiad1CWO6JCf0d5LdwygahCZTVzfGeg/640?wx_fmt=png)

THE END

![](https://mmbiz.qpic.cn/mmbiz_png/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewy0Vl69NQBBap7BEOSlf5jlB00HBfd6894vbMJJBdSVEo3tNONREQIRw/640?wx_fmt=png)

当一份缘分摆在你面前时，你努力争取过，但求转身离开时不留遗憾与后悔，人生尽力而为，剩下的交给天意。

如果喜欢我的文字，请关注公众号：数据 EDTA

![](https://mmbiz.qpic.cn/mmbiz_jpg/MzF2oGuple2zfXzxkcayD5kj6lCs3Ewyr5y93UqTKCEciapddbEdMYyRPynQtxrVEWibhdXhW8PbukwDF4CfrDAw/640?wx_fmt=jpeg)

欢迎大家沟通数据的那些有趣玩法~ 
 [https://mp.weixin.qq.com/s/\_hcYIbQMjzPehsA8-9qJMw](https://mp.weixin.qq.com/s/_hcYIbQMjzPehsA8-9qJMw)
