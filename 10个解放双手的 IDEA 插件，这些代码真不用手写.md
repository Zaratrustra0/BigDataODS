# 10个解放双手的 IDEA 插件，这些代码真不用手写
鸽了很久没发文，不写文章的日子真的好惬意，每天也不用愁着写点什么，不用为那点可怜的阅读量发愁，不那么熬夜，留出了更多时间陪家人。

不过，惬意过后就是极度的焦虑，看着圈子里这些卷怪朋友们没日没夜的更文，比你优秀的人比你更努力，这本身就是一件很有压力的事情。

总是给自己找借口，哎~ ，工作忙哪来时间弄，可越是这么自我安慰就越没时间做，打工人哪来大块大块时间让你做这些，真正热爱一件事就是要全身心的投入，时间挤一挤总会有的，贵在坚持吧！

虽然慢步走，但我一直在路上~

* * *

之前分享过一篇 [《10 个 解放双手的 IDEA 插件，少些冤枉代码》](https://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247484298&idx=1&sn=f6c0269b344d327ca0113531e3564864&scene=21#wechat_redirect)反响还不错，这里再介绍 10 个我用着还算顺手的`IDEA`插件，绝对实用不花哨。

### aiXcoder

`aiXcoder` 一款国产代码开发工具，提供了比较强大的代码补全、预测的功能，它的宗旨就是让我们少些代码，能自动生成的绝不手写，上手感受下就会爱上它。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xnlCXsh5Qhia54xgedFlMaqmGPniauVYhk0ibIyL2mY0q5sviccosmL7deQ/640?wx_fmt=gif)

简单演示 功能远不止于此

实际开发中我会结合`IDEA`的`postfix completion`和`aiXcoder`配置使用，`IDEA`本身就已经提供了许多快速补全的快捷方式，不过我发现组内很多人并没有真正用起来。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xFfkloyd0ibFO2RVZGZJqumwxm6D1EaCDrGADEpVojMRjLYQxpPPrkIA/640?wx_fmt=gif)

也可以自行定义快捷方式生成的代码块。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xyF17aW5gNKibutIMRXBlRAbdIJsfdLpic5GlGKUTxH9eq6cGd0ZKAFyQ/640?wx_fmt=png)

`aiXcoder`支持相似代码搜索功能，如果哪个`API`不会用，直接选中右键全网搜索实用案例。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84x5HMicAucBQDK4JcOGbV1gDtlP427skbQhXH1GrmWO4LDXZiaB0hSkqHw/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xqRiaBhRViczSDY7J6trtjBDq6U9IJHYv5vQibpDQ3xcyflcCpaoX9xWhA/640?wx_fmt=png)

### Java Stream Debugger

`Java8`的`stream API`很大程度的简化了我们的代码量，可在使用过程中总会出现奇奇怪怪的`bug`而且不能`debug`。

`Java Stream Debugger`支持了对`stream API`的调试，可以清晰的看到每一步操作数据的变化过程。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84x2I1wdDMLa5VsIv8BqmvPQMWrRhWVNrKvjibmhmSfX6FbTjr4fyJYOWA/640?wx_fmt=png)

### easy_javadoc

`easy_javadoc`一个可以快速为`Java`的类、方法、属性加注释的插件，还支持自定义注释样式，`IDEA`自身的`Live Templates`也支持，不过操作稍显繁琐，使用时效率不太高。

在为类、方法、属性加注释时，不仅会生成注释，还是会将对应变量、类、方法翻译成中文名，不过翻译的怎么样还要取决于你的命名水平。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xdV3a2QP1yK3R9NDAx8RC63K9EFbUdWSFApQuQFyvkiabjW8dreSyNGA/640?wx_fmt=gif)

快捷键：`crtl + \`

是不是觉得一点点加注释效率太低了，你也可以尝试批量添加注释。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xFB9IIDvAUxMWN2mQiaZpDgJdUboDBZu1ea9Xc9MaAoH69b3ZCWAy5zg/640?wx_fmt=gif)

快捷键：`crtl + shift + \`

如果现有的注释样式不适合你，可以自定义你的注释模板。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xaj3krWI6PnpibILbHK5eFtY5NSMr7KF1libWz4GWXAXVaaCmynoMBtMw/640?wx_fmt=png)

### Easy Code

`Easy Code`我个人在写博客案例`demo`时用的比较多，它可以快速的将数据库表映射成 Java 中的`entity`、`controller`、`service`、`dao`、`mapper`等文件，少量编码实现快速开发。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84x1wliay8CAFWc6YnKZCgdNqP6lRndaxt19SfY4SoFNpb8CqVjKJicpqpg/640?wx_fmt=png)

先用`database`连接数据库，在对应表上直接右键执行`EasyCode`即可生成相应 Java 代码，真的很方便。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xa50AySzBpDBpJLzF77IaXXLkIsKPyRzN82Ts4TfvW7Euuicia47T40NA/640?wx_fmt=png)

### Restfultoolkit

`Restfultoolkit`一套`RESTful`服务开发辅助工具集，维护项目通常会涉及到查找一个请求所对应的类，一般用`ctrl + shift + f`进行全局搜索，但是如果项目文件太多，这种查找方式的效率就很低。

`Restfultoolkit`管理项目中全部的请求链接，可以快速查找。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xSibgLF4lV58Qdn5Op7uOpCQChFIguYaC9D5KXSwcoTh3CIic1bMvHfoQ/640?wx_fmt=png)

快捷键：`ctrl+ alt + n`

可以复制当前请求的`全路径`和`JSON`格式的参数，开发测试中非常的实用。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xaoGAJh0BRqclHsZsSQCnL2fs5q1DtZksDpa0OQnRjCDdaKt715kT2A/640?wx_fmt=gif)

`IDEA`右侧会出现一栏`RestServices`，这里有整个项目的`http`请求，还会显示每个请求的入参、出参`JSON`数据，可以进行简单的模拟请求。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xbz888iavWzgFKlg7549ThL5kFH7mXdBggTL5icRAbwddjdLuUIQ0IDIw/640?wx_fmt=png)

### Key promoter X

`Key promoter X`是`IDEA`的快捷键提示插件，这是我个人非常喜欢的一个功能，它让我快速的记忆了很多操作的快捷键。当你点击某个功能且该功能有快捷键时，会提示当前操作的快捷方式。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xbt90eDiaPZEyEKia9EHPmiaC5DdINy5niaiajSOAFbtAlqqfAQicNWooKS5A/640?wx_fmt=gif)

### String Manipulation

`String Manipulation`一个比较实用的字符串转换工具，比如我们平时的变量命名可以一键转换驼峰等格式，还支持对字符串的各种加、解密（`MD5`、`Base64`等）操作。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xqaXglxibZdial7icCSsWOot21wYib1icKyFh61JgPTMxQGnyNRE7oh09jrg/640?wx_fmt=gif)
快捷键：`alt + m`

### Translation

`Translation`一个很方便的翻译插件，比如选中代码、控制台的报错信息可直接翻译。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xf896LCLsLjMj8ialkKUOXp7bAvG8v59AyADoQ8q3dhOo9AAsbgiapicvw/640?wx_fmt=gif)

### Git Auto Pull

团队多人开发项目时，由于频繁提交代码，等我在`commit`本地代码的时必须先进行`pull`，否则就会代码冲突产生`merge`记录。

`GitAutoPull`插件帮我们在`push`前先进行`pull`，避免了不必要的代码冲突。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xKcnicoc84NOLjfDgs6x40ziaFhxUmBELfDMzwO5xDr0ZWN6eprsPEeOg/640?wx_fmt=png)

### .ignore

当我们在向`github`提交代码时，有一些文件不希望一并提交，这时候我们可以创建一个`.gitignore`文件来忽略某些文件的提交。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xgq2Oam7PMUQT5UMWOm12xoJZmwCG9dEErtcgo4piaguUibLJ9aJLnNJg/640?wx_fmt=png)

也可以添加指定文件到`.gitignore`中，被忽略的文件将变成灰色。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aOYVlIj0gIc1GibJk3uof84xNnTxn7kM38qnGdhpNMCP6yFYsqV8eL3p1hwSuPcnicl1xdKNU8r1cFg/640?wx_fmt=png)

以上就是本次分享的 10 个比较实用的`IDEA`插件，对提升开发效率还是有一定帮助的。

**温馨提示**：插件虽好但也不要贪装，装多了真的会卡、卡、卡！

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aN0sXSNoiaIdgDia04G2I1anY7vGagHzjYQrSoPxNdb1xhwzeokkRGJPxw46GQI1DabFfek94j2fnIA/640?wx_fmt=gif)

你的每个赞和在看，我都喜欢！

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aNhvrFicKVM7CoIQaorevsJuIo7hGvmZUSHCIaicBxIKDMlPqvyfprCa9svys6akhIv7ren0QJSqvaA/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/FLC-5gqcnnMnBpRlWlNvLQ](https://mp.weixin.qq.com/s/FLC-5gqcnnMnBpRlWlNvLQ)
