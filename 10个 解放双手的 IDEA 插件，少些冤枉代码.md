# 10个 解放双手的 IDEA 插件，少些冤枉代码
点击 “程序员内点事” 关注，选择“ [设置星标](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486188&idx=3&sn=f160d91ea23e5113e6077c500a2e30c4&chksm=fa49755dcd3efc4bf4f566fbbbf74c191d0b79f2d3222fd211bc52d80b5ef127f52b1158ed71&scene=21#wechat_redirect) ”

坚持学习，好文每日送达！

> ❝
>
> 友情提示：插件虽好，可不要贪装哦，装多了会 卡 、卡 、卡 ~
>
> ❞

### 正经干活用的

分享一点自己工作中得心应手的`IDEA`插件，可不是在插件商店随随便便搜的，都经过实战检验，用过的都说好。可能有一些大家用过的就快速划过就行了。

#### 1、GenerateAllSetter

实际的开发中，可能会经常为某个对象中多个属性进行 `set` 赋值，尽管可以用`BeanUtil.copyProperties()`方式批量赋值，但这种方式有一些弊端，存在属性值覆盖的问题，所以不少场景还是需要手动 `set`。如果一个对象属性太多 `set` 起来也很痛苦，`GenerateAllSetter`可以一键将对象属性都 `set` 出来。

快捷键：`Alt+Enter`![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jvJtowaYibfS8RPl0ZLz6DLCSMRhFIibDscl8wPM2xmpNY6OibPoxtOIrw/640?wx_fmt=gif)

#### 2、Alibaba Java Coding Guidelines

阿里出品的《Java 开发手册》时下已经成为了很多公司新员工入职必读的手册，前一段阿里发布了《Java 开发手册 (泰山版)》， 又一次对`Java`开发规范做了完善。不过，又臭又长的手册背下来是不可能的，但集成到`IDEA`开发工具中就方便很多。

举个栗子：开发手册上不允许用`Executors`去创建线程池，而是通过`ThreadPoolExecutor`的方式。![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jkqwO3cjiaZzx0TU56d7bvJGxqJTLw7cpuSsJbdHL58AWfqRCAD0CGeA/640?wx_fmt=png)
集成插件后会再去使用`Executors`去创建线程池会有如下的提示。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jRvLKQ56QrrgqHFklu5oWMRWX1nk3mGkGt7u8nmAL1ib3lSKK5jibrwaQ/640?wx_fmt=gif)

#### 3、GsonFormat

`GsonFormat` 个人觉得是一个非常非常实用的插件，它可以将`JSON`字符串自动转换成`Java`实体类。特别是在和其他系统对接时，往往以`JSON`格式传输数据，而我们需要用`Java`实体接收数据入库或者包装转发，如果字段太多一个一个编写那就太麻烦了。

快捷键：`Alt+ S`

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jn8bSQh4ibJs8jgnzYdibicSr5vibNv0rMpZ9Fibkia8Yyk85sWeFuXEIT54w/640?wx_fmt=gif)

在这里插入图片描述

#### 4、Maven Helper

`Maven Helper` 是解决`Maven`依赖冲突的利器，可以快速查找项目中的依赖冲突。安装后打开`pom`文件，底部有 `Dependency Analyzer` 视图。显示红色表示存在依赖冲突，点进去直接在包上右键`Exclude`排除，`pom`文件中会做出相应排除包的操作。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jEBlkCMNAe78x4n6myzHarX8OhayUc5jibmsxp6ehAFQyOa5CpQJXZ6w/640?wx_fmt=gif)

在这里插入图片描述

-   Conflicts(冲突)
-   All Dependencies as List(列表形式查看所有依赖)
-   All Dependencies as Tree(树结构查看所有依赖)，并且这个页面还支持搜索。

#### 5、Codota

用了`Codota` 后不再怕对`API`不会用，举个栗子：当我们用`stream().filter()`对`List`操作，可是对`filter()`用法不熟，按常理我们会百度一下，而用`Codota` 会提示很多`filter()`用法，节省不少查阅资料的时间。

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jSoyI9Ss0lrHZqBGicxHtOzZcVUKvHeGZPHMpKAB2jn28kiaIwa9dXkaA/640?wx_fmt=gif)

在这里插入图片描述

#### 6、Free MyBatis Plugin

在使用`MyBatis` 作为持久框架时有一个尴尬的问题：`SQL` `xml`文件和定义的`Java`接口无法相互跳转，不能像 Java 接口间调用那样，只能全局搜索稍显麻烦。`Free MyBatis Plugin`将两者之间进行关联。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jnlJwZibuCkACRjZZL9WLFRz0ySxULiaDGvOIOico2LOiaKsC05Ch1ArK5w/640?wx_fmt=gif)

#### 7、IntelliJad

`IntelliJad`是一个 Java class 文件的反编译工具，需要在 `setting` 中设置本地`Java` `jad.exe`工具的地址。随便找个`Jar`架包选择`class`文件右键`Decompile`，会出现反编译的结果。

#### 8、Properties to YAML Converter

将`Properties` 配置文件一键转换成`YAML` 文件，很实用的一个插件。**「注意：要提前备份原`Properties` 文件」**![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jUatibmXLZDbCrraesJiaib67oJdWhCyCIQ5ibJWqJyiakBHyfuGB3RJhYew/640?wx_fmt=gif)

#### 9、Lombok

`Lombok` 插件应该比较熟，它替我们解决了那些繁琐又重复的代码，比如`Setter`、`Getter`、`toString`、`equals`等方法。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jmOUib6fwHcicmxvaXsjptnxroDZPSRCb1VVc1YicwZfgk0D0MyqLyDuvg/640?wx_fmt=gif)

#### 10、CodeGlance

`CodeGlance` 是一款代码编辑区迷你缩放图插件，可以很方便的知道我们方法大致在什么位置。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jHcpnChqm8zqJEiayKxHgC7DbYKzmFBtVdUN9Xd6v4fEZ0y8WGmZb9bg/640?wx_fmt=gif)

`IDEA`还有不少的开发小技巧，有助于我们少些代码，不知道大家有没有发现？变量后`.`可以联想提示，而在联想列表的最后边有很多简洁的命令。

例如：

`list.sout` =  `System.out.println(list);`

`list.var` =  `List<User> list1 = list`

`list.nn = list.if (list != null)`

......![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2j9TKtxFZOPvXq0K3eruPIcfkW4bSZ5E2QqSk3oG2OibXyyE6TEphwgog/640?wx_fmt=gif)

### 装 X 用的

下边这些就属于装 X 神器了，可以根据个人的喜好来耍一下。

#### 1、Material Theme UI

使用插件后界面图标样式都会变的很漂亮。![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jPVuC8kgTOzxDFkwUicdBXbkn60L4juVSuXY1btpJ7Af30nibr7stcgFw/640?wx_fmt=png)

#### 2、activate-power-mode

这个震动的效果看似很是酷炫，可写了十分钟代码我就快被它晃悠吐了。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jFJxM9FkYQ41OGABCzS4fsXCjPCUzZp2Z09uYL9p5wZ26HibiaMfbpfyA/640?wx_fmt=gif)

#### 3、Nyan progress bar

会让`IDEA`所有进度条都变得萌萌的，但我并不建议你安装因为会很卡，不知道是不是只有我这样。![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jcvPg5Ood2TlW3TBuV1F6A1gpNWUddeJicoIso5GibmQviassrs1TDfrBw/640?wx_fmt=gif)

#### 4、Rainbow Brackets

彩虹颜色的括号，看着很舒服，有点赏心悦目的感觉。![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aP8BzcrS7KTgLndFGth9F2jVAw6S7wDqtW8uEsMdSPyPvwYUicppYpJNywiaWsfhyxJiaicjLRuyHQMuA/640?wx_fmt=png)

整理了几百本各类技术电子书相送 ，嘘~，**「免费」** 送给小伙伴们。关注公众号回复【**666**】自行领取。和一些小伙伴们建了一个技术交流群，一起探讨技术、分享技术资料，旨在共同学习进步，如果感兴趣就扫码加入我们吧！

 [https://mp.weixin.qq.com/s/aWQDlujb-j1ufdraA-bC6g](https://mp.weixin.qq.com/s/aWQDlujb-j1ufdraA-bC6g)
