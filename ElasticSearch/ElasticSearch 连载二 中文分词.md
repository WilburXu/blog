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

接着，重新启动 Elastic，就会自动加载这个新安装的插件。

然后，新建一个 Index，指定需要分词的字段。这一步根据数据结构而异，下面的命令只针对本文。基本上，凡是需要搜索的`中文字段`，都要单独设置一下。

