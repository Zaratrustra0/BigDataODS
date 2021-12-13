# 附模板和代码 | Excel数据模型自动生成Hive建表语句
点击上方蓝字关注我们！

平时多流汗，战时少流血  

最近在「空白女侠」公号上看到她回答了大家会困扰的精力问题，比如为什么我（空白女侠）能同时做那么多事情，精力那么充沛？工作中遵循一个真理：

**复杂的事情简单化，简单的事情标准化，标准的事情流程化，流程化的事情工具化，工具化的事情自动化，无法自动化的事情外包化。** 

数据开发过程中，有些过程是可以工具化来提高工作效率，腾出更多的时间和精力去提高自己（摸鱼～）

工具化

在日常数据开发过程中，会经常需要根据数据模型编写建表语句，每次写建表语句都会用几分钟的时间，而且还容易出一些低级的错误，于是打算做个 Excel 模板，把表字段、表分区、表名写在里面，通过程序自动生成建表语句。效果如下图：

![](https://mmbiz.qpic.cn/mmbiz_gif/zKQJJopg9UM9NLeHCtfrZdh1JQrttBJ5s8icFVUic6KBj6kXImKs2ib39weCTBLtpTmkcaNPZLLNicpA2DSzFsq3Eg/640?wx_fmt=gif)

数据模型生成建表语句

制作模板

主要包括表名、表中文描述、字段名、数据类型、字段说明、是否可以为空、唯一主键、表分区等写在里面（可以根据实际情况进行调整）

模板组成分为三个部分：

-   1-3 行是表名、表中文描述和模板列说明
-   4-19 行为创建表字段的基本信息
-   20 行为分区字段

![](https://mmbiz.qpic.cn/mmbiz_png/zKQJJopg9UM9NLeHCtfrZdh1JQrttBJ5S78Z65FnDnGLjPtLlUVNZ6ygib6s9ToOrVBNPHNryY7MKVEVhAJcrsQ/640?wx_fmt=png)

数据模型模板

实现思路

获取指定目录下的数据模型文件，约定为以「数据模型. xls」或「数据模型. xlsx」文件名结尾的 Excel 文件

循环遍历每个模板文件，根据模板组成规范，解析文件，拼接 SQL 语句，生成建表语句文件

生成结果

    CREATE EXTERNAL TABLE ods_cbonddescription (    object_id string  COMMENT '对象ID',    b_info_fullname string  COMMENT '债券名称',    s_info_name string  COMMENT '债券简称',    b_info_issuer string  COMMENT '发行人',    b_issue_announcement string  COMMENT '发行公告日',    b_issue_firstissue string  COMMENT '发行起始日',    b_issue_lastissue string  COMMENT '发行截至日',    b_issue_amountplan bigint  COMMENT '计划发行总量（亿元）',    b_issue_amountact bigint  COMMENT '实际发行总量（亿元）',    b_info_issueprice bigint  COMMENT '发行价格',    b_info_par bigint  COMMENT '面值',    b_info_term_year int  COMMENT '债券期限（年）',    b_info_term_day int  COMMENT '债券期限（天）',    b_info_paymentdate int  COMMENT '兑付日',    b_info_paymenttype int  COMMENT '计息方式',    s_info_exchmarket string  COMMENT '交易所')COMMENT '债券基本信息' PARTITIONED BY(dt string) ROW FORMAT DELIMITED '\t' STORED AS ORC LOCATION 'hdfs://host:8020/dw/ods/ods_cbonddescription';

结语

如果大家在工作中使用了什么样的模板，可以一起交流如何形成一定的工具化的脚本来提高我们工作的效率。

最后，关注公众号，回复「666」获取模板和代码。

\[

![](https://mmbiz.qpic.cn/mmbiz_jpg/zKQJJopg9UNr0Fo3XCL6y5c1h1z4UOHKnfRIXRcVHQZmob2yteTcQdFBGGT6TkF7icjhxhgNsFawSdKUib2Nu3Cg/640?wx_fmt=jpeg)

大数据 ETL 处理工具 Kettle 入门实践

]([http://mp.weixin.qq.com/s?\_\_biz=MzA5Njk3Njc5Mw==&mid=2247486730&idx=1&sn=af2bcd9af2ff6fbd8aa78d22e0da7578&chksm=90a6a7fca7d12eead4a352e931d0cad36944cba9679fef053690beb4f8c0c519b9ae981e4c73&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzA5Njk3Njc5Mw==&mid=2247486730&idx=1&sn=af2bcd9af2ff6fbd8aa78d22e0da7578&chksm=90a6a7fca7d12eead4a352e931d0cad36944cba9679fef053690beb4f8c0c519b9ae981e4c73&scene=21#wechat_redirect))

\[

![](https://mmbiz.qpic.cn/mmbiz_jpg/zKQJJopg9UMgL7Kib2DmOY3JHZib0Yvf3zkPaLvnvpdd4u4aAyeia0rgpML0rZSiaFn5BzaVqZNJJ4twXn10DY4hkw/640?wx_fmt=jpeg)

大数据 ETL 处理工具 Kettle 的核心概念

]([http://mp.weixin.qq.com/s?\_\_biz=MzA5Njk3Njc5Mw==&mid=2247486749&idx=1&sn=f1b1adb020ed300e24136ed5970041a8&chksm=90a6a7eba7d12efdba39d9db8511690f6df683872cdb716edc1f8d1a3bf66dc60374d96faf68&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzA5Njk3Njc5Mw==&mid=2247486749&idx=1&sn=f1b1adb020ed300e24136ed5970041a8&chksm=90a6a7eba7d12efdba39d9db8511690f6df683872cdb716edc1f8d1a3bf66dc60374d96faf68&scene=21#wechat_redirect))

\[

![](https://mmbiz.qpic.cn/mmbiz_jpg/zKQJJopg9UPwU0v8JMf314FicWSPDFian2RiceF8zXI7ZI5VET3ziaDvzaiapVykNKwf4ibgRT3Q1iajN6A92vxibjrjmQ/640?wx_fmt=jpeg)

大数据 ETL 处理工具 Kettle 完成一个作业任务

]([http://mp.weixin.qq.com/s?\_\_biz=MzA5Njk3Njc5Mw==&mid=2247486774&idx=1&sn=05fbeb3be7c9a5b130c15968d7a0e24f&chksm=90a6a7c0a7d12ed64d0422326be8b560c1c70722083058315077af8f457a13913d363811f19a&scene=21#wechat_redirect](http://mp.weixin.qq.com/s?__biz=MzA5Njk3Njc5Mw==&mid=2247486774&idx=1&sn=05fbeb3be7c9a5b130c15968d7a0e24f&chksm=90a6a7c0a7d12ed64d0422326be8b560c1c70722083058315077af8f457a13913d363811f19a&scene=21#wechat_redirect))

🧐**分享、点赞、在看**，给个**3 连击**呗！**👇** 
 [https://mp.weixin.qq.com/s/nqH_fEkvLSPZnyrBFi7SvQ](https://mp.weixin.qq.com/s/nqH_fEkvLSPZnyrBFi7SvQ)
