# kafka
## 目录
- 简介
    - 使用场景
    - 安装并运行
- 核心概念
    - Topics和Logs
    - 生产者消费者
    - broker
    - 分区(partition)和副本(replication)
- Kafka的保证
  - 数据可靠性保证
  - 数据一致性保证
  - 消息投递保证
- 配置
	- Broker配置
	- Topic配置
	- Producer配置
	- Consumer配置
- 设计与原理
	- 消息(Messages)
    - 生产者
    - 消费者
    - 消息传递保障
    - 副本和leader选举
    - 日志压缩
    -
- 操作
    - 基本操作
        - Adding and removing topics
        - Modifying topics
        - Graceful shutdown
        - Balancing leadership
        - Checking consumer position
        - Mirroring data between clusters
        - Expanding your cluster
        - Decommissioning brokers
        - Increasing replication factor
    - 重要的配置
        - Important Client Configs
        - A Production Server Configs


## 简介
Kafka是一个分布式的流式平台,提供三个关键功能:
- 发布和订阅记录流，类似于消息队列或企业消息传递系统
- 以容错持久的方式存储记录流
- 实时处理发生的记录流

由LinkedIn公司开发(已捐赠Apache)，使用Scala语言编写，Kafka作为一个集群运行在一台或多台可以跨越多个数据中心的服务器上。

Kafka通常用于两大类应用:
- 构建可在系统或应用程序之间可靠获取数据的实时流数据管道
- 构建实时流应用程序，用于转换或响应数据流

### 使用场景
- Messaging (消息中间件)  
 消息一般要求低吞吐量，但是要求在端到端的延时最小。kafka可以保证消息的持久性（不会丢失），在这种案例下，kafka和ActiveMQ或者RabbitMQ相当。
- Website Activity Tracking (网页活动追踪)  
 Kafka最初的使用案例就是将用户的活动（页面浏览、搜索、或者其他一些用户可能发出的动作）跟踪管道重建为一组实时发布-订阅源。这些源可以被广泛的使用，如实时的处理。实时监控，或者加载数据到Hadoop以供线下处理和生成报告。
- Metrics (度量)  
卡夫卡通常用于运行监控数据。这涉及从分布式应用程序汇总统计数据以生成操作数据的集中式源。
- Log Aggregation (日志聚合)  
许多人使用Kafka作为日志聚合解决方案的替代品。 日志聚合通常从服务器收集物理日志文件，并将其置于中央位置（可能是文件服务器或HDFS）进行处理。 Kafka抽象提取文件的细节，并将日志或事件数据作为消息流进行更清晰的抽象。 这样可以实现更低延迟的处理，并且更容易支持多个数据源和分布式数据消耗。 与Scribe或Flume等以日志为中心的系统相比，Kafka提供同样出色的性能，由复制产生的更强大的持久性保证以及更低的端到端延迟。

- Stream Processing (流处理)  
Kafka的许多用户在处理管道中处理数据，这些数据由多个阶段组成，其中原始输入数据从Kafka主题中消耗，然后聚合，丰富或以其他方式转化为新的主题，以供进一步消费或后续处理。  
    > 除了Kafka Streams之外，替代性的开源流处理工具还包括Apache Storm和Apache Samza

- Event Sourcing  
事件源是应用程序设计的一种风格，其中状态更改以时间排序的记录序列进行记录。 Kafka对非常大的存储日志数据的支持使得它成为以这种风格构建的应用程序的优秀后端。

- Commit Log (提交日志)  
Kafka可以作为分布式系统的一种外部提交日志。 日志有助于复制节点之间的数据，并作为失败节点恢复数据的重新同步机制。 Kafka中的日志压缩功能有助于支持这种用法。 在这个用法中，Kafka与Apache BookKeeper项目类似。  

- 其他
Client和Server之间的通讯，是通过一条简单的、高性能并且和开发语言无关的**TCP协议**


### 安装并运行

环境要求
- JDK1.7 或以上
- OS: window or Linux
- ZK

下载后，默认配置下使用命令
```Bash
> bin/kafka-server-start.sh config/server.properties
```
即可启动Kafka

## 核心概念
### Topics和Logs
Kafka集群以类别(categories)存储记录，这个类别叫做主题(topic)，发送记录时，需要制定要发送到的主题。
一个主题可以有0个1个或者多个消费者订阅。

对于每一个Topic，Kafka集群维护着一个分区(partition)的log，就像下图中的示例:  

![image](http://kafka.apache.org/10/images/log_anatomy.png)  

每一个partition中的记录是有序且不可变的序列，分区中的每一个记录都会被分配一个分区范围内唯一的一个ID号，叫做偏移量(offset)，用于唯一标识分区内每条记录。

> 为什么一个Topic可以有多个分区，这样做的好处是什么?  
- 可以处理更多的消息，不受单台服务器的限制。Topic拥有更多的分区意味着它可以不受限制的处理更多的数据
- 分区可以作为并行处理的单元。  

![image](http://kafka.apache.org/10/images/log_consumer.png)  

kafka保证了在服务端消息的顺序性，但不保证消费者收到消息的顺序性。如果要顺序性的处理topic中的所有消息，那就只提供一个分区。因为Kafka保证了一个分区中的消息只会由消费者组中的唯一一个消费者处理。  

> 每条记录由key,value,timstamp组成

> 关于kafka版本号的写法如: kafka_2.12-1.0.0，其中2.12是scala的版本号，后面的1.0.0才是kafka的版本号


### 生产者和消费者
生产者负责向broke发送消息到一个topic，生产者可以选择随机的发送到topic的任意分区，或者通过某些语义发送到指定的分区。

消费者负责消费Kafka中的消息，消费者通过`consumer group`来标识自己，topic中的记录将被发送到每一个订阅它的`consumer group`中的一个`consumer`。

如果所有的`consumer`都属于同一个`consumer group`那么消息将在所有的消费者之间负载均衡。

如果所有的消费者实例属于不同的`consumer group`，那么消息将广播给所有的消费者.

![image](http://kafka.apache.org/11/images/consumer-groups.png)

Kafka中的分区将按照`consumer`个数划分到每个`consumer `实例中，当`consumer group`中有新加入的`consumer`时，新加入的`consumer`将从其他`consumer`中接管一些分区，如果一个`consumer`死亡，它所消费的分区将被分配给其他`consumer`实例。

![image](http://kafka.apache.org/10/images/kafka-apis.png)

### broker
一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。

### 分区(partition)和副本(replica)
为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，一个topic可以分为多个partition，每个partition是一个有序的队列。每个partition可以有多个replica。partition中的每条消息都会被分配一个有序的id（offset）。kafka只保证按一个partition中的顺序将消息发给consumer，不保证一个topic的整体（多个partition间）的顺序。

partition的多个replica中一个为leader，其余为follow，Producer只与Leader交互，把数据写到leader中，follower从Leader中拉取数据进行同步

> ISR(In-Sync-Replicas):是Replicas的一个子集，由leader replicas维护,包含所有不落后的replica集合, 不落后有两层含义:距离上次FetchRequest的时间不大于某一个值或落后的消息数不大于某一个值,任一超过阈值后该Follower将被踢出ISR,放入OSR(Outof-Sync-Replicas)。Leader失败后会从ISR中选取一个Follower做Leader

![image](https://github.com/johnxue2013/tools/blob/master/images/20140415105154875.png)

## Kafka的保证(Guarantees)  
### 数据可靠性保证
当Producer向Leader发送数据时,可以通过`acks`参数设置数据可靠性的级别

- 0: 不论写入是否成功,server不需要给Producer发送Response,如果发生异常,server会终止连接,触发Producer更新meta数据;
- 1: Leader写入成功后即发送Response,此种情况如果Leader fail,会丢失数据
- -1: 或者为`all`，等待所有ISR接收到消息后再给Producer发送Response,这是最强保证
仅设置acks=-1也不能保证数据不丢失,当Isr列表中只有Leader时,同样有可能造成数据丢失。要保证数据不丢除了设置acks=-1, 还要保证ISR的大小大于等于2,具体参数设置:
  - `request.required.acks`:设置为-1 等待所有ISR列表中的Replica接收到消息后采算写成功;
  - `min.insync.replicas`: 设置为大于等于2,保证ISR中至少有两个Replica

    Producer要在吞吐率和数据可靠性之间做一个权衡

如果一个Topic配置了复制因子（replication factor）为N， 那么可以允许N-1服务器宕机而不丢失任何已经提交（committed）的消息。

### 数据一致性保证
一致性定义:若某条消息对Consumer可见,那么即使Leader宕机了,在新Leader上数据依然可以被读到。

1. `HighWaterMark`简称HW: Partition的高水位，取一个partition对应的ISR中最小的LEO(LogEndOffset)作为HW，消费者最多只能消费到HW所在的位置，另外每个replica都有highWatermark，leader和follower各自负责更新自己的highWatermark状态，highWatermark <= leader. LogEndOffset

2. 对于Leader新写入的msg，Consumer不能立刻消费，Leader会等待该消息被所有ISR中的replica同步后,更新HW,此时该消息才能被Consumer消费，即Consumer最多只能消费到HW位置

### 消息投递保证(message delivery guarantee)
有如下几种可能
- `At most once` 消息可能会丢，但绝不会重复传输
- `At least one` 消息绝不会丢，但可能会重复传输
- `Exactly once` 每条消息肯定会被传输一次且仅传输一次，很多时候这是用户所想要的。

#### 从producer到broker(仅限于high-api)
Producer向broker发送消息时，一旦这条消息被commit，因数replication的存在，它就不会丢。但是如果Producer发送数据给broker后，遇到网络问题而造成通信中断，那Producer就无法判断该条消息是否已经commit。所以目前默认情况下一条消息从Producer到broker是确保了`At least once`，可通过设置Producer异步发送实现`At most once`

#### 从broke到consumer
读完消息先commit消费状态(保存offset)再处理消息。这种模式下，如果Consumer在commit后还没来得及处理消息就crash了，下次重新开始工作后就无法读到刚刚已提交而未处理的消息，这就对应于`At most once`

读完消息先处理再commit消费状态(保存offset)。这种模式下，如果在处理完消息之后commit之前Consumer crash了，下次重新开始工作时还会处理刚刚未commit的消息，实际上该消息已经被处理过了。这就对应于`At least once`。在很多使用场景下，消息都有一个主键，所以消息的处理往往具有幂等性，即多次处理这一条消息跟只处理一次是等效的，那就可以认为是`Exactly once`。

如果一定要做到`Exactly once`，就需要协调offset和实际操作的输出。经典的做法是引入两阶段提交。如果能让offset和操作输入存在同一个地方，会更简洁和通用。这种方式可能更好，因为许多输出系统可能不支持两阶段提交。比如，Consumer拿到数据后可能把数据放到HDFS，如果把最新的offset和数据本身一起写到HDFS，那就可以保证数据的输出和offset的更新要么都完成，要么都不完成，间接实现`Exactly once`。（目前就high level API而言，offset是存于Zookeeper中的，无法存于HDFS，而low level API的offset是由自己去维护的，可以将之存于HDFS中）

> 总之，Kafka默认保证`At least once`，并且允许通过设置Producer异步提交来实现`At most once`。而`Exactly once`要求与外部存储系统协作，幸运的是Kafka提供的offset可以非常直接非常容易得使用这种方式。
## 多线程处理
