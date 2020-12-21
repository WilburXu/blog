# kafka入门

Kafka 体系架构包括若干 Producer、若干 Broker、若干Consumer:

- Producer：生产者，也就是发送消息的一方。生产者负责创建消息，然后将其投递到Kafka中。
- Consumer：消费者，也就是接收消息的一方。消费者连接到Kafka上并接收消息，进而进行相应的业务逻辑处理。
- Broker：服务代理节点。对于Kafka而言，Broker可以简单地看作一个独立的Kafka服务节点或Kafka服务实例。大多数情况下也可以将Broker看作一台Kafka服务器，前提是这台服务器上只部署了一个Kafka实例。一个或多个Broker组成了一个Kafka集群。一般而言，我们更习惯使用首字母小写的broker来表示服务代理节点。

在Kafka中还有两个特别重要的概念—主题（Topic）与分区（Partition）。Kafka中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题（发送到Kafka集群中的每一条消息都要指定一个主题），而消费者负责订阅主题并进行消费。



## 客户端

一个正常的生产逻辑需要具备以下几个步骤：

1. 配置生产者客户端参数及创建相应的生产者实力
2. 构建待发送的消息
3. 发送消息
4. 关闭生产者实例

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

kafka-run-class.sh kafka.tools.ConsumerOffsetChecker

##### topic offset 最小

`kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic [topic] --time -2`

##### topic offset最大

`kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic [topic] --time -1`



### 生产

##### 添加数据

`kafka-console-producer.sh --broker-list localhost:9092 --topic [topic]`

### 消费

`kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [topic] --from-beginning`

