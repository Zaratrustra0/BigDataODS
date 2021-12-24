#  如何提高Elasticsearch搜索的相关性_犀牛饲养员-CSDN博客
## 什么是相关性

首先需要了解什么是相关性？默认情况下，搜索返回的结果是按照 相关性 进行排序的，也就是最相关的文档排在最前。相关性是由一个所谓的打分机制决定的，每个文档在搜索过程中都会被计算一个`_score`字段，这是一个浮点数类型，值越高表示分数越高，也就是相关性越大。

具体的评分算法不是本文的重点，但是我们可以通过一个查询示例了解下评分的过程。ES 对于一次搜索请求提供了一种`explain`的机制，设置为 true 的情况下，查询结果会额外输出一些信息，我们一起来看下这些信息。

查询的 demo 是，

```
GET kibana_sample_data_logs/_search
{
  "explain": true, 
  "size": 1, 
  "query": {
    "match": {
      "message": "metricbeat"
    }
  }
}

```

查询结果里包含了 `_explanation`字段 。其中包含了`description` 、 `value` 、 `details` 字段，它分别告诉你计算的类型、计算结果和计算细节。

```
"_explanation" : {
          "value" : 2.912974,
          "description" : "weight(message:metricbeat in 6) [PerFieldSimilarity], result of:",
          "details" : [
            {
              "value" : 2.912974,
              "description" : "score(freq=2.0), computed as boost * idf * tf from:",
              "details" : [
                {
                  "value" : 2.2,
                  "description" : "boost",
                  "details" : [ ]
                },
                {
                  "value" : 2.1402972,
                  "description" : "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",
                  "details" : [
                    {
                      "value" : 1655,
                      "description" : "n, number of documents containing term",
                      "details" : [ ]
                    },
                    {
                      "value" : 14074,
                      "description" : "N, total number of documents with field",
                      "details" : [ ]
                    }
                  ]
                },
                {
                  "value" : 0.6186426,
                  "description" : "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",
                  "details" : [
                    {
                      "value" : 2.0,
                      "description" : "freq, occurrences of term within document",
                      "details" : [ ]
                    },
                    {
                      "value" : 1.2,
                      "description" : "k1, term saturation parameter",
                      "details" : [ ]
                    },
                    {
                      "value" : 0.75,
                      "description" : "b, length normalization parameter",
                      "details" : [ ]
                    },
                    {
                      "value" : 28.0,
                      "description" : "dl, length of field",
                      "details" : [ ]
                    },
                    {
                      "value" : 27.013002,
                      "description" : "avgdl, average length of field",
                      "details" : [ ]
                    }
                  ]
                }
              ]
            }
          ]
        }

```

首先，前面两行，

```
"_explanation" : {
          "value" : 2.912974,
          "description" : "weight(message:metricbeat in 15) [PerFieldSimilarity], result of:",
          ......

```

告诉了我们 metricbeat 在 message 字段中的检索评分结果。15 是文档的内部 id，这个可以不用管。

紧接着是`details`字段，它是个嵌套的结构，里面可以包含多个`details`。

```
"details" : [
            {
              "value" : 2.912974,
              "description" : "score(freq=2.0), computed as boost * idf * tf from:",
              ......

```

这部分告诉我们，2.912974 这个值是有三部分相乘得到的：

```
boost * idf * tf

```

这三个值分别是 2.2，2.1402972，0.6186426，相乘的结果确实是 2.912974。

后面三个嵌套的 details，就是对应上面三部分，告诉你上面三部分是怎么计算的，比如 idf 部分：

```
{
                  "value" : 2.1402972,
                  "description" : "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",
                  "details" : [
                    {
                      "value" : 1655,
                      "description" : "n, number of documents containing term",
                      "details" : [ ]
                    },
                    {
                      "value" : 14074,
                      "description" : "N, total number of documents with field",
                      "details" : [ ]
                    }
                  ]
                },

```

这个是说 idf 这个值，是由

```
log(1 + (N - n + 0.5) / (n + 0.5))

```

这个公式计算出来的。其中 n 是 1655，N 是 14074，另外也告诉你这两个字母分别表示啥意思。其中 n 表示包含`metricbeat`这个词的文档数量。N 表示一共有多少文档（基于分片）。

## 提高搜索的相关性

我们通过一个示例来展开这部分的讨论。首先写入一些测试数据，

```
PUT demo_idx/_doc/1
{
  "content": "Distributed nature, simple REST APIs, speed, and scalability"
}
PUT demo_idx/_doc/2
{
  "content": "Distributed nature, simple APIs, speed, and scalability"
}
PUT demo_idx/_doc/3
{
  "content": "Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization."
}

```

先来一个基本的查询看看效果，

```
GET demo_idx/_search
{
  "query": {
    "match": {
      "content": {
        "query": "simple rest apis distributed nature"
      }
    }
  }
}

```

返回结果，

```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.2689934,
    "hits" : [
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.2689934,
        "_source" : {
          "content" : "Distributed nature, simple REST APIs, speed, and scalability"
        }
      },
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.6970792,
        "_source" : {
          "content" : "Distributed nature, simple APIs, speed, and scalability"
        }
      },
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.69611007,
        "_source" : {
          "content" : "Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization."
        }
      }
    ]
  }
}

```

可以看到是文档 1 评分最高，其次是文档 2 和文档 3。这个结果好不好呢？答案是不一定，具体要看你的业务场景，或者说在你的业务场景下你期望什么结果。

默认情况下，上面的查询 ES 会使用 OR 来分别查询每个 term，也就是说上面的查询会被解析为

```
simple OR rest OR apis OR distributed OR nature

```

然后查询的结果相加的分数就是整个查询的分数。文档 1 包含所有的查询 term，并且文档比较短（跟算法有关），所以它的分数最高。文档 2 也比较短，但是它少了一些 term。文档 3 包含了所有的查询 term，但是它太长了，导致算分贡献太少。

注意到文档 1 和文档 2 的 term 顺序和查询语句里不一样，但是这并不影响最后的算分，因为 OR 查询是不关心顺序的。

所以我上面说，这个结果究竟好不好，取决于你的业务场景。比如你的场景对顺序要求很严格，可能你期望文档 3 算分最高。再比如你对顺序没有要求，但是要求所有的查询 term 都必须存在，那么文档 2 就不能在返回结果里。下面就来使用示例来看看这些场景。

### 场景 1，要求查询 term 都存在

这种场景，需要使用 AND 操作符，如下：

```
GET demo_idx/_search
{
  "query": {
    "match": {
      "content": {
        "query": "simple rest apis distributed nature",
        "operator": "and"
      }
    }
  }
}

```

结果如下：

```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.2689934,
    "hits" : [
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.2689934,
        "_source" : {
          "content" : "Distributed nature, simple REST APIs, speed, and scalability"
        }
      },
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.69611007,
        "_source" : {
          "content" : "Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization."
        }
      }
    ]
  }
}


```

只有文档 1 和文档 3 返回了，符合预期。

### 场景 2，对 term 顺序有要求

这个场景下，希望文档里 term 出现的顺序和查询语句一样。ES 提供了`match phrase`查询可以满足这种场景。

```
GET demo_idx/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "simple rest apis distributed nature"
      }
    }
  }
}

```

结果如下：

```
{
  "took" : 26,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.6961101,
    "hits" : [
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.6961101,
        "_source" : {
          "content" : "Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization."
        }
      }
    ]
  }
}

```

### 场景 3，组合场景

比如我们期望 term 都存在，或者顺序相同的 term 查询，任意满足都可以，可以使用 bool 查询组合条件。

```
GET demo_idx/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "content": {
              "query": "simple rest apis distributed nature",
              "operator": "and"
            }
          }
        },
        {
          "match_phrase": {
            "content": {
              "query": "simple rest apis distributed nature"
            }
          }
        }
      ]
    }
  }
}

```

这个查询，should 包含两个查询条件，每个查询条件都会对文档贡献算分，并且默认情况下权重是一样的。这个查询的结果是，

```
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.3922203,
    "hits" : [
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.3922203,
        "_source" : {
          "content" : "Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization."
        }
      },
      {
        "_index" : "demo_idx",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.2689934,
        "_source" : {
          "content" : "Distributed nature, simple REST APIs, speed, and scalability"
        }
      }
    ]
  }
}


```

返回了文档 3 和文档 1，并且文档 3 的算分更高。文档 3 更高的原因在于它两个条件都满足，而文档 1 只满足第一个条件。

## 总结

ES 提供了多种查询方式，没有哪种是绝对最优的。在实际项目中，我们应该根据自己的业务场景选择合适的查询方式，才能获得最优的查询结果。

* * *

参考:

-   [https://www.elastic.co/cn/blog/how-to-improve-elasticsearch-search-relevance-with-boolean-queries](https://www.elastic.co/cn/blog/how-to-improve-elasticsearch-search-relevance-with-boolean-queries) 
    [https://blog.csdn.net/pony_maggie/article/details/114357736](https://blog.csdn.net/pony_maggie/article/details/114357736)
