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
- 常见操作
    - 创建和删除topic
    - 修改topic
    - leader 平衡
    - 检查消费者位置

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

> 如果不使用Kafka内置的ZK，则需要先启动ZK，再执行上述命令

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

kafka保证了在服务端消息的顺序性，但不保证消费者收到消息的顺序性。如果要顺序性的处理topic中的所有消息，那就只提供一个分区。因为**Kafka保证了一个分区中的消息只会由消费者组中的唯一一个消费者处理**。  

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

#### 发现消费者故障
订阅一组topic后，当调用`poll(long timeout)`时，消费者将自动加入到组中。只要持续的调用`poll`，消费者将一直保持可用，并继续从分配的分区中接收消息。此外，消费者向服务器定时发送心跳。 如果消费者崩溃或无法在`session.timeout.ms`配置的时间内发送心跳，则消费者将被视为死亡，并且其分区将被重新分配。

还有一种可能，消费可能遇到"活锁"的情况，它持续的发送心跳，但是没有处理。为了预防消费者在这种情况下一直持有分区，可以使用`max.poll.interval.ms`活跃检测机制。 在此基础上，如果你调用的poll的频率大于最大间隔，则客户端将主动地离开组，以便其他消费者接管该分区。 发生这种情况时，你会看到offset提交失败（调用`commitSync()`引发的`CommitFailedException`）。这是一种安全机制，保障只有活动成员能够提交offset。所以要留在组中，你必须持续调用`poll`。

消费者提供两个配置设置来控制poll循环：

`max.poll.interval.ms`：最大poll的间隔，可以为消费者提供更多的时间去处理返回的消息（调用`poll(long timeout)`返回的消息，通常返回的消息都是一批）。缺点是此值越大将会延迟组重新平衡。

`max.poll.records`：此设置限制每次调用poll返回的消息数，这样可以更容易的预测每次poll间隔要处理的最大值。通过调整此值，可以减少poll间隔，减少重新平衡分组的

对于消息处理时间不可预测地的情况，这些选项是不够的。 处理这种情况的推荐方法是将消息处理移到另一个线程中，让消费者继续调用`poll`。 但是必须注意确保已提交的offset不超过实际位置。另外，必须禁用自动提交，并只有在线程完成处理后才为记录手动提交偏移量（取决于你）。 还要注意，如果你的处理能力比拉取消息的慢，那创建新线程将导致你机器内存溢出，此时,你需要pause暂停分区，不会从poll接收到新消息，让线程处理完之前返回的消息。

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
## 配置
### broker配置
必须的配置项如下
- `broker.id`:
 unique and permanent name of each node in the cluster.
- `log.dirs`:
  The directories in which the log data is kept. If not set, the value in log.dir is used
- `zookeeper.connect`:
  Zookeeper host string

其他重要配置  

参数 | 默认值 | 描述
:--------|:---------|:-------
broker.id |	-1　|	每一个boker都有一个唯一的id作为它们的名字。当该服务器的IP地址发生改变时，broker.id没有变化，则不会影响consumers的消息情况
port |	9092	| broker server服务端口
host.name |	""	| broker的主机地址，若是设置了，那么会绑定到这个地址上，若是没有，会绑定到所有的接口上，并将其中之一发送到ZK
log.dirs	| /tmp/kafka-logs		| kafka数据的存放地址，多个地址的话用逗号分割,多个目录分布在不同磁盘上可以提高读写性能  /data/kafka-logs-1，/data/kafka-logs-2
message.max.bytes	| 	1000012		| 表示消息体的最大大小，单位是字节
num.network.threads		| 3		| broker处理消息的最大线程数，一般情况下数量为cpu核数
num.io.threads		| 8		| 处理IO的线程数
log.flush.interval.messages		| Long.MaxValue		| 在数据被写入到硬盘和消费者可用前最大累积的消息的数量
log.flush.interval.ms	| 	Long.MaxValue		| 在数据被写入到硬盘前的最大时间
log.flush.scheduler.interval.ms		| Long.MaxValue		| 检查数据是否要写入到硬盘的时间间隔。
log.retention.hours		| 168 (24*7)	| 	控制一个log保留多长个小时
log.retention.bytes		| -1		| 控制log文件最大尺寸
log.cleaner.enable		| false		| 是否log cleaning
log.cleanup.policy		| delete　	| 	delete还是compat.
log.segment.bytes		| 1073741824	| 	单一的log segment文件大小
log.roll.hours		| 168		| 开始一个新的log文件片段的最大时间
background.threads		| 10		| 后台线程数
num.partitions		| 1		| 默认分区数
zookeeper.connection.timeout.ms	| 	6000		| 指定客户端连接zookeeper的最大超时时间
zookeeper.session.timeout.ms　　	| 	6000		| 连接zk的session超时时间
zookeeper.sync.time.ms		| 2000		| zk follower落后于zk leader的最长时间

其他默认配置和topic级别的配置详见[此处][1]

### topic 配置
与主题相关的配置包含broker配置的默认值，也包含每个主题的覆盖至。如果没有topic级别的配置那么将使用broker配置的默认值。覆盖默认值可以在创建topic时使用一个或者多个`--config`选项。

如下语法创建一个名为**my-topic**的主题，并设置max message size和flush rate:
```Bash
> bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic my-topic --partitions 1 --replication-factor 1 --config max.message.bytes=64000 --config flush.messages=1
```

覆盖默认值也可通过`alter`参数修改一个已经创建的topic
```Bash
> bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name my-topic --alter --add-config max.message.bytes=128000
```

可以通过如下命令检查某个主题覆盖了那些默认值
```Bash
> bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name my-topic --describe
```

通过如下命令移除覆盖值
```Bash
> bin/kafka-configs.sh --zookeeper localhost:2181  --entity-type topics --entity-name my-topic --alter --delete-config max.message.bytes
```

其他topic相关配置详见[此处][2]

### producer配置

参数 | 默认值 | 描述
:--------|:---------|:-------
producer.type	 | sync | 指定消息发送是同步还是异步。异步asyc成批发送用kafka.producer.AyncProducer， 同步sync用kafka.producer.SyncProducer
metadata.broker.list	 | broker list	 | 使用这个参数传入boker和分区的静态信息，如host1:port1,host2:port2, 这个可以是全部boker的一部分
compression.codec	 | NoCompressionCodec	 | 消息压缩，默认不压缩
compressed.topics	 | null	 | 在设置了压缩的情况下，可以指定特定的topic压缩，未指定则全部压缩
message.send.max.retries	 | 3	 | 消息发送最大尝试次数
retry.backoff.ms	 | 300	 | 每次尝试增加的额外的间隔时间
topic.metadata.refresh.interval.ms	 | 600000 | 定期的获取元数据的时间。当分区丢失，leader不可用时producer也会主动获取元数据，如果为0，则每次发送完消息就获取元数据，不推荐。如果为负值，则只有在失败的情况下获取元数据。
queue.buffering.max.ms	 | 5000	 | 在producer queue的缓存的数据最大时间，仅仅for asyc
queue.buffering.max.message	 | 10000 | 	producer 缓存的消息的最大数量，仅仅for asyc
queue.enqueue.timeout.ms	 | -1	 | 0当queue满时丢掉，负值是queue满时block,正值是queue满时block相应的时间，仅仅for asyc
batch.num.messages | 	200	 | 一批消息的数量，仅仅for asyc
request.required.acks	 | 0	 | 0表示producer无需等待leader的确认，1代表需要leader确认写入它的本地log并立即确认，-1代表所有的备份都完成后确认。 仅仅for sync
request.timeout.ms	 | 10000	 | 确认超时时间

其他相关配置详见[此处][3]

### consumer配置
参数 | 默认值 | 描述
:--------|:---------|:-------
groupid	 | 	groupid | 一个字符串用来指示一组consumer所在的组
socket.timeout.ms	 | 	30000	 | 	socket超时时间
socket.buffersize	 | 	64*1024	 | 	socket receive buffer
fetch.message.max.bytes	 | 	1024 * 1024	 | 控制一个请求中获取的消息的字节数
fetch.min.bytes | 1 | The minimum amount of data the server should return for a fetch request. If insufficient data is available the request will wait for that much data to accumulate before answering the request.
auto.commit.enable	| true	| 如果true,consumer定期地往zookeeper写入每个分区的offset
auto.commit.interval.ms		| 10000		| 往zookeeper上写offset的频率
auto.offset.reset	| largest | 	如果ZK没有初始化的offset或者offset超出了范围时应该如何处理，smallest: 自动设置reset到最小的offset. largest : 自动设置offset到最大的offset. 其它值则向consumer抛出异常.
consumer.timeout.ms		| -1		| 默认-1,consumer在没有新消息时无限期的block。如果设置一个正值， 一个超时异常会抛出
rebalance.retries.max	| 	4		| rebalance时的最大尝试次数

其他相关配置详见[此处][4]

## 常见操作
### 创建和删除topic
用户可以手动创建一个topic也可以在producer发送消息到一个不存在的topic自动创建。使用如下命令可以创建一个topic
```Bash
> bin/kafka-topics.sh --zookeeper zk_host:port/chroot --create --topic my_topic_name --partitions 20 --replication-factor 3 --config <property key>=<property value>
```
复制因此表明有多少server将会复制消息，3台server将容忍2台server宕机而保证数据可访问(即N台server最多容忍N-1台server宕机)。

每个分区中的记录将被存放在Kafka的log目录中，文件名由topic的名字加上一个"-"符号，再加上一个分区ID(partition ID)构成

### 修改topic

**增加一个topic的分区数**
```Bash
> bin/kafka-topics.sh --zookeeper zk_host:port/chroot --alter --topic my_topic_name --partitions 40
```
> 注意：分区的个数的变化，不会影响已有数据，因此如果数据应该于改分区，则会对消费者造成影响。如数据是通过散列(键)%number_of_partitions进行分区的。Kafka不会尝试以任何方式自动重新分配数据。

> Kafka暂不支持减少分区数

**增加topic配置**
```Bash
> bin/kafka-configs.sh --zookeeper zk_host:port/chroot --entity-type topics --entity-name my_topic_name --alter --add-config <property key>=<property value>
```

**删除topic配置**
```Bash
> bin/kafka-configs.sh --zookeeper zk_host:port/chroot --entity-type topics --entity-name my_topic_name --alter --delete-config <property key>
```

**删除topic**
```Bash
> bin/kafka-topics.sh --zookeeper zk_host:port/chroot --delete --topic my_topic_name
```
### leader 平衡
默认情况下，无论一个broker是停止还是宕机在该broker上的分区leader都将转移到其他的副本上,也就是说当一个broker重启，它只能是其他分区的follower，将不用来处理client的读写。

为了避免这种平衡，Kafka有一个preferred replicas。比如副本的列表为1,5,9，则1将更有可能成为leader，因为1在表头。

可以使用如下命令是Kafka重新恢复已经恢复副本的leader地位：

```Bash
> bin/kafka-preferred-replica-election.sh --zookeeper <zk_host:port>/chroot
```

由于运行此命令是乏味的，可以配置Kafka自动执行该操作

```xml
auto.leader.rebalance.enable=true
```

### 检查消费者位置
如检查一个名为my-group的消费者组消费主题为my-topic的情况
```Bash
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group

Note: This will only show information about consumers that use the Java consumer API (non-ZooKeeper-based consumers).

TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                           CLIENT-ID
my-topic                       0          2               4               2          consumer-1-029af89c-873c-4751-a720-cefd41a669d6   /127.0.0.1                     consumer-1
my-topic                       1          2               3               1          consumer-1-029af89c-873c-4751-a720-cefd41a669d6   /127.0.0.1                     consumer-1
my-topic                       2          2               3               1          consumer-2-42c1abd4-e3b2-425d-a8bb-e1ea49b29bb2   /127.0.0.1                     consumer-2
```

该命令也可运行在基于ZK的集群中:
```Bash
> bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --describe --group my-group

Note: This will only show information about consumers that use ZooKeeper (not those using the Java consumer API).

TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID
my-topic                       0          2               4               2          my-group_consumer-1
my-topic                       1          2               3               1          my-group_consumer-1
my-topic                       2          2               3               1          my-group_consumer-2
Managing Consumer Groups
```
其他更多操作可以参考[官网][5]

[1]:http://kafka.apache.org/documentation/#brokerconfigs "broker配置列表"

[2]:http://kafka.apache.org/documentation/#topicconfigs "topic级别配置列表"

[3]:http://kafka.apache.org/documentation/#producerconfigs "producer配置列表"

[4]:http://kafka.apache.org/documentation/#consumerconfigs "consumer的配置列表"
[5]:http://kafka.apache.org/documentation/#operations "更多操作"
