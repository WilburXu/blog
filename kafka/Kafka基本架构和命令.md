# Kafka基本架构和命令

## Kafka体系架构
<img src="images/1-1.png" alt="1-1" style="zoom:50%;" />

### Broker服务代理节点

服务代理节点。对于Kafka而言，Broker可以简单地看作一个独立的Kafka服务节点或Kafka服务实例。大多数情况下也可以将Broker看作一台Kafka服务器，前提是这台服务器上只部署了一个Kafka实例，一个或多个Broker组成了一个Kafka集群。

### Producer生产者

生产者，也就是发送消息的一方。生产者负责创建消息，然后将其投递到Kafka中。

<img src="images/1-2.png" alt="1-2" style="zoom:50%;" />

一个正常的生产逻辑需要具备以下几个步骤：

1. 创建生产者实例
2. 构建待发送的消息
3. 发送消息到指定的`Topic`、`Partition`、`Key`
4. 关闭生产者实例

### Consumer消费者

消费者，也就是接收消息的一方。消费者连接到Kafka上并接收消息，从而进行相应的业务逻辑处理。

消费一般有三种消费模式：

#### 单线程模式

#### 多线程模式

##### 独立消费者模式

##### 消费组模式



### Topic主题

Kafka中的消息以主题为单位进行归类（逻辑概念，生产者负责将消息发送到特定的主题（发送到Kafka集群中的每一条消息都要指定一个主题），而消费者负责订阅主题并进行消费。

### Partition分区

物理分区，主题细分为了1或多个分区，一个分区只能属于单个主题，一般也会把分区称为主题分区（Topic-Partition）。

### Segment

实际存储数据的地方，`Segment`包含一个数据文件和一个索引文件。一个`Partition`有多个大小相同的`Segment`，可以理解为`Partition`是在`Segment`之上进行的逻辑抽象。





## 查看所有Broker

### zookeeper

broker节点保存在zookeeper，所有需要：

1. 进入zookeeper，然后 `./bin/zkCli.sh`

2. 执行`ls /brokers/ids`

### 查看broker详情

`kafka-log-dirs.sh --describe --bootstrap-server kafka:9092 --broker-list 1`




## kafka 命令

### topic

#### 查看列表

`kafka-topics.sh --list --zookeeper zookeeper:2181`

#### 创建

` kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 3 --topic [topic_name]`

#### 查看详情

` kafka-topics.sh --describe --zookeeper zookeeper:2181 --topic [topic_name]`

#### 删除

`kafka-topics.sh --zookeeper zookeeper:2181 --delete --topic [topic_name]`



#### topic消费情况

##### topic offset 最小

`kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic [topic_name] --time -2`

##### topic offset最大

`kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic [topic_name] --time -1`



### 生产

##### 添加数据

`kafka-console-producer.sh --broker-list localhost:9092 --topic [topic_name]`

### 消费

##### 从头部开始消费

`kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [topic_name] --from-beginning`

##### 从尾部开始消费，必需要指定分区

`kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [topic_name] --offset latest --partition 0`

##### 从某个位置开始消费(--offset [n])

`kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [topic_name] --offset 100 --partition 0`

##### 消费指定个数(--max-messages [n])

`kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [topic_name] --offset latest --partition 0 --max-messages 2`



### 消费组

##### 查看消费组列表

`kafka-consumer-groups.sh  --list --bootstrap-server localhost:9092`

##### 查看消费组情况

`kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group [group_id]`

##### offset 偏移设置为最早

`kafka-consumer-groups.bat --bootstrap-server kafka:9092 --group kafka_consumer_session --reset-offsets --to-earliest --all-topics --execute`

##### offset 偏移设置为新

`kafka-consumer-groups.bat --bootstrap-server kafka:9092 --group kafka_consumer_session --reset-offsets --to-latest --all-topics --execute`

##### offset 偏移设置为指定位置

`kafka-consumer-groups.bat --bootstrap-server kafka:9092 --group kafka_consumer_session --reset-offsets --to-offset 2000 --all-topics --execute`

##### offset 偏移设置某个时间之后最早位移

`kafka-consumer-groups.bat --bootstrap-server kafka:9092 --group kafka_consumer_session --reset-offsets --to-datetime 2020-12-26T00:00:00.000 --all-topics --execute`



