# (8条消息) 一文带你彻底搞懂Elasticsearch中的模糊查询_犀牛饲养员-CSDN博客_elasticsearch 模糊查询
### 写在前面

Elasticsearch（以下简称 ES）中的模糊查询官方是建议慎用的，因为的它的性能不是特别好。不过这个性能不好是相对 ES 自身的其它查询（term，match）而言的，如果跟其它的搜索工具相比 ES 的模糊查询性能还是不错的。

ES 都多种方法可以支持模糊查询，比如 wildcard，query_string 等，这篇文章可能是全网最全的关于模糊查询的技术博客（哈哈）。

### 可以支持模糊查询的方案

#### wildcard

wildcard 的用法是这样的，

```
GET kibana_sample_data_flights/_search
{
  "query": {
    "wildcard": {
      "OriginCityName": {
        "value": "Frankfurt*"
      }
    }
  }
}

```

\* 或者? 也可以放在前面，但是不建议这么做，最好是前缀开始避免太大的性能消耗。查询的字段可以是 text 类型也可以是 keyword 类型，两种都支持。

大小写的话默认情况下，是根据字段本身是否对大小写敏感决定的。什么意思呢？比如上面那个查询，OriginCityName 字段是 keyword 类型，我们知道 keyword 是要求精确匹配，自然就是大小写敏感的。所以如果用下面的这个查询，结果是不一样的：

```
GET kibana_sample_data_flights/_search
{
  "query": {
    "wildcard": {
      "OriginCityName": {
        "value": "frankfurt*"
      }
    }
  }
}

```

如果查询的字段是 text 类型，wildcard 模糊查询的时候就是大小写不敏感的。

前面说过，模糊查询的性能都不高，wildcard 也不例外。不过在 ES7.9 中引入了一种新的`wildcard` 字段类型，该字段类型经过优化，可在字符串值中快速查找模式。

```
PUT my-index
{
  "mappings": {
    "properties": {
      "my_wildcard": {
        "type": "wildcard"
      }
    }
  }
}

```

在引入这个字段类型之前，wildcard 要么是在 text 类型字段查找，要么是 keyword 类型。而`wildcard`类型做了特殊的处理，如果某个字段指定了 wildcard 类型，

-   与 text 字段不同，它不会将字符串视为由标点符号分隔的单词的集合。
-   与 keyword 字段不同，它可以快速地搜索许多唯一值，并且没有大小限制。

wildcard 字段类型通过两种优化的数据结构提高模糊查询的性能，一种使用`n-gram`分词器，这个分词器不打算在这里详细讲，只需要知道它会把单词在继续细分存储就行，比如，

```
POST _analyze
{
  "tokenizer": "ngram",
  "text": "Quick Fox"
}

```

输出的是，

```
[ Q, Qu, u, ui, i, ic, c, ck, k, "k ", " ", " F", F, Fo, o, ox, x ]

```

相当于把可能用于模糊查询的词项都提前拆分好存储了，这样就减少了查询阶段需要比较的词项。

第二种数据结构是`binary doc value`，可以自动查询验证由 n-gram 语法匹配产生的匹配候选，关于它的具体介绍可以参考下面这篇文章：

[https://www.amazingkoala.com.cn/Lucene/DocValues/2019/0412/49.html](https://www.amazingkoala.com.cn/Lucene/DocValues/2019/0412/49.html)

#### fuzzy

fuzzy 也是一种模糊查询，我理解它其实属于比较轻量级别的模糊查询。fuzzy 中有个编辑距离的概念，编辑距离是对两个字符串差异长度的量化，及一个字符至少需要处理多少次才能变成另一个字符，比如 lucene 和 lucece 只差了一个字符他们的编辑距离是 1。

因为可以限制编辑距离，它的性能相对会好一些，毕竟它不是完全的 “模糊”。

这样说可能有点抽象，看个例子，

先写入一些测试数据，

```
POST /my_index/_bulk
{ "index": { "_id": 1 }}
{ "text": "Surprise me!"}
{ "index": { "_id": 2 }}
{ "text": "That was surprising."}
{ "index": { "_id": 3 }}
{ "text": "I wasn't surprised."}

```

然后我们可以这样查询，

```
GET /my_index/_search
{
  "query": {
    "fuzzy": {
      "text": "surprize"
    }
  }
}

```

查询结果是文档 1 和文档 3 会被查询出来，surprise 比较 surprise 和 surprised 都在编辑距离 2 以内。为什么默认值 2 呢，其实 fuzzy 有个`fuzziness`参数，可以赋值为 0，1，2 和 AUTO，默认其实是 AUTO。

AUTO 的意思是，根据查询的字符串长度决定允许的编辑距离，规则是：

-   0…2 完全匹配（就是不允许模糊）
-   3…5 编辑距离是 1
-   大于 5 编辑距离是 2

其实我们仔细想一下，即使限制了编辑距离，查询的字符串比较长的情况下需要查询的词项也是非常巨大的。所以 fuzzy 还有一个选项是 prefix_length，表示不能被 “模糊化” 的初始字符数，通过限制前缀的字符数量可以显著降低匹配的词项数量。

#### query string

query string query 是 ES 的一种高级搜索，它支持复杂的搜索方式比如操作符，可以用类似

```
"query": "this AND that"

```

这样的组合操作语法。

query string 支持 wildcard，并且查询的字段名和查询字符串都可以使用 wildcard，比如：

```
GET /_search
{
  "query": {
    "query_string" : {
      "fields" : ["city.*"],
      "query" : "this AND that OR thus"
    }
  }
}'

```

```
GET /_search
{
  "query": {
    "query_string" : {
      "query" : "city.\\*:(this AND that OR thus)"
    }
  }
}

```

所以 query string 对模糊搜索的支持本质上还是 wildcard。

#### prefix 前缀查询

这种只支持前缀查询，属于模糊查询的子集。比如要查找所有以 W1 开始的邮编，可以使用简单的 prefix 查询。

```
GET /my_index/_search
{
    "query": {
        "prefix": {
            "postcode": "W1"
        }
    }
}

```

prefix 的工作原理这里也简单说下。我们知道文档在写入 ES 时会建立倒排索引，倒排索引都会将包含词的文档 ID 列入 倒排表（postings list），下面是一个示例：

| Term       | Doc IDs |
| ---------- | ------- |
| “SW5 0BE”  | 5       |
| “W1F 7HW”  | 3       |
| “W1V 3DG”  | 1       |
| “W2F 8HW”  | 2       |
| “WC1N 1LZ” | 4       |

查询的步骤是：

1.  扫描 postings list 并查找到第一个以 W1 开始的词。
2.  搜集关联的文档 ID 。
3.  移动到下一个词。如果这个词也是以 W1 开头，查询跳回到第二步再重复执行，直到下一个词不以 W1 为止。

可以看到，如果倒排表比较大，满足前缀的词项比较多的情况下，查询的代价也是非常大的。不过对于前缀查询 ES 提供了一种名叫`index_prefixes`的机制来提高查询性能。

原理也比较简单，就是字段在 mapping 中指定`index_prefixes`，然后 ES 在索引的时候就会把指定范围的前缀都先存起来，这样查询的时候需要比较的次数就会大大降低。

```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "body_text": {
        "type": "text",
        "index_prefixes": { }    
      }
    }
  }
}

```

#### regexp 正则表达式模糊查询

regexp 对模糊查询的支持更智能，它能支持更为复杂的匹配模式。比如下面这个示例

```
GET /my_index/_search
{
    "query": {
        "regexp": {
            "postcode": "W[0-9].+" 
        }
    }
}

```

这个正则表达式要求词必须以 W 开头，紧跟 0 至 9 之间的任何一个数字，然后接一或多个其他字符。

regexp 查询的工作方式与 prefix 查询基本是一样的，需要扫描倒排索引中的词列表才能找到所有匹配的词，然后依次获取每个词相关的文档 ID。

* * *

参考:

-   [https://www.elastic.co/guide/en/elasticsearch/reference/7.11/index.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/index.html)
-   [https://www.elastic.co/cn/blog/find-strings-within-strings-faster-with-the-new-elasticsearch-wildcard-field](https://www.elastic.co/cn/blog/find-strings-within-strings-faster-with-the-new-elasticsearch-wildcard-field) 
    [https://blog.csdn.net/pony_maggie/article/details/113951893](https://blog.csdn.net/pony_maggie/article/details/113951893)
