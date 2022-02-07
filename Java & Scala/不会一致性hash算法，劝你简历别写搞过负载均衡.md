# 不会一致性hash算法，劝你简历别写搞过负载均衡
**大家好，我是小富~**  

这两天看到技术群里，有小伙伴在讨论一致性 hash 算法的问题，正愁没啥写的题目就来了，那就简单介绍下它的原理。下边我们以分布式缓存中经典场景举例，面试中也是经常提及的一些话题，看看什么是一致性 hash 算法以及它有那些过人之处。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibFu0dUQeB5uNbC4hCJzfjuYOxjNFAGVCRO8u3mBoLO9a69X2ibW7l24w/640?wx_fmt=png)

### 构建场景

假如我们有三台缓存服务器编号`node0`、`node1`、`node2`，现在有 3000 万个`key`，希望可以将这些个 key 均匀的缓存到三台机器上，你会想到什么方案呢？

我们可能首先想到的方案，是取模算法`hash（key）% N`，对 key 进行 hash 运算后取模，N 是机器的数量。key 进行 hash 后的结果对 3 取模，得到的结果一定是 0、1 或者 2，正好对应服务器`node0`、`node1`、`node2`，存取数据直接找对应的服务器即可，简单粗暴，完全可以解决上述的问题。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibjzMNa43l9YBSPMyY4NmbgICAPia3ia4WUoLOPY9Ds5AWjzn8icoUyI9OQ/640?wx_fmt=png)

### hash 的问题

取模算法虽然使用简单，但对机器数量取模，在集群扩容和收缩时却有一定的局限性，因为在生产环境中根据业务量的大小，调整服务器数量是常有的事；而服务器数量 N 发生变化后`hash（key）% N`计算的结果也会随之变化。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTib25jddOdc0X9DbdYjYbNZsn3BHpicKaWoXojz2SYDueCzqJIic4rBv3OQ/640?wx_fmt=png)

比如：一个服务器节点挂了，计算公式从`hash（key）% 3`变成了`hash（key）% 2`，结果会发生变化，此时想要访问一个 key，这个 key 的缓存位置大概率会发生改变，那么之前缓存 key 的数据也会失去作用与意义。

大量缓存在同一时间失效，造成缓存的雪崩，进而导致整个缓存系统的不可用，这基本上是不能接受的，为了解决优化上述情况，一致性 hash 算法应运而生~

那么，一致性哈希算法又是如何解决上述问题的？

### 一致性 hash

一致性 hash 算法本质上也是一种取模算法，不过，不同于上边按服务器数量取模，一致性 hash 是对固定值 2^32 取模。

> “
>
> IPv4 的地址是 4 组 8 位 2 进制数组成，所以用 2^32 可以保证每个 IP 地址会有唯一的映射

**hash 环**

我们可以将这 2^32 个值抽象成一个圆环⭕️（**不得意圆的，自己想个形状，好理解就行**），圆环的正上方的点代表 0，顺时针排列，以此类推，1、2、3、4、5、6…… 直到 2^32-1，而这个由 2 的 32 次方个点组成的圆环统称为`hash 环`。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTib26ic4p4R6shTRmQhh9bfJZ36u6n6y961zia9rrOIB5UdVKCxmo7r8lVg/640?wx_fmt=png)

那么这个 hash 环和一致性 hash 算法又有什么关系嘞？我们还是以上边的场景为例，三台缓存服务器编号`node0`、`node1`、`node2`，3000 万个`key`。

**服务器映射到 hash 环**

这个时候计算公式就从**hash（key）% N** 变成了**hash（服务器 ip）% 2^32**，使用服务器 IP 地址进行 hash 计算，用哈希后的结果对 2^32 取模，结果一定是一个 0 到 2^32-1 之间的整数，而这个整数映射在 hash 环上的位置代表了一个服务器，依次将`node0`、`node1`、`node2`三个缓存服务器映射到 hash 环上。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibj8NPAHgl75AiaGDAfez1QNAyZbDZ8kibZbvxSSYeJPnAKc3MiaQ0UBOCg/640?wx_fmt=png)

**对象 key 映射到 hash 环**

接着在将需要缓存的 key 对象也映射到 hash 环上，**hash（key）% 2^32**，服务器节点和要缓存的 key 对象都映射到了 hash 环，那对象 key 具体应该缓存到哪个服务器上呢？

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibYFdrISHfofj51NeDX7WHo1wu7RWC4hibxULuD9wArzWtlqYO7R1UckQ/640?wx_fmt=png)

**对象 key 映射到服务器**

> “
>
> **从缓存对象 key 的位置开始，沿顺时针方向遇到的第一个服务器，便是当前对象将要缓存到的服务器**。

因为被缓存对象与服务器 hash 后的值是固定的，所以，在服务器不变的条件下，对象 key 必定会被缓存到固定的服务器上。根据上边的规则，下图中的映射关系：

-   `key-1 -> node-1`
-   `key-3 -> node-2`
-   `key-4 -> node-2`
-   `key-5 -> node-2`
-   `key-2 -> node-0`

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibficIE6TPFAHcAxxocyJj2sJ9CPiaVjvuKYAOONF0JgfddEtA9ucvF2og/640?wx_fmt=png)

如果想要访问某个 key，只要使用相同的计算方式，即可得知这个 key 被缓存在哪个服务器上了。

### 一致性 hash 的优势

我们简单了解了一致性 hash 的原理，那它又是如何优化集群中添加节点和缩减节点，普通取模算法导致的缓存服务，大面积不可用的问题呢？

先来看看扩容的场景，假如业务量激增，系统需要进行扩容增加一台服务器`node-4`，刚好`node-4`被映射到`node-1`和`node-2`之间，沿顺时针方向对象映射节点，发现原本缓存在`node-2`上的对象`key-4`、`key-5`被重新映射到了`node-4`上，而整个扩容过程中受影响的只有`node-4`和`node-1`节点之间的一小部分数据。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibZmeCicL4pF7dq7zryicykJFcWJriaibDbfrTu3xxxOr314pGCmu5jXGjmw/640?wx_fmt=png)

反之，假如`node-1`节点宕机，沿顺时针方向对象映射节点，缓存在`node-1`上的对象`key-1`被重新映射到了`node-4`上，此时受影响的数据只有`node-0`和`node-1`之间的一小部分数据。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTib8WkDMRCE5sYibGHKC9DWaMbaibQ5G01qdLnibg6eXIev5GKZxJVDUeQ9w/640?wx_fmt=png)

从上边的两种情况发现，当集群中服务器的数量发生改变时，一致性 hash 算只会影响少部分的数据，保证了缓存系统整体还可以对外提供服务的。

### 数据偏斜问题

前边为了便于理解原理，画图中的 node 节点都很理想化的相对均匀分布，但理想和实际的场景往往差别很大，就比如办了个健身年卡的我，只去过健身房两次，还只是洗了个澡。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibp2aNsemkDNqaq3N9Gp9VoPvH9b2hhe8ehbib0Olsm12YArHPdyKl1oQ/640?wx_fmt=png)

想要健身的你

在服务器节点数量太少的情况下，很容易因为节点分布不均匀而造成**数据倾斜**问题，如下图被缓存的对象大部分缓存在`node-4`服务器上，导致其他节点资源浪费，系统压力大部分集中在`node-4`节点上，这样的集群是非常不健康的。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibw1RPaLDhWMCNUafHJFmG6YzAInegU6T9icjfJRvfbQ0alIadluQfHWQ/640?wx_fmt=png)

解决数据倾斜的办法也简单，我们就要想办法让节点映射到 hash 环上时，相对分布均匀一点。

一致性 Hash 算法引入了一个**虚拟节点**机制，即对每个服务器节点计算出多个 hash 值，它们都会映射到 hash 环上，映射到这些虚拟节点的对象 key，最终会缓存在真实的节点上。

虚拟节点的 hash 计算通常可以采用，对应节点的 IP 地址加数字编号后缀 **hash（10.24.23.227#1)** 的方式，举个例子，node-1 节点 IP 为 10.24.23.227，正常计算`node-1`的 hash 值。

-   `hash（10.24.23.227#1）% 2^32`

假设我们给 node-1 设置三个虚拟节点，`node-1#1`、`node-1#2`、`node-1#3`，对它们进行 hash 后取模。

-   `hash（10.24.23.227#1）% 2^32`
-   `hash（10.24.23.227#2）% 2^32`
-   `hash（10.24.23.227#3）% 2^32`

下图加入虚拟节点后，原有节点在 hash 环上分布的就相对均匀了，其余节点压力得到了分摊。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibCP9OGPaW3hzlAp1aVicxVxCG1qQic62icYFECszpnicibvzicqxdtN0Nqiahw/640?wx_fmt=png)

> “
>
> 但需要注意一点，分配的虚拟节点个数越多，映射在 hash 环上才会越趋于均匀，节点太少的话很难看出效果

引入虚拟节点的同时也增加了新的问题，要做虚拟节点和真实节点间的映射，`对象 key-> 虚拟节点 -> 实际节点`之间的转换。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibSPoZATyibah7DW4lWCkopAQjvoyrUmATsCiauLfWvJN3hP12o9zNiacxQ/640?wx_fmt=png)

### 一致性 hash 的应用场景

一致性 hash 在分布式系统中应该是实现负载均衡的首选算法，它的实现比较灵活，既可以在客户端实现，也可以在中间件上实现，比如日常使用较多的缓存中间件`memcached`和`redis`集群都有用到它。

memcached 的集群比较特殊，严格来说它只能算是**伪集群**，因为它的服务器之间不能通信，请求的分发路由完全靠客户端来的计算出缓存对象应该落在哪个服务器上，而它的路由算法用的就是一致性 hash。

![](https://mmbiz.qpic.cn/mmbiz_png/0OzaL5uW2aO7kfG7JPxoFX9A3hju1lTibyrdZFhfIJSCoEMXOxmVScliatUwkt47usCqkr8icSZicQWgwdw9q4penQ/640?wx_fmt=png)

还有 redis 集群中 hash 槽的概念，虽然实现不尽相同，但思想万变不离其宗，看完本篇的一致性 hash，你再去理解 redis 槽位就轻松多了。

其它的应用场景还有很多：

-   `RPC`框架`Dubbo`用来选择服务提供者
-   分布式关系数据库分库分表：数据与节点的映射关系
-   `LVS`负载均衡调度器
-   .....................

### 总结

简单的阐述了下一致性 hash，如果有不对的地方大家可以留言指正，任何技术都不会十全十美，一致性 Hash 算法也是有一些潜在隐患的，如果 Hash 环上的节点数量非常庞大或者更新频繁时，检索性能会比较低下，而且整个分布式缓存需要一个路由服务来做负载均衡，一旦路由服务挂了，整个缓存也就不可用了，还要考虑做高可用。

不过话说回来，只要是能解决问题的都是好技术，有点副作用还是可以忍受的。

**我是小富～**，如果对你有用**在看**、**关注**支持下，咱们下期见~

![](https://mmbiz.qpic.cn/mmbiz_png/lYDNMgmFjur6mfkPywZNgKZYXia5C4lA8Kd6CUYOibay3HbIfxBnNXH3v6jQZf6QquSL5dt6qYMrtQGh7go67FZg/640?wx_fmt=png)

 往期推荐 

**🔗**

[**面试官问：订单 30 分钟未支付，自动取消，该怎么实现？**](http://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247497637&idx=1&sn=4bd9d411830e60aabe790b5acea71304&chksm=9b866758acf1ee4e785fea28a16e6def02aefb04e8c9a1a95ea55f1d855ab1b564a307c27977&scene=21#wechat_redirect)

[**11 张图总结下，微服务增量拉取**](http://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247498064&idx=1&sn=712a1e2aec80d7846fccb1731c876972&chksm=9b8669adacf1e0bb235828505aaab5d21f73d3ff7660a49128f57fdb146e40da7d64dbca07ce&scene=21#wechat_redirect)**[25 张图吃透「偏向锁」，这个 JVM 又爱又恨的崽](http://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247498652&idx=1&sn=f3c389b31356e164e488595b05797339&chksm=9b866b61acf1e2770656bb7713b9546b01023ac542f3c3caa04d4bedfeca2e81d7c2973e7ae9&scene=21#wechat_redirect)**

**[10 个解放双手的 IDEA 插件，这些代码真不用手写（第二弹）](http://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247494157&idx=1&sn=c6b614ecff050b524d0e7c27af06994c&chksm=9b867af0acf1f3e681316e7177f1bc9b36223c097dd83dfba6b7185cb87771bc23a588748112&scene=21#wechat_redirect)**

**实战干货！Spring Cloud Gateway 整合 OAuth2.0 实现分布式统一认证授权！**

**在看 \*\***、\***\* 点赞 \*\***、\***\* 转发 \*\***，是对我最大的鼓励 \*\*。

整理了几百本各类技术电子书，有需要的同学公众号内回复\[ **666** ]自取。技术群快满了，想进的同学可以加我好友，和大佬们一起吹吹技术。

                                           你的每个赞和在看，我都喜欢！

![](https://mmbiz.qpic.cn/mmbiz_gif/0OzaL5uW2aNhvrFicKVM7CoIQaorevsJuIo7hGvmZUSHCIaicBxIKDMlPqvyfprCa9svys6akhIv7ren0QJSqvaA/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/8JSqeoxuSX0zZu2w0swjKA](https://mp.weixin.qq.com/s/8JSqeoxuSX0zZu2w0swjKA)
