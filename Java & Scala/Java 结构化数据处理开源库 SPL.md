# Java 结构化数据处理开源库 SPL
现代 Java 应用架构越来越强调数据存储和处理分离，以获得更好的可维护性、可扩展性以及可移植性，比如火热的微服务就是一种典型。这种架构通常要求业务逻辑要在 Java 程序中实现，而不是像传统应用架构中放在数据库中。

应用中的业务逻辑大都会涉及结构化数据处理。数据库（SQL）中对这类任务有较丰富的支持，可以相对简易地实现业务逻辑。但 Java 却一直缺乏这类基础支持，导致用 Java 实现业务逻辑非常繁琐低效。结果，虽然架构上有各种优势，但开发效率却反而大幅下降了。

如果我们在 Java 中也提供有一套完整的结构化数据处理和计算类库，那这个问题就能得到解决：即享受到架构的优势，又不致于降低开发效率。

需要什么样的能力？

Java 下理想的结构化数据处理类库应当具备哪些特征呢？我们可以从 SQL 来总结：

**1.\*\***集合运算能力 \*\*

结构化数据经常是批量（以集合形式）出现的，为了方便地计算这类数据，有必要提供足够的集合运算能力。

如果没有集合运算类库，只有数组（相当于集合）这种基础数据类型，我们要对集合成员做个简单地求和也需要写四五行循环语句才能完成，过滤、分组聚合等运算则要写出数百行代码了。

SQL 提供有较丰富的集合运算，如 SUM/COUNT 等聚合运算，WHERE 用于过滤、GROUP 用于分组，也支持针对集合的交、并、差等基本运算。这样写出来的代码就会短小很多。

**2. Lambda\*\***语法 \*\*

有了集合运算能力是否就够了呢？假如我们为 Java 开发一批的集合运算类库，是否就可以达到 SQL 的效果呢？

没有这么简单！

以过滤运算为例。过滤通常需要一个条件，把满足条件的集合成员保留。在 SQL 中这个条件是以一个表达式形式出现的，比如写 WHERE x>0，就表示保留那些使得 x>0 计算结果为真的成员。这个表达式 x>0 并不是在执行这个语句之前先计算好的，而是在遍历时针对每个集合成员计算的。本质上，这个表达式本质上是一个函数，是一个以当前集合成员为参数的函数。对于 WHERE 运算而言，相当于把一个用表达式定义的函数用作了 WHERE 的参数。

这种写法有一个术语叫做 Lambda 语法，或者叫函数式语言。

如果没有 Lambda 语法，我们就要经常临时定义函数，代码会非常繁琐，还容易发生名字冲突。

SQL 中大量使用了 Lambda 语法，不在于必须过滤、分组运算中，在计算列等不必须的场景也可以使用，大大简化了代码。

**3.\*\***在 Lambda 语法中直接引用字段 \*\*

结构化数据并非简单的单值，而是带有字段的记录。

我们发现，SQL 的表达式参数中引用记录字段时，大多数情况可以直接使用字段名称而不必指明字段所属的记录，只有在多个同名字段时才需要冠以表名（或别名）以区分。

新版本的 Java 虽然也开始支持 Lambda 语法了，但只能把当前记录作为参数传入这个用 Lambda 语法定义的函数，然后再写计算式时就总要带上这个记录。比如用单价和数量计算金额时，如果用于表示当前成员的参数名为 x，则需要写成 “x. 单价 \*x. 数量” 这种啰嗦的形式。而在 SQL 中可以更为直观地写成 " 单价 \* 数量”。

**4.\*\***动态数据结构 \*\*

SQL 还能很好地支持动态数据结构。

结构化数据计算中，返回值经常也是有结构的数据，而结果数据结构和运算相关，没办法在代码编写之前就先准备好。所以需要支持动态的数据结构能力。

SQL 中任何一个 SELECT 语句都会产生一个新的数据结构，在代码中可以随意添加删除字段，而不必事先定义结构（类）。Java 这类语言则不行，在代码编译阶段就要把用到的结构（类）都定义好，原则上不能在执行过程中动态产生新的结构。

**5.\*\***解释型语言 \*\*

从前面几条的分析，我们已经可以得到结论：Java 本身并不适合用作结构化数据处理的语言。它的 Lambda 机制不支持特征 3，而且作为编译型语言，也不能实现特征 4。

其实，前面说到的 Lambda 语法也不太适合采用编译型语言来实现。编译器不能确定这个写到参数位置的表达式是应该当场计算出表达式的值再传递，还是把整个表达式编译成一个函数传递，需要再设计更多的语法符号加以区分。而解释型语言则没有这个问题，作为参数的表达式是先计算还是遍历集合成员时再计算，可以由函数本身来决定。

SQL 确实是解释型语言。

引入 SPL

**Stream**是 Java8 以官方身份推出的结构化数据处理类库，但并不符合上述的要求。它没有专业的结构化数据类型，缺乏很多重要的结构化数据计算函数，不是解释型语言，不支持动态数据类型，Lambda 语法的接口复杂。

**Kotlin**属于 Java 生态系统的一部分，它在 Stream 的基础上进行了小幅改进，也提供了结构化数据计算类型，但因为结构化数据计算函数不足，不是解释型语言，不支持动态数据类型，Lambda 语法的接口复杂，仍然不是理想的结构化数据计算类库。

**Scala**提供了较丰富的结构化数据计算函数，但编译型语言的特点，也使它不能成为理想的结构化数据计算类库**。** 

那么，Java 生态下还有什么可以用呢？

**集算器 SPL。** 

SPL 是由 Java 解释执行的程序语言，具备丰富的结构化数据计算类库、接口简单的 Lambda 语法和方便易用的动态数据结构，是 Java 下理想的结构化处理类库。

**丰富的集合运算函数**

SPL 提供了专业的结构化数据类型，即序表。和 SQL 的数据表一样，序表是批量记录组成的集合，具有结构化数据类型的一般功能，下面举例说明。

解析源数据并生成序表：  
Orders=T("d:/Orders.csv")

按列名从原序表生成新的序表：  
Orders.new(OrderID, Amount, OrderDate)

计算列：  
Orders.new(OrderID, Amount, year(OrderDate))

字段改名：  
Orders.new(OrderID:ID, SellerId, year(OrderDate):y)

按序号使用字段：  
Orders.groups(year(\_5),\_2; sum(\_4))

序表改名（左关联）  
join@1(Orders:o,SellerId ; Employees:e,EId).groups(e.Dept; sum(o.Amount))

序表支持所有的结构化计算函数，计算结果也同样是序表，而不是 Map 之类的数据类型。比如对分组汇总的结果，继续进行结构化数据处理：

Orders.groups(year(OrderDate):y; sum(Amount):m).new(y:OrderYear, m\*0.2:discount)

在序表的基础上，SPL 提供了丰富的结构化数据计算函数，比如过滤、排序、分组、去重、改名、计算列、关联、子查询、集合计算、有序计算等。这些函数具有强大的计算能力，无须硬编码辅助，就能独立完成计算：

组合查询：  
Orders.select(Amount>1000 && Amount&lt;=3000 && like(Client,"\*bro\*"))

排序：  
Orders.sort(-Client,Amount)

分组汇总：  
Orders.groups(year(OrderDate),Client; sum(Amount))

内关联：  
join(Orders:o,SellerId ; Employees:e,EId).groups(e.Dept; sum(o.Amount))

**简洁的 Lambda\*\***语法 \*\*

SPL 支持接口简单的 Lambda 语法，无须定义函数名和函数体，可以直接用表达式当作函数的参数，比如过滤：  
Orders.select(Amount>1000)

修改业务逻辑时，也不用重构函数，只须简单修改表达式：  
Orders.select(Amount>1000 && Amount&lt;2000)

SPL 是解释型语言，使用参数表达式时不必明确定义参数类型，使 Lambda 接口更简单。比如计算平方和，想在 sum 的过程中算平方，可以直观写作：  
Orders.sum(Amount\*Amount)

和 SQL 类似，SPL 语法也支持在单表计算时直接使用字段名：  
Orders.sort(-Client, Amount)

**动态数据结构**

SPL 是解释型语言，天然支持动态数据结构，可以根据计算结果结构动态生成新序表。特别适合计算列、分组汇总、关联这类计算，比如直接对分组汇总的结果再计算：  
Orders.groups(Client;sum(Amount):amt).select(amt>1000 && like(Client,"\*S\*"))

或直接对关联计算的结果再计算：  
join(Orders:o,SellerId ; Employees:e,Eid).groups(e.Dept; sum(o.Amount))

较复杂的计算通常都要拆成多个步骤，每个中间结果的数据结构几乎都不同。SPL 支持动态数据结构，不必先定义这些中间结果的结构。比如，根据某年的客户回款记录表，计算每个月的回款额都在前 10 名的客户：  
Sales2021.group(month(sellDate)).(~.groups(Client;sum(Amount):sumValue)).(~.sort(-sumValue)) .(~.select(#&lt;=10)).(~.(Client)).isect()

**直接执行 SQL**

SPL 中还实现了 SQL 的解释器，可以直接执行 SQL，从基本的 WHERE、GROUP 到 JOIN、甚至 WITH 都能支持：

```sql
$select * from d:/Orders.csv where (OrderDate<date('2020-01-01') and Amount<=100)or (OrderDate>=date('2020-12-31') and Amount>100)
```

```sql
$select year(OrderDate),Client ,sum(Amount),count(1) from d:/Orders.csv
group by year(OrderDate),Client
having sum(Amount)<=100
```

```cs
$select o.OrderId,o.Client,e.Name e.Dept from d:/Orders.csv o
join d:/Employees.csv e on o.SellerId=e.Eid
```

```sql
$with t as (select Client ,sum(amount) s from d:/Orders.csv group by Client)
select t.Client, t.s, ct.Name, ct.address from t
left join ClientTable ct on t.Client=ct.Client
```

更多语言优势

作为专业的结构化数据处理语言，SPL 不仅覆盖了 SQL 的所有计算能力，在语言方面，还有更强大的优势：

**离散性及其支挂下的更彻底的集合化**

集合化是 SQL 的基本特性，即支持数据以集合的形式参与运算。但 SQL 的离散性很不好，所有集合成员必须作为一个整体参于运算，不能游离在集合之外。而 Java 等高级语言则支持很好的离散性，数组成员可以单独运算。

但是，更彻底的集合化需要离散性来支持，集合成员可以游离在集合之外，并与其它数据随意构成新的集合参与运算 。

SPL 兼具了 SQL 的集合化和 Java 的离散性，从而可以实现更彻底的集合化。

比如，SPL 中很容易表达 “集合的集合”，适合**分组后计算**。比如，找到各科成绩均在前 10 名的学生：

\|  
 | A |
| 1 | =T("score.csv").group(subject) |
| 2 | =A2.(~.rank(score).pselect@a(~&lt;=10)) |
| 3 | =A1.(~(A3(#)).(name)).isect() |

SPL 序表的字段可以存储记录或记录集合，这样可以用**对象引用**的方式，直观地表达关联关系，即使关系再多，也能直观地表达。比如，根据员工表找到女经理下属的男员工：

Employees.select(性别:"男", 部门. 经理. 性别:"女")

**有序计算**是离散性和集合化的典型结合产物，成员的次序在集合中才有意义，这要求集合化，有序计算时又要将每个成员与相邻成员区分开，会强调离散性。SPL 兼具集合化和离散性，天然支持有序计算。

具体来说，SPL 可以按绝对位置引用成员，比如，取第 3 条订单可以写成 Orders(3)，取第 1、3、5 条记录可以写成 Orders(\[1,3,5])。

SPL 也可以按相对位置引用成员，比如，计算每条记录相对于上一条记录的金额增长率：Orders.derive(amount/amount\[-1]-1)

SPL 还可以用 #代表当前记录的序号，比如把员工按序号分成两组，奇数序号一组，偶数序号一组：Employees.group(#%2==1)

**更方便的函数语法**

大量功能强大的结构化数据计算函数，这本来是一件好事，但这会让相似功能的函数不容易区分。无形中提高了学习难度。

SPL 提供了特有的函数选项语法，功能相似的函数可以共用一个函数名，只用**函数选项**区分差别。比如 select 函数的基本功能是过滤，如果只过滤出符合条件的第 1 条记录，只须使用选项 @1：  
Orders.select@1(Amount>1000)

数据量较大时，用并行计算提高性能，只须改为选项 @m：  
Orders.select@m(Amount>1000)

对排序过的数据，用二分法进行快速过滤，可用 @b：  
Orders.select@b(Amount>1000)

函数选项还可以组合搭配，比如：  
Orders.select@1b(Amount>1000)

结构化运算函数的参数常常很复杂，比如 SQL 就需要用各种关键字把一条语句的参数分隔成多个组，但这会动用很多关键字，也使语句结构不统一。

SPL 支持**层次参数**，通过分号、逗号、冒号自高而低将参数分为三层，用通用的方式简化复杂参数的表达：  
join(Orders:o,SellerId ; Employees:e,EId)

**扩展的 Lambda\*\***语法 \*\*

普通的 Lambda 语法不仅要指明表达式（即函数形式的参数），还必须完整地定义表达式本身的参数，否则在数学形式上不够严密，这就让 Lambda 语法很繁琐。比如用循环函数 select 过滤集合 A，只保留值为偶数的成员，一般形式是：  
A.select(f(x):{x%2==0} )

这里的表达式是 x%2==0，表达式的参数是 f(x) 里的 x，x 代表集合 A 里的成员，即循环变量。

SPL 用**固定符号 \*\***~\***\* 代表循环变量**，当参数是循环变量时就无须再定义参数了。在 SPL 中，上面的 Lambda 语法可以简写作：A.select(~ %2==0)

普通 Lambda 语法必须定义表达式用到的每一个参数，除了循环变量外，常用的参数还有循环计数，如果把循环计数也定义到 Lambda 中，代码就更繁琐了。

SPL 用**固定符号 \*\***#\***\* 代表循环变量**。比如，用函数 select 过滤集合 A，只保留序号是偶数的成员，SPL 可以写作：A.select(# %2==0)

相对位置经常出现在难度较大的计算中，而且相对位置本身就很难计算，当要使用相对位置时，参数的写法将非常繁琐。

SPL 用**固定形式 \*\***\[\***\* 序号]\*\***代表相对位置 \*\*：

\|  
 | A | B |
| 1 | =T("Orders.txt") | / 订单序表 |
| 2 | =A1.groups(year(Date):y,month(Date):m;   sum(Amount):amt) | / 按年月分组汇总 |
| 3 | =A2.derive(amt/amt\[-1]:lrr, amt\[-1:1].avg():ma) | / 计算比上期和移动平均 |

无缝集成、低耦合、热切换

作为用 Java 解释的脚本语言，SPL 提供了 JDBC 驱动，可以无缝集成进 Java 应用程中。

简单语句可以像 SQL 一样直接执行：

```powershell
…
Class.forName("com.esproc.jdbc.InternalDriver");
Connection conn =DriverManager.getConnection("jdbc:esproc:local://");
PrepareStatement st = conn.prepareStatement("=T(\"D:/Orders.txt\").select(Amount>1000 && Amount<=3000 && like(Client,\"*S*\"))");
ResultSet result=st.execute();
...
```

复杂计算可以存成脚本文件，以存储过程方式调用  

```properties
…
Class.forName("com.esproc.jdbc.InternalDriver");
Connection conn =DriverManager.getConnection("jdbc:esproc:local://");
Statement st = connection.();
CallableStatement st = conn.prepareCall("{call splscript1(?, ?)}");
st.setObject(1, 3000);
st.setObject(2, 5000); 
ResultSet result=st.execute();
...
```

将脚本外置于 Java 程序，一方面可以降低代码耦合性，另一方面利用解释执行的特点还可以支持热切换，业务逻辑变动时只要修改脚本即可立即生效，不像使用 Java 时常常要重启整个应用。这种机制特别适合编写微服务架构中的业务处理逻辑。

![](https://mmbiz.qpic.cn/mmbiz_png/3dBJseibic1ce5ibR15d3hewLL1THeqrvJfSEZ4XjIY2mh7J1F3iawVj2xoXQoJCmAjd3D6SSzxicictFQU8pLMz3dMg/640?wx_fmt=png)

**重磅！开源 SPL 交流群成立了**

简单好用的 SPL 开源啦！

为了给感兴趣的小伙伴们提供一个相互交流的平台，

特地开通了交流群（群完全免费，不广告不卖课）

需要进群的朋友，可长按扫描下方二维码

![](https://mmbiz.qpic.cn/mmbiz_jpg/3dBJseibic1ce5ibR15d3hewLL1THeqrvJfo8f5bn3qxvSuaadhicl0sQK0iaPTqevTtmDTfRBlWmY978XhhmslyFEQ/640?wx_fmt=jpeg)

**本文感兴趣的朋友，请转到阅读原文去收藏 ^\_^** 
 [https://mp.weixin.qq.com/s/yQ7aWR4zzacFe2DLLOZQfg](https://mp.weixin.qq.com/s/yQ7aWR4zzacFe2DLLOZQfg)
