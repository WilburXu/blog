# ElasticSearch 连载二 中文分词

上一章ElasticSearch 连载一 基础入门 对`Elastic`的概念、安装以及基础操作进行了介绍。

那是不是有童鞋会有以下几个问题呢？

1. 什么是中文分词器？
2. 分词器怎么安装？
3. 如何使用中文分词器？

那么接下来就为大家细细道来。

## 什么是中文分词器

搜索引擎的核心是 [倒排索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html) 而倒排索引的基础就是`分词`。所谓分词可以简单理解为将一个完整的句子切割为一个个单词的过程。在 es 中单词对应英文为 `term`。我们简单看下面例子：

```
我爱北京天安门
```

ES 的倒排索引即是根据分词后的单词创建，即 `我`、`爱`、`北京`、`天安门`这4个单词。这也意味着你在搜索的时候也只能搜索这4个单词才能命中该文档。



## 分词器安装

首先，安装中文分词插件。这里使用的是 [ik](https://github.com/medcl/elasticsearch-analysis-ik/) 。

```shell
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
```

上面代码安装的是5.5.1版的插件，与 Elastic 5.5.1 配合使用。

安装结束后，会发现目录 `/elasticsearch-5.5.1/plugins` 多了一个`analysis-ik` 的文件。

接着，重新启动 `Elastic` ，就会自动加载这个新安装的插件。

## 最简单的测试

用下面命令测试一下`ik`分词器：

```shell
curl -X GET 'http://localhost:9200/_analyze?pretty&analyzer=ik_smart' -d '我爱北京天安门'
```

返回结果：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "爱",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "北京",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "天安门",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 3
    }
  ]
}
```

那么恭喜你，完成了`ik`分词器的安装。



## 如何使用中文分词器

### 概念

这里介绍下 [什么是_all字段](http://blog.csdn.net/jiao_fuyou/article/details/49800969), 其实_all字段是为了在不知道搜索哪个字段时，使用的。`ES`会把所有的字段（除非你手动设置成false），都放在_all中，然后通过分词器去解析。当你使用query_string的时候，默认就在这个_all字段上去做查询，而不需要挨个字段遍历，节省了时间。

`properties`中定义了特定字段的分析方式

- type，字段的类型为string，只有string类型才涉及到分词，像是数字之类的是不需要分词的。
- store，定义字段的存储方式，no代表不单独存储，查询的时候会从_source中解析。当你频繁的针对某个字段查询时，可以考虑设置成true。
- term_vector，定义了词的存储方式，with_position_offsets，意思是存储词语的偏移位置，在结果高亮的时候有用。
- analyzer，定义了索引时的分词方法
- search_analyzer，定义了搜索时的分词方法
- include_in_all，定义了是否包含在_all字段中
- boost，是跟计算分值相关的。



### 添加Index

然后，新建一个 Index，指定需要分词的字段。这一步根据数据结构而异，下面的命令只针对本文。基本上，凡是需要搜索的`中文字段`，都要单独设置一下。

```json
curl -X PUT 'localhost:9200/school' -d '
{
  "mappings": {
    "student": {
        "_all": {
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_max_word",
            "term_vector": "no",
            "store": "false"
        },
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word",
          "include_in_all": "true",
          "boost": 8
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word",
          "include_in_all": "true",
          "boost": 10
        }
      }
    }
  }
}'
```

上面代码中，首先新建一个名称为`school`的 Index，里面有一个名称为`student`的 Type。`student`有三个字段。

>   - user
>   - desc

这两个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

上面代码中，`analyzer`是字段文本的分词器，`search_analyzer`是搜索词的分词器。`ik_max_word`分词器是插件`ik`提供的，可以对文本进行最大数量的分词。



## 数据操作

创建好了`Index`后，我们来实际演示下：

### 新增记录

```json
curl -X PUT 'localhost:9200/school/student/1' -d '
{
  "user": "许星星",
  "desc": "这是一个不可描述的姓名"
}'
```

```json
curl -X PUT 'localhost:9200/school/student/2' -d '
{
  "user": "天上的星星",
  "desc": "一闪一闪亮晶晶，爸比会跳舞"
}'
```

```json
curl -X PUT 'localhost:9200/school/student/3' -d '
{
  "user": "比克大魔王",
  "desc": "拿着水晶棒，亮晶晶的棒棒。"
}'
```

返回数据：

```json
{
    "_index": "school",
    "_type": "student",
    "_id": "3",
    "_version": 2,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": false
}
```



### 全文搜索

Elastic 的查询非常特别，使用自己的[查询语法](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl.html)，要求 GET 请求带有数据体。

```json
curl 'localhost:9200/school/student/_search'  -d '
{
  "query" : { "match" : { "desc" : "晶晶" }}
}'
```

上面代码使用 [Match 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-match-query.html)，指定的匹配条件是`desc`字段里面包含"晶晶"这个词。返回结果如下。

```json
{
    "took": 7,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 2.5811603,
        "hits": [
            {
                "_index": "school",
                "_type": "student",
                "_id": "3",
                "_score": 2.5811603,
                "_source": {
                    "user": "比克大魔王",
                    "desc": "拿着水晶棒，亮晶晶的棒棒。"
                }
            },
            {
                "_index": "school",
                "_type": "student",
                "_id": "2",
                "_score": 2.5316024,
                "_source": {
                    "user": "天上的星星",
                    "desc": "一闪一闪亮晶晶，爸比会跳舞"
                }
            }
        ]
    }
}
```

Elastic 默认一次返回10条结果，可以通过`size`字段改变这个设置。

```json
curl 'localhost:9200/school/student/_search'  -d '
{
  "query" : { "match" : { "desc" : "晶晶" }},
  "size" : 1
}'
```

上面代码指定，每次只返回一条结果。

还可以通过`from`字段，指定位移

```json
curl 'localhost:9200/school/student/_search'  -d '
{
  "query" : { "match" : { "desc" : "晶晶" }},
  "size" : 1,
  "from" : 1
}'
```

上面代码指定，从位置1开始（默认是从位置0开始），只返回一条结果。



### 逻辑运算

如果有多个搜索关键字， Elastic 认为它们是`or`关系。

```json
curl 'localhost:9200/school/student/_search'  -d '
{
  "query" : { "match" : { "desc" : "水晶棒 这是" }}
}'
```

返回结果：

```json
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 5.1623206,
        "hits": [
            {
                "_index": "school",
                "_type": "student",
                "_id": "3",
                "_score": 5.1623206,
                "_source": {
                    "user": "比克大魔王",
                    "desc": "拿着水晶棒，亮晶晶的棒棒。"
                }
            },
            {
                "_index": "school",
                "_type": "student",
                "_id": "1",
                "_score": 2.5811603,
                "_source": {
                    "user": "许星星",
                    "desc": "这是一个不可描述的姓名"
                }
            }
        ]
    }
}
```



如果要执行多个关键词的`and`搜索，必须使用[布尔查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-bool-query.html)。

```json
curl 'localhost:9200/school/student/_search'  -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "desc": "水晶棒" } },
        { "match": { "desc": "亮晶晶" } }
      ]
    }
  }
}'
```

返回结果：

```json
{
    "took": 24,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 10.324641,
        "hits": [
            {
                "_index": "school",
                "_type": "student",
                "_id": "3",
                "_score": 10.324641,
                "_source": {
                    "user": "比克大魔王",
                    "desc": "拿着水晶棒，亮晶晶的棒棒。"
                }
            }
        ]
    }
}
```



## 总结

本章介绍了分词器的基本概念和使用，至此`Elastic`算是有一个基本的入门，下一章节将进一步学习分词器的特性以及场景案例。