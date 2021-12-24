# 工作中常见的 Hive 复合数据类型
![](https://mmbiz.qpic.cn/sz_mmbiz_png/NOM5HN2icXzxRe5tibNONfeDZpRealiaOOPfAreHtbs3eLZ61LANNh5EmW9f3YfUeEt4SSX2R8ycBqL2cvSaVMeug/640?wx_fmt=png)

今天想跟大家聊一聊 Hive 中常见的两种复合数据类型：**Map、Array**。

全篇共计 2310 字，预计阅读时间 6 分钟。  

# 1. 背景

昨天下午，同事说要跟我请教一个问题，有大概如下一张表，问：如何能判断字段 B 中包含 56？（其中字段 A 的数据类型为 string，字段 B 的数据类型为 array）

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG8226vChAnJJGQiccZ4vGXPkywrvVVKWLnicLl9CPFwbqqicxPbP84NiaRBNHw/640?wx_fmt=png)

我：

你 like 一下？不对哦，万一有 156，256 这种呢？  

不然把 array split 一下？不行哦，array 长度不固定，且有的挺长。  

有没有函数啊？应该会有函数吧？  

一问 ETL 大佬，**array_contains()**一下就行了。

你看，总有那些方便、好用但你在字面上没见过的函数，就如同之前说的窗口函数，有很多，可能你无法一一认识它们，但你可以在实操、向别人请教中学到。  

学到了**array_contains()**这个函数，我微微有点兴奋，想着借此把接触过的一些跟 Hive 复合数据类型相关的函数或操作记录一下，如果你早就知道了，那你真的太棒了，如果你还不知道，恭喜你！学了新知识！

# 2. Array 相关

Array 就是数组，跟你在其他编程语言中认识的一样，里面存储了 0 个或多个元素，比如\['幼儿园','小学','初中','高中', 大学']，这就是一个 array。  

**1. Array 的创建**

我去翻了翻别人的资料，一般都是长这个样子。  

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG82295kC45TuKXNUYxal5dlRz7h7PD3765a7Te8cJCDU5icgAS4Moukh9JA/640?wx_fmt=png)

我在实际工作中一般直接接触的公司平台，只需要 HQL 写好，粘贴进对应位置进行建表操作就 OK 了，一般我的 array 类型的数据来源有两个，第一个是从上游表中 select 某些字段，其类型是 array，所以我复制过来的时候也需要搞一个 array 类型的字段。第二个是在处理一些量很大的数据时，比如用户日志，会根据用户 id 将其某些信息进行整合，存储到一个 array 中。  

所以我一般建表用到 array 的时候就是这样子。

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG8220ZLfiaEic0xD2571Oyzoh60WwM7LXmSpoVD2j1sUk3qvTqW5VndDaftw/640?wx_fmt=png)

**2. Array 的一些操作**

对于 array 的一些操作，主要说几个日常中常见的，涉及 array 的生成、拆分、查找、及其他的几个操作函数。

**A. Array 的生成**  

我在刚刚 array 的创建中提及了一个函数**collect_set()**，使用这个函数我们就可以根据不同的 key 进行分组，将其他信息合并为一个 array。  

除此之外，也可以采用**collect_list()、concat_ws()、group_concat()**。

它们之间的区别是：  

**collect_set()**对合并后的 array 中的数据去重；

**collect_list()**对合并后的的 array 中的数据不去重；

**concat_ws()**可自定义分隔符进行 array 的生成；  

**group_concat()**可自定义分隔符进行 array 的生成；

现实中按某列分组聚合时，记得别忘记 group by 列。  

**B. Array 的拆分**

关于 array 的拆分，可以使用**split()**和**lateral view explode**，即可实现 A 中合并数组的拆分，恢复原貌。

**C. Array 的查找**

与其叫 array 的查找，不如叫在 array 中查找，回到背景里遇到的那个问题，想着数组中快速找到某一字符，直接上**array_contains()**函数就行，无需拆分，快速实现 array 中数据定位。

除此之外，也可以试试**find_in_set()**。

**D. Array 的其他操作**  

还想提两个可能会用到的函数：**sort_array()**和**size()**，实现对 array 中数据的排序和计数操作，其中需要注意的是 sort_array() 只能进行**升序**操作。

# 3. Map 相关

Map 又叫字典，也跟其他编程语言中认识的一样，map 里以 key-value 的形式存储了对应的数据。

**1. Map 的创建**

同样还是翻了别人的资料，一般长这样。

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG822zlTWHJLnqyyqYD2oVKSsXIwIc71mib3QZmwERrhmOqhskbotHUibib15A/640?wx_fmt=png)

与 array 相似，实际工作中，我的 map 类型的数据来源一般只有一个，那就是 select 上游表的数据。埋点的时候一些次要信息可能会一股脑地塞进某个 map 中，经过数仓处理，到业务层面来，也是以 map 类型存储。

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG822qcSFkSbHjBVsib8icETUEcIPXXkTQRbkyrYFCN19ygEJ5mcWVnjAkNZQ/640?wx_fmt=png)

**2. Map 的一些操作**

对于 Map 的一些操作，也主要说几个日常中常见的，如 map 的查询、计数。

**size()**函数既可以对 array 的长度进行统计，也可以对 map 的长度进行统计；  

**map_keys()**函数可得到 map 中所有的 key 值，格式为 array；

**map_values()**函数可得到 map 中所有的 value 值，格式为 array；

既然如此，想从 map 中查询到某个值，是不是也可以用 array_contains()？答案是 YES！先用相应函数转为 array，再查询定位具体值。

# 4. 结尾

还想介绍一个在我日常工作中经常用到的函数**get_json_object()**，用于对 json 字符串进行解析。表中常有一些声称自己是 string 类型的字段，以为很友善，展开一看长得可吓人，比如：  

![](https://mmbiz.qpic.cn/mmbiz_png/rfaNuFeWzwBiamgaOkCG9h0WctH4oG822zV0wGAkhoFrq3wQQDlefNOPU8CibKviacPljbNj91y2LgEGRKTBEFy5g/640?wx_fmt=png)

要想从 p 中取 email 地址，就得：

```cs
select get_json_object(p, '$.email') from table;
```

要想取到 bag 的 price 呢？

```sql
select get_json_object(p, '$.store.bag.price') from table;
```

上述例子都是解析了 p 中的一个字段，若是想同时解析 store、email 和 owner 字段呢？当然可以用三个**get_json_object()**实现，也可以使用**json_tuple**结合**lateral view**进行多个字段的解析。

* * *

我是东哥，最近正在原创👉「[pandas100 个骚操作](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzUzODYwMDAzNA==&action=getalbum&album_id=1699019347278561282#wechat_redirect)」系列话题，欢迎订阅。订阅后，文章更新可第一时间推送至订阅号，每篇都不错过。

最后给大家**分享《100 本 Python 电子书》**，包括 Python 编程技巧、数据分析、爬虫、Web 开发、机器学习、深度学习。

现在免费分享出来，有需要的读者可以下载学习，在下面的公众号「**GitHuboy**」里回复关键字：**Python**，就行。

**拓展阅读**

[终于把所有的 Python 库，都整理出来啦！](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247524491&idx=2&sn=11834714ba8bb015e60060d781691a3f&chksm=fad71786cda09e9009743e8ea9cf5e4fa2435ab3d73ca298c07ebfeac9c336b10060c89564eb&scene=21#wechat_redirect)

[知乎高赞：拼多多和国家电网，选哪个？](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247524478&idx=1&sn=3a79c4fcc99bb217f5579d1dbf178e22&chksm=fad71773cda09e6520f814e2471f7701d36eebbab9f61d4fa8bba5868e4ddb71cf1174c72dc8&scene=21#wechat_redirect)

[太酷炫了，我用 Python 画出了北上广深的地铁路线动态图](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247524224&idx=2&sn=5b24f08e45c4ab7d381678dcd36dcb87&chksm=fad7e88dcda0619b5334660b9ec98f6d8833a18808917ae4891013e288c6e950d3b28cea3cd5&scene=21#wechat_redirect)

[详解 16 个 pandas 函数，让你的 “数据清洗” 能力提高 100 倍！](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247524113&idx=1&sn=806bfda0138bdae0ab15d4988eacba45&chksm=fad7e81ccda0610a68d53142cf8b6fadbacebe15eb3427d2855280d78a2785df16208283416d&scene=21#wechat_redirect)

[吊打 Pyecharts，这个新 Python 绘图库竟然这么漂亮](http://mp.weixin.qq.com/s?__biz=MzUzODYwMDAzNA==&mid=2247524113&idx=2&sn=2005fdce48ff6449512f58a3e77d80de&chksm=fad7e81ccda0610af714f30d02b8ac92bd8eb218668ab828bbc909d1e8771ea4d91fb09c85a7&scene=21#wechat_redirect) 
 [https://mp.weixin.qq.com/s/3vj3KbTV4MfAJOC1k_xZxQ](https://mp.weixin.qq.com/s/3vj3KbTV4MfAJOC1k_xZxQ)
