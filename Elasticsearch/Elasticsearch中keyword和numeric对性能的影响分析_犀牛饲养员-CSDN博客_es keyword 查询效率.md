# Elasticsearch中keyword和numeric对性能的影响分析_犀牛饲养员-CSDN博客_es keyword 查询效率
初学者认为这两个关键字的没啥关系，一个是用于字符串的精确匹配查询，一个是数字类型的字段用在计数的场景，比如说博客的点赞数，订单金额等。

但是有些场景似乎两个关键字都可以用，比如电商场景下的订单状态，一般我们也是用数字表示不同的状态，比如 1 表示待支付，2 表示支付成功。第一反应是用 Byte（属于 numeric），没有问题。但是用 keyword 是否可以呢？

numeric 除了支持等值精确查询，还可以范围查询。但是大部分情况下我们业务场景对于订单状态的使用都是精确查询的，不会有大于某个状态或者小于某个状态这样的情况。

所以刚才说的订单状态的场景，用 keyword 和 numeric 肯定都可以满足。但是那种方案好呢？答案是 keyword。

对于 keyword 类型的 term query，ES 使用的是倒排索引。但是 numeric 类型为了能有效的支持范围查询，它的存储结构并不是倒排索引。

我们知道倒排索引在内存里维护了词典 (Term Dictionary)和文档列表 (Postings List) 的映射关系，倒排索引本身对于精确匹配查询是非常快的，直接从字典表找到 term，然后就直接找到了 posting list。

numeric 类型从 lucene6.0 开始，使用了一种名为`block KD tree`的存储结构。

## Block KD tree 介绍

kd-tree（k-dimensional 树的简称），是一种对 k 维空间中的实例点进行存储以便对其进行快速检索的树形数据结构。  
这种存储结构类似于 mysql 里的 B + 数，我们知道 B + 数这种数据结构对于范围查找的支持是非常好的。不同于 mysql， Block KD tree 的叶子节点存储的是一组值的集合 (block)，大概是 512~1024 个值一组。这也是为什么叫 block kd tree。

Block KD tree 对于范围查询，邻近搜索支持的非常好，尤其是多维空间的情况下。

![](https://img-blog.csdnimg.cn/20210118221613853.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvbnlfbWFnZ2ll,size_16,color_FFFFFF,t_70#pic_center)

看上图，每个节点有三个元素，所以这里 K=3，不同于简单二叉树每个节点都是一个元素（如下面这个图）。这样就可以方便的在一个三维的空间进行范围的比较。

![](https://img-blog.csdnimg.cn/20210118221623875.png#pic_center)

标准的二叉树

对于上图中的 kd-tree，搜索的过程是这样的：首先和根节点比较第一项，小于往左，大于往右，第二层比较第二项，依次类推。每层参与比较的数据是不一样的。

具体的 ES 内部（其实是 Lucene），目前的版本是基于所谓的`PointValues`，比如整型在 Lucene 内部是`IntPoint`类表示，还有`DoublePoint`等，完整的对应关系是：

```
Java type  Lucene class
int         IntPoint
long        LongPoint
float       FloatPoint
double      DoublePoint
byte[]      BinaryPoint

```

而这些 PointValues 是基于 kd-tree 存储的，根据官方文档的介绍，lucene 把叶子节点在磁盘是顺序存储的，这样搜索的效率就会非常高。

## 为啥 numeric 对于 term 精确匹配的查询性能没有 keyword 好

前面我们提到了`IntPoint`类，这个类有三个查询方法：

```
//构造精确查询，内部还是调用newRangeQuery
Query newExactQuery(String field, int value) 

```

```
//构造一维查询，内部是调用多维查询的方法
Query newRangeQuery(String field, int lowerValue, int upperValue)

```

```
//构造多维查询
Query newRangeQuery(String field, int[] lowerValue, int[] upperValue)

```

IntPoint.java  
![](https://img-blog.csdnimg.cn/20210118221638791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvbnlfbWFnZ2ll,size_16,color_FFFFFF,t_70#pic_center)

比如我们有这样一个索引：

```
PUT blogs 
{
  "mappings": {
    "properties": { 
      "title":    { "type": "keyword"  }, 
      "content":    { "type": "text"  }, 
      "status":     { "type": "integer"  }
    }
  }
}

```

如果我们基于 status 查询，

```
{
  "query": {
    "term": {
      "title": {
        "status": 2
      }
    }
  }
}

```

在 lucene 内部其实还是进行了一个 2~2 的范围查询。即便 kd-tree 的性能也很高，但是对于这种精确查询还是要到树上走一遭，而倒排索引相当于是直接在内存里就定位到了结果集的文档 id。如果是 bool 组合查询的话，term 还可以利用跳表，这点 numeric 字段也是做不到的。

引用:

-   [https://www.elastic.co/cn/blog/searching-numb3rs-in-5.0](https://www.elastic.co/cn/blog/searching-numb3rs-in-5.0) 
    [https://blog.csdn.net/pony_maggie/article/details/112796329](https://blog.csdn.net/pony_maggie/article/details/112796329)
