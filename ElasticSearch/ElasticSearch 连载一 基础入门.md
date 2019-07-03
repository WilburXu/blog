# ElasticSearch基础入门

ElasticSearch简写ES，ES是一个高扩展、开源的全文检索和分析引擎，它可以准实时地快速存储、搜索、分析海量的数据。

**应用场景**

- 我们常见的商城商品的搜索
- 日志分析系统（ELK）
- 基于大量数据（数千万的数据）需要快速调查、分析并且并将结果可视化的业务需求

## 安装并运行ES

### Java环境安装

Elastic 需要 Java 8 环境。如果你的机器还没安装 `Java`，可以参考[JAVA安装](https://www.cnblogs.com/benjamin77/p/8460030.html)

###ElasticSearch安装

安装完Java环境后，我们可以开始以下`ElasticSearch`安装或者根据[官方文档](https://www.elastic.co/guide/cn/elasticsearch/guide/current/running-elasticsearch.html)安装

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.zip
unzip elasticsearch-5.5.1.zip
cd elasticsearch-5.5.1/
```

进入解压目录之后，运行下面命令，启动`ElasticSearch`

`./bin/elasticsearch`

#### 如果此时报以下错误

##### 错误一

```bash
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
```

打开: `elasticsearch-5.5.1/config/jvm.options`

在末尾添加: 
```bash
-XX:-AssumeMP
```

##### 错误二

```bash
OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x0000000085330000, 2060255232, 0) failed; error='Cannot allocate memory' (errno=12)
```

先执行：

```shell
sysctl -w vm.max_map_count=262144
```



再打开`elasticsearch-5.5.1/config/jvm.options`

```bash
-Xmx512m
-Xms512m
```

##### 错误三

``` bash
[2019-06-27T15:01:43,165][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
```

原因：elasticsearch自5版本之后，处于安全考虑，不允许使用root用户运行。

解决：创建一个普通用户，将elasticsearch 安装目录权限修改一下，切换至普通用户运行elasticsearch就可以了

```shell
useradd elk
chown -R elk.elk /usr/local/share/applications/elasticsearch-5.5.1
su - elk
cd /usr/local/share/applications/elasticsearch-5.5.1
```

#### 重新启动

```shell
./bin/elasticsearch
```

如果一切正常，`Elastic ` 就会在默认的`9200`端口运行。这时，打开另一个命令行窗口，请求该端口，会得到说明信息。

```shell
$ curl 'localhost:9200'
```

```json
{
  "name" : "cWyaT72",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "A7akNm1SRw2Gm-BdSBkdaw",
  "version" : {
    "number" : "5.5.1",
    "build_hash" : "19c13d0",
    "build_date" : "2017-07-18T20:44:24.823Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

#### 访问配置

 `Elastic`  默认情况下，只允许本地访问，如果需要远程访问，可以修改 `config/elasticsearch.yml`文件，去掉`network.host`的注释，将它的值改成`0.0.0.0`，然后重新启动 Elastic。

```shell
network.host: 0.0.0.0
```

上面代码中，设成`0.0.0.0`让任何人都可以访问。线上服务不要这样设置，要设成具体的 IP。



## 基本概念

### Node 与 Cluster

`Elastic`本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 `Elastic` 实例。

单个 `Elastic` 实例称为一个节点`（node）`。一组节点构成一个集群`（cluster）`。

#### 查看Cluster Health

```shell
curl -X GET 'http://localhost:9200/_cat/health?v'
```

#### 获取集群的所有节点

```shell
curl -X GET 'http://localhost:9200/_cat/nodes?v'
```

### Index

`Elastic`会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。(一个 `Index` 类似于传统关系数据库中的一个 `数据库` ，是一个存储关系型文档的地方）。 

所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。

下面的命令可以查看当前节点的所有 Index。

```bash
curl -X GET 'http://localhost:9200/_cat/indices?v'
```

### Document

Index里的单条记录称为`Document`，多条`Document`构成一个`Index`.

`Document`使用JSON格式表示，如：

```json
{
	"goods_name": "空调",
	"category_name": "家电分类",
	"price": "3999.00"
}
```

同一个 Index 里面的 `Document`，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

### Type

`Document`是可以分组的，如`goods_list`这个`Index` ，可以按照`category（家电、衣服）`分类，也可以按照`price（>1000、 <1000）`分类。这种分组叫`Type`它是虚拟的逻辑分组，用于过滤`Document`。

列出每个`Index`下面的`Type`

```bash
curl 'http://localhost:9200/_mapping?pretty=true'
```

根据[规划](https://www.elastic.co/blog/index-type-parent-child-join-now-future-in-elasticsearch)，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。



## Index操作

### 新建（Create Index）

新建 `Index`，可以直接向 `Elastic`服务器发出 PUT 请求。下面的例子是新建一个名叫`goods_list`的 `Index`。

```shell
curl -X PUT 'http://localhost:9200/goods_list'
```

服务器返回一个 JSON 对象，里面的`acknowledged`字段表示操作成功。

```json
{
    "acknowledged": true,
    "shards_acknowledged": true
}
```

### 删除（Delete Index）

```bash
curl -X DELETE 'http://localhost:9200/goods_list'
```

```json
{
    "acknowledged": true
}
```



## 数据操作

上面介绍了`Index`和`Type`的一些基本的概念和`Index`的基本操作，现在先来创建一个完整的`Index`结构，并对数据进行操作。

### 新建Index结构

```shell
curl -X PUT 'localhost:9200/goods_list' -d '
{
    "mappings": {
        "goods_info": {
            "properties": {
                "goods_name": {
                    "type": "keyword"
                },
                "category_name": {
                    "type": "keyword"
                },
                "price": {
                    "type": "float"
                }
            }
        }
    }
}
'

{
    "acknowledged": true
}
```

执行上面命名，重新创建一个新的`Index`

### 新增记录

向指定的 `/Index/Type` 发送 `PUT` 请求，就可以在 `Index` 里面新增一条记录。比如，向`/goods_list/goods_info`发送请求，就可以新增一条商品记录。

```shell
curl -X PUT 'localhost:9200/goods_list/goods_info/1' -d '
{
  "goods_name": "华为笔记本",
  "category_name": "计算机",
  "price": "1000"
}' 
```

服务器返回的 JSON 对象，会给出 Index、Type、Id、Version 等信息：

```json
{
    "_index": "goods_list",
    "_type": "goods_info",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```

相信细心的你会发现`/goods_list/goods_info/1`，后面多了一个`1`，这个`1`是该条记录的 ID。可以是任意字符串

新增记录的时候，也可以不指定 Id，这时要改成 POST 请求。

```shell
curl -X POST 'localhost:9200/goods_list/goods_info' -d '
{
  "goods_name": "洗衣机",
  "category_name": "家电",
  "price": "899.99"
}'
```

如果没有指定`ID`，那么`Elastic`会随机生成一串字符串作为`ID`

```json
{
    "_index": "goods_list",
    "_type": "goods_info",
    "_id": "AWub5f7FFq1D5epJJhqT",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```

### 查看记录

```shell
curl 'localhost:9200/goods_list/goods_info/1?pretty=true'
```

上面代码请求查看`/goods_list/goods_info/1`这条记录，URL 的参数`pretty=true`表示以易读的格式返回。

返回的数据中，`found`字段表示查询成功，`_source`字段返回原始记录：

```json
{
  "_index" : "goods_list",
  "_type" : "goods_info",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "goods_name" : "华为笔记本",
    "category_name" : "计算机",
    "price" : "1000"
  }
}
```

如果 `ID`不正确，就查不到数据，`found`字段就是`false`。

```shell
curl 'localhost:9200/goods_list/goods_info/2?pretty=true'
```

`ID=2`并不存在，所以会返回以下结果：

```json
{
  "_index" : "goods_list",
  "_type" : "goods_info",
  "_id" : "2",
  "found" : false
}
```

### 删除记录

```shell
curl -X DELETE 'localhost:9200/goods_list/goods_info/1'
```

PS：这里先不要删除这条记录，后面还要用到。



### 更新记录

```shell
curl -X PUT 'localhost:9200/goods_list/goods_info/1' -d '
{
    "user" : "华为笔记本",
    "title" : "计算机",
    "desc" : "5000"
}'
```

更新记录就是使用 PUT 请求，重新发送一次数据。

```json
{
    "_index": "goods_list",
    "_type": "goods_info",
    "_id": "1",
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

返回结果里面，有几个字段发生了变化：

```json
"_version" : 2,
"result" : "updated",
"created" : false
```



## 数据查询

### 返回所有记录

```shell
curl 'localhost:9200/goods_list/goods_info/_search'
```



```json
{
    "took": 127,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 1,
        "hits": [
            {
                "_index": "goods_list",
                "_type": "goods_info",
                "_id": "AWub5f7FFq1D5epJJhqT",
                "_score": 1,
                "_source": {
                    "goods_name": "洗衣机",
                    "category_name": "家电",
                    "price": "899.99"
                }
            },
            {
                "_index": "goods_list",
                "_type": "goods_info",
                "_id": "1",
                "_score": 1,
                "_source": {
                    "user": "华为笔记本",
                    "title": "计算机",
                    "desc": "5000"
                }
            }
        ]
    }
}
```

上面代码中，返回结果的 `took`字段表示该操作的耗时（单位为毫秒），`timed_out`字段表示是否超时，`hits`字段表示命中的记录，里面子字段的含义如下：

> - `total`：返回记录数，本例是2条。
> - `max_score`：最高的匹配程度，本例是`1.0`。
> - `hits`：返回的记录组成的数组。

返回的记录中，每条记录都有一个`_score`字段，表示匹配的程序，默认是按照这个字段降序排列。



## 总结

这里主要介绍了`Elastic`的安装、基本概念以及数据的基本操作，在下一章带来`Elastic`的分词和全文搜索以及相关的技术点。

