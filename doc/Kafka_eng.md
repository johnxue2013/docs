# kafka
## 目录
- 简介
    - 为什么需要使用MQ?
    - 安装并运行
- 核心概念
    - Topics和Logs
    - 生产者消费者
    - broker
    - 分区(partition)
- 核心API
    - Producer API
    - Consumer API
    - Streams API
    - Connector API
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

### 为什么需要使用MQ?


### 安装并运行

环境要求
- JDK1.7 或以上
- OS: window or Linux
- ZK

下载后，默认配置下使用命令
```
> bin/kafka-server-start.sh config/server.properties
```
即可启动Kafka

## 核心概念
### Topics和Logs
#### Topics
Kafka集群以类别(categories)存储记录，这个类别叫做主题(topic)，发送记录时，需要制定要发送到的主题。
一个主题可以有0个1个或者多个消费者订阅。

对于每一个Topic，Kafka集群维护着一个分区(partition)的log，就像下图中的示例:  

![image](http://kafka.apache.org/10/images/log_anatomy.png)  

每一个partition中的记录是有序且不可变的序列，分区中的每一个记录都会被分配一个分区范围内唯一的一个ID号，叫做偏移量(offset)，
用于唯一标识分区内每条记录。

> 为什么一个Topic可以有多个分区，这样做的好处是什么?  
- 可以处理更多的消息，不受单台服务器的限制。Topic拥有更多的分区意味着它可以不受限制的处理更多的数据
- 分区可以作为并行处理的单元。  

![image](http://kafka.apache.org/10/images/log_consumer.png)  

kafka保证了在服务端消息的顺序性，但不保证消费者收到消息的顺序性。如果要顺序性的处理topic中的所有消息，那就只提供一个分区。因为Kafka保证了一个分区中的消息只会由消费者组中的唯一一个消费者处理。  



# 概念
- kafka以一个集群运行在一台或者多台服务器上
- kafka集群以目录(categories)存储流式记录(record)，称作主题(topic)
- 每条记录由key,value,timstamp组成

> 关于kafka版本号的写法如: kafka_2.12-1.0.0，其中2.12是scala的版本号，后面的1.0.0才是kafka的版本号

> 关于下载kafka并运行，刚开始到github下载了一个kafka，发现运行不起来，后来才发现下载的是source，也就是源码。如果要运行的话应该下载binary，也就是执行文件。源码是不能运行的
# 核心API
- `Producer API`允许应用程序发布一条记录到一个或多个主题中
- `Consumer API`允许运行应用程序订阅一个或多个主题，并处理产生的的消息(record)
- `Streams API`充当一个流式处理器，从一个或多个topic消费输入流，再产生输出流到一个或多个topic，可以有效的转换输入流到输出流。
    > Sterams API在Kafka中的核心：使用producer和consumer API作为输入，利用Kafka做状态存储，使用相同的组机制在stream处理器实例之间进行容错保障。
- `Connector API`允许构建和运行可重复使用的生产者或消费者，将Kafka主题连接到现有的应用程序或数据系统。例如，连接到关系数据库的连接器可能会捕获对表的每个更改。
    > Connect API实现一个连接器（connector），不断地从一些数据源系统拉取数据到kafka，或从kafka推送到宿系统（sink system）。

![image](http://kafka.apache.org/10/images/kafka-apis.png)

# 使用场景
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

# 其他
Client和Server之间的通讯，是通过一条简单的、高性能并且和开发语言无关的**TCP协议**

## kafka中的术语

- Topic  
Kafka将消息分门别类，每一类的消息称之为一个主题(Topic).

- Producer  
发布消息的对象称之为主题生产者(Kafka topic producer)

- Consumer  
订阅消息并处理发布的消息的种子的对象称之为主题消费者(consumers)

- Broker  
已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个主题（topic），并从Broker拉数据，从而消费这些已发布的消息。

## Topic和Log  

# Kafka的保证(Guarantees)  
- 生产者发送到一个特定的Topic的分区上，消息将会按照它们发送的顺序依次加入，也就是说，如果一个消息M1和M2使用相同的producer发送，M1先发送，那么M1将比M2的offset低，并且优先的出现在日志中。  
- 消费者收到的消息也是此顺序。
- 如果一个Topic配置了复制因子（replication factor）为N， 那么可以允许N-1服务器宕机而不丢失任何已经提交（committed）的消息。


## 消费者组
通常来讲，消息模型可以分为两种， 队列和发布-订阅式。 队列的处理方式是 一组消费者从服务器读取消息，一条消息只有其中的一个消费者来处理。在发布-订阅模型中，消息被广播给所有的消费者，接收到消息的消费者都可以处理此消息。Kafka为这两种模型提供了单一的消费者抽象模型： 消费者组 （consumer group）。 消费者用一个消费者组名标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。 假如所有的消费者都在一个组中，那么这就变成了queue模型。 假如所有的消费者都在不同的组中，那么就完全变成了发布-订阅模型。 更通用的， 我们可以创建一些消费者组作为逻辑上的订阅者。每个组包含数目不等的消费者， 一个组内多个消费者可以用来扩展性能和容错

> kafka中消费者组有两个概念：`队列`：消费者组（consumer group）允许同名的消费者组成员瓜分处理。`发布订阅`：允许你广播消息给多个消费者组（不同名）。

## 多线程处理
Kafka消费者不是线程安全的，所有的`网络I/O`都发生在进行调用的应用程序线程中。因此多线程处理时需要确保多线程访问的同步。非同步访问将抛出`ConcurrentModificationException`。

此规则唯一的例外是wakeup()，它可以安全地从外部线程来中断活动操作。在这种情况下，将从操作的线程阻塞并抛出一个WakeupException。这可用于从其他线程来关闭消费者。 以下代码段显示了典型模式：
```
public class KafkaConsumerRunner implements Runnable {
     private final AtomicBoolean closed = new AtomicBoolean(false);
     private final KafkaConsumer consumer;

     public void run() {
         try {
             consumer.subscribe(Arrays.asList("topic"));
             while (!closed.get()) {
                 ConsumerRecords records = consumer.poll(10000);
                 // Handle new records
             }
         } catch (WakeupException e) {
             // Ignore exception if closing
             if (!closed.get()) throw e;
         } finally {
             consumer.close();
         }
     }

     // Shutdown hook which can be called from a separate thread
     public void shutdown() {
         closed.set(true);
         consumer.wakeup();
     }
 }
```

在单独的线程中，可以通过设置关闭标志和唤醒消费者来关闭消费者。

```
closed.set(true);
consumer.wakeup();
```
# Kafka Connect

Kafka Connect 是一个可扩展、可靠的在Kafka和其他系统之间流传输的数据工具。它可以通过connectors（连接器）简单、快速的将大集合数据导入和导出kafka。Kafka Connect可以接收整个数据库或收集来自所有的应用程序的消息到Kafka Topic。使这些数据可用于低延迟流处理。导出可以把topic的数据发送到secondary storage（辅助存储也叫二级存储）也可以发送到查询系统或批处理系统做离线分析。Kafka Connect功能包括：

- Kafka连接器通用框架：  
Kafka Connect 规范了kafka与其他数据系统集成，简化了connector的开发、部署和管理。

- 分布式和单机模式:  
扩展到大型支持整个organization的集中管理服务，也可缩小到开发，测试和小规模生产部署。

- REST 接口  
通过REST API来提交（和管理）connector到Kafka Connect集群。

- 自动的offset管理   
从connector获取少量的信息，Kafka Connect来管理offset的提交，所以connector的开发者不需要担心这个容易出错的部分。

- 分布式和默认扩展:  
- Kafka Connect建立在现有的组管理协议上。更多的工作可以添加扩展到Kafka Connect集群。

- 流/批量集成  
利用kafka现有的能力，Kafka Connect是一个桥接流和批量数据系统的理想解决方案。

## <span id = "why-need-mq">跳转到的位置</span>