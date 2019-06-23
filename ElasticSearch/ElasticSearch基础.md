# ElasticSearch基础

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

如果此时报以下错误

```bash
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x0000000085330000, 2060255232, 0) failed; error='Cannot allocate memory' (errno=12)
```

