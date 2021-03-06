---
title: kafka知识整理
date: 2019-09-05 12:53:17
categories: 
- MQ
tags:
    - go,kafka
---
在每个公司的应用架构中，消息队列处理和消息发布订阅是一个核心关键的地方。在传统的应用中rabbitmq是老牌子的了，这里趁有时间整理一下kafka的知识。
<!-- more -->

## 应用领域

- 构建实时的流数据管道，可靠地获取系统和应用程序之间的数据。

- 构建实时流的应用程序，对数据流进行转换或反应。

## 应用场景

- 作为一个消息系统

Kafka的流与传统企业消息系统相比的概念如何？
传统的消息有两种模式：队列和发布订阅。 在队列模式中，消费者池从服务器读取消息（每个消息只被其中一个读取）; 发布订阅模式：消息广播给所有的消费者。这两种模式都有优缺点，队列的优点是允许多个消费者瓜分处理数据，这样可以扩展处理。但是，队列不像多个订阅者，一旦消息者进程读取后故障了，那么消息就丢了。而发布和订阅允许你广播数据到多个消费者，由于每个订阅者都订阅了消息，所以没办法缩放处理。

kafka中消费者组有两个概念：队列：消费者组（consumer group）允许同名的消费者组成员瓜分处理。发布订阅：允许你广播消息给多个消费者组（不同名）。

kafka的每个topic都具有这两种模式。

kafka有比传统的消息系统更强的顺序保证。
传统的消息系统按顺序保存数据，如果多个消费者从队列消费，则服务器按存储的顺序发送消息，但是，尽管服务器按顺序发送，消息异步传递到消费者，因此消息可能乱序到达消费者。这意味着消息存在并行消费的情况，顺序就无法保证。消息系统常常通过仅设1个消费者来解决这个问题，但是这意味着没用到并行处理。

kafka做的更好。通过并行topic的parition —— kafka提供了顺序保证和负载均衡。每个partition仅由同一个消费者组中的一个消费者消费到。并确保消费者是该partition的唯一消费者，并按顺序消费数据。每个topic有多个分区，则需要对多个消费者做负载均衡，但请注意，相同的消费者组中不能有比分区更多的消费者，否则多出的消费者一直处于空等待，不会收到消息。

- 作为一个存储系统

所有发布消息到消息队列和消费分离的系统，实际上都充当了一个存储系统（发布的消息先存储起来）。Kafka比别的系统的优势是它是一个非常高性能的存储系统。

写入到kafka的数据将写到磁盘并复制到集群中保证容错性。并允许生产者等待消息应答，直到消息完全写入。

kafka的磁盘结构 - 无论你服务器上有50KB或50TB，执行是相同的。

client来控制读取数据的位置。你还可以认为kafka是一种专用于高性能，低延迟，提交日志存储，复制，和传播特殊用途的分布式文件系统。

- 作为流处理

仅仅读，写和存储是不够的，kafka的目标是实时的流处理。

在kafka中，流处理持续获取输入topic的数据，进行处理加工，然后写入输出topic。例如，一个零售APP，接收销售和出货的输入流，统计数量或调整价格后输出。

可以直接使用producer和consumer API进行简单的处理。对于复杂的转换，Kafka提供了更强大的Streams API。可构建聚合计算或连接流到一起的复杂应用程序。

助于解决此类应用面临的硬性问题：处理无序的数据，代码更改的再处理，执行状态计算等。

Sterams API在Kafka中的核心：使用producer和consumer API作为输入，利用Kafka做状态存储，使用相同的组机制在stream处理器实例之间进行容错保障。

## 内部概念
kafka作为一个集群运行在一个或多个服务器上，存储的消息是以topic为类别记录的，每个消息是由一个key，一个value和时间戳构成。
![](/images/kafka/kafka.png)

### 四个核心API

- Producer API
发布消息到1个或多个topic（主题）

- Consumer API
订阅一个或多个topic，并处理产生的消息

- Streams API
充当一个流处理器，从1个或多个topic消费输入流，并生产一个输出流到1个或多个输出topic，有效地将输入流转换到输出流。

- Connector API

允许构建或运行可重复使用的生产者或消费者，将topic连接到现有的应用程序或数据系统。例如，一个关系数据库的连接器可捕获每一个变化。


### 基本术语

- Topic
Kafka将消息种子(Feed)分门别类，每一类的消息称之为一个主题(Topic).

- Producer
发布消息的对象称之为主题生产者(Kafka topic producer)

- Consumer
订阅消息并处理发布的消息的种子的对象称之为主题消费者(consumers)

- Broker
已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个主题（topic），并从Broker拉数据，从而消费这些已发布的消息。

### Topic和Log

Topic是发布的消息的类别或者种子Feed名。对于每一个Topic，Kafka集群维护这一个分区的log，就像下图中的示例：

![](/images/kafka/topic.png)

每一个分区都是一个顺序的、不可变的消息队列， 并且可以持续的添加。分区中的消息都被分了一个序列号，称之为偏移量(offset)，在每个分区中此偏移量都是唯一的。

Kafka集群保持所有的消息，直到它们过期， 无论消息是否被消费了。 实际上消费者所持有的仅有的元数据就是这个偏移量，也就是消费者在这个log中的位置。 这个偏移量由消费者控制：正常情况当消费者消费消息的时候，偏移量也线性的的增加。但是实际偏移量由消费者控制，消费者可以将偏移量重置为更老的一个偏移量，重新读取消息。 可以看到这种设计对消费者来说操作自如， 一个消费者的操作不会影响其它消费者对此log的处理。 再说说分区。Kafka中采用分区的设计有几个目的。一是可以处理更多的消息，不受单台服务器的限制。Topic拥有多个分区意味着它可以不受限的处理更多的数据。第二，分区可以作为并行处理的单元，稍后会谈到这一点。

![](/images/kafka/producer.png)

### 分布式(Distribution)

Log的分区被分布到集群中的多个服务器上。每个服务器处理它分到的分区。 根据配置每个分区还可以复制到其它服务器作为备份容错。 每个分区有一个leader，零或多个follower。Leader处理此分区的所有的读写请求，而follower被动的复制数据。如果leader宕机，其它的一个follower会被推举为新的leader。 一台服务器可能同时是一个分区的leader，另一个分区的follower。 这样可以平衡负载，避免所有的请求都只让一台或者某几台服务器处理。

### Geo-Replication(异地数据同步技术)

Kafka MirrorMaker为群集提供geo-replication支持。借助MirrorMaker，消息可以跨多个数据中心或云区域进行复制。 您可以在active/passive场景中用于备份和恢复; 或者在active/passive方案中将数据置于更接近用户的位置，或数据本地化。

### 生产者(Producers)

生产者往某个Topic上发布消息。生产者也负责选择发布到Topic上的哪一个分区。最简单的方式从分区列表中轮流选择。也可以根据某种算法依照权重选择分区。开发者负责如何选择分区的算法。

### 消费者(Consumers)

通常来讲，消息模型可以分为两种， 队列和发布-订阅式。 队列的处理方式是 一组消费者从服务器读取消息，一条消息只有其中的一个消费者来处理。在发布-订阅模型中，消息被广播给所有的消费者，接收到消息的消费者都可以处理此消息。Kafka为这两种模型提供了单一的消费者抽象模型： 消费者组 （consumer group）。 消费者用一个消费者组名标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。 假如所有的消费者都在一个组中，那么这就变成了queue模型。 假如所有的消费者都在不同的组中，那么就完全变成了发布-订阅模型。 更通用的， 我们可以创建一些消费者组作为逻辑上的订阅者。每个组包含数目不等的消费者， 一个组内多个消费者可以用来扩展性能和容错。正如下图所示：

![](/images/kafka/cluster.png)

2个kafka集群托管4个分区（P0-P3），2个消费者组，消费组A有2个消费者实例，消费组B有4个。

正像传统的消息系统一样，Kafka保证消息的顺序不变。 再详细扯几句。传统的队列模型保持消息，并且保证它们的先后顺序不变。但是， 尽管服务器保证了消息的顺序，消息还是异步的发送给各个消费者，消费者收到消息的先后顺序不能保证了。这也意味着并行消费将不能保证消息的先后顺序。用过传统的消息系统的同学肯定清楚，消息的顺序处理很让人头痛。如果只让一个消费者处理消息，又违背了并行处理的初衷。 在这一点上Kafka做的更好，尽管并没有完全解决上述问题。 Kafka采用了一种分而治之的策略：分区。 因为Topic分区中消息只能由消费者组中的唯一一个消费者处理，所以消息肯定是按照先后顺序进行处理的。但是它也仅仅是保证Topic的一个分区顺序处理，不能保证跨分区的消息先后处理顺序。 所以，如果你想要顺序的处理Topic的所有消息，那就只提供一个分区。

### Kafka的保证(Guarantees)

- 生产者发送到一个特定的Topic的分区上，消息将会按照它们发送的顺序依次加入，也就是说，如果一个消息M1和M2使用相同的producer发送，M1先发送，那么M1将比M2的offset低，并且优先的出现在日志中。

- 消费者收到的消息也是此顺序。

- 如果一个Topic配置了复制因子（replication factor）为N， 那么可以允许N-1服务器宕机而不丢失任何已经提交（committed）的消息。
有关这些保证的更多详细信息，请参见文档的设计部分。


## 使用实例

### 安装
使用docker-compose安装
```sh
docker-compose -f kafka.yml up -d
```
```yml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper
    container_name: zookeeper
    mem_limit: 1024M
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  kafka:
    image: confluentinc/cp-kafka
    container_name: kafka
    mem_limit: 1024M
    depends_on:
      - zookeeper
    ports:  # 增加
      - 9092:9092  # 增加
    environment:
      KAFKA_BROKER_NO: 1
      KAFKA_ADVERTISED_HOST_NAME: 192.168.43.137
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.43.137:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_HEAP_OPTS: "-Xmx512M -Xms16M"
```

查看状态
```sh
double@double:~/docker$ docker-compose -f kafka.yml ps
  Name               Command            State              Ports            
----------------------------------------------------------------------------
kafka       /etc/confluent/docker/run   Up      0.0.0.0:9092->9092/tcp      
zookeeper   /etc/confluent/docker/run   Up      2181/tcp, 2888/tcp, 3888/tcp
```

### 客户端访问

我们使用两个命令窗口分别打开作为生产者和消费者。

```sh
docker run -it --rm --name consumer confluentinc/cp-kafka /bin/bash
docker run -it --rm --name producer confluentinc/cp-kafka /bin/bash
```

#### 订阅消息
```sh
kafka-console-consumer --bootstrap-server 192.168.43.137:9092 --topic kafkatest --from-beginning
```

#### 发布消息

```sh
kafka-console-producer --broker-list 192.168.43.137:9092 --topic kafkatest
```

效果
```sh
double@double:~/docker$ docker run -it --rm --name producer confluentinc/cp-kafka /bin/bash
root@c7ee6e565283:/# kafka-console-producer --broker-list 192.168.43.137:9092 --topic kafkatest
>hello word
>
```

```sh
double@double:~/docker$ docker run -it --rm --name consumer confluentinc/cp-kafka /bin/bash
root@1a44f5e9f0c3:/# kafka-console-consumer --bootstrap-server 192.168.43.137:9092 --topic kafkatest --from-beginning
hello word
```

### GO实例

在使用kafka时，需要注意的是offset的管理，因为kafka消息就算消费了也依旧保存，而每次读取是根据offset去获取的。在实际应用中我们通常会涉及自动管理offset和手动管理offset。

我们使用segmentio/kafka-go作为client交互。
[https://github.com/segmentio/kafka-go](https://github.com/segmentio/kafka-go)

创建Reader
```sh
// make a new reader that consumes from topic-A, partition 0, at offset 42
r := kafka.NewReader(kafka.ReaderConfig{
    Brokers:   []string{"localhost:9092"},
    Topic:     "topic-A",
    Partition: 0,
    MinBytes:  10e3, // 10KB
    MaxBytes:  10e6, // 10MB
})

// 我们可以手动设置offset，默认可以通过CommitInterval: time.Second, // flushes commits to Kafka every second
r.SetOffset(42)

for {
    m, err := r.ReadMessage(context.Background())
    if err != nil {
        break
    }
    fmt.Printf("message at offset %d: %s = %s\n", m.Offset, string(m.Key), string(m.Value))
}
r.Close()
```

创建Writer
```sh
// make a writer that produces to topic-A, using the least-bytes distribution
w := kafka.NewWriter(kafka.WriterConfig{
	Brokers: []string{"localhost:9092"},
	Topic:   "topic-A",
	Balancer: &kafka.LeastBytes{},
})

w.WriteMessages(context.Background(),
	kafka.Message{
		Key:   []byte("Key-A"),
		Value: []byte("Hello World!"),
	},
	kafka.Message{
		Key:   []byte("Key-B"),
		Value: []byte("One!"),
	},
	kafka.Message{
		Key:   []byte("Key-C"),
		Value: []byte("Two!"),
	},
)

w.Close()
```

参考链接
[https://juejin.im/post/5bbcb21cf265da0add51dc38](https://juejin.im/post/5bbcb21cf265da0add51dc38)
[https://www.orchome.com/5](https://www.orchome.com/5)
[https://github.com/segmentio/kafka-go](https://github.com/segmentio/kafka-go)






