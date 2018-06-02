# kafka消费者

## 偏移量和消费者的位置

kafka为每条消息保存了一个`偏移量(offset)`，这个`偏移量`是该分区中一条消息的唯一标识符。也表示消费者在分区的位置。

`已提交`的位置是已安全保存的最后偏移量，如果消费者进程失败或者重启时，消费者将恢复到这个偏移量。消费者可以选择定期自动提交偏移量，也可以选择通过调用commit API来手动的控制(如: commitSync和commitAsync)。此时一条消息什么时候被消费，控制权在消费者。

## 消费者组和订阅主题

Kafka的`消费者组`概念，通过进程池瓜分消费和处理消息的工作。这些进程可以在同一台机器运行，也可分布到多台机器上，增加可扩展性和容错性,相同group.id的消费者将视为同一个`消费者组`。

分组中的每个消费者通过`subscribe API`动态的订阅一个topic列表。kafka将已订阅topic的消息发送到每个消费者组中。并通过平衡分区在消费者分组中所有成员之间来达到平均。因此每个分区恰好地分配1个消费者（一个消费者组中）。所有如果一个topic有4个分区，并且一个消费者分组有2个消费者。那么每个消费者消费2个分区。

**消费者组的成员是动态维护的**：如果一个消费者故障。分配给它的分区将重新分配给同一个分组中其他的消费者。同样的，如果一个新的消费者加入到分组，将从现有消费者中移一个给它。这被称为`重新平衡分组`，并在下面更详细地讨论。 当新分区添加到订阅的topic时，或者当创建与订阅的正则表达式匹配的新topic时，也将重新平衡。将通过定时刷新自动发现新的分区，并将其分配给分组的成员。

当分组重新分配自动发生时，可以通过`ConsumerRebalanceListener`通知消费者，这允许他们完成必要的应用程序级逻辑，例如状态清除，手动偏移提交等。

## 发现消费者故障

订阅一组topic后，当调用poll(long）时，消费者将自动加入到组中。只要持续的调用poll，消费者将一直保持可用，并继续从分配的分区中接收消息。此外，消费者向服务器定时发送心跳。 如果消费者崩溃或无法在session.timeout.ms配置的时间内发送心跳，则消费者将被视为死亡，并且其分区将被重新分配。

还有一种可能，消费可能遇到"活锁"的情况，它持续的发送心跳，但是没有处理。为了预防消费者在这种情况下一直持有分区，我们使用max.poll.interval.ms活跃检测机制。 在此基础上，如果你调用的poll的频率大于最大间隔，则客户端将主动地离开组，以便其他消费者接管该分区。 发生这种情况时，你会看到offset提交失败（调用commitSync（）引发的CommitFailedException）。这是一种安全机制，保障只有活动成员能够提交offset。所以要留在组中，你必须持续调用poll。

消费者提供两个配置设置来控制poll循环：

- `max.poll.interval.ms`：最大poll的间隔，可以为消费者提供更多的时间去处理返回的消息（调用poll(long)返回的消息，通常返回的消息都是一批）。缺点是此值越大将会延迟组重新平衡。

- `max.poll.records`：此设置限制每次调用poll返回的消息数，这样可以更容易的预测每次poll间隔要处理的最大值。通过调整此值，可以减少poll间隔，减少重新平衡分组的

对于消息处理时间不可预测地的情况，这些选项是不够的。 处理这种情况的推荐方法是将消息处理移到另一个线程中，让消费者继续调用poll。 但是必须注意确保已提交的offset不超过实际位置。另外，你必须禁用自动提交，并只有在线程完成处理后才为记录手动提交偏移量（取决于你）。 还要注意，你需要pause暂停分区，不会从poll接收到新消息，让线程处理完之前返回的消息（如果你的处理能力比拉取消息的慢，那创建新线程将导致你机器内存溢出）。

## 示例

### 自动提交偏移量

```java
Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "true");
     props.put("auto.commit.interval.ms", "1000");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records)
             System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
     }
```

设置`enable.auto.commit`，偏移量由`auto.commit.interval.ms`控制自动提交的频率。

集群是通过配置`bootstrap.servers`指定一个或多个broker。不用指定全部的broker，它将自动发现集群中的其余的borker（最好指定多个，万一有服务器故障）。

在这个例子中，客户端订阅了主题`foo`和`bar`。消费者组叫`test`。

broker通过心跳机器自动检测test组中失败的进程，消费者会自动`ping`集群，告诉进群它还活着。只要消费者能够做到这一点，它就被认为是活着的，并保留分配给它分区的权利，如果它停止心跳的时间超过`session.timeout.ms`,那么就会认为是故障的，它的分区将被分配到别的进程。

这个`deserializer`设置如何把byte转成object类型，例子中，通过指定string解析器，我们告诉获取到的消息的key和value只是简单个string类型。

### 手动控制偏移量

不需要定时的提交offset，可以自己控制offset，当消息认为已消费过了，这个时候再去提交它们的偏移量。这个很有用的，当消费的消息结合了一些处理逻辑，这个消息就不应该认为是已经消费的，直到它完成了整个处理。

```java
Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "false");
     props.put("auto.commit.interval.ms", "1000");
     props.put("session.timeout.ms", "30000");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     final int minBatchSize = 200;
     List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records) {
             buffer.add(record);
         }
         if (buffer.size() >= minBatchSize) {
             insertIntoDb(buffer);
             consumer.commitSync();
             buffer.clear();
         }
     }
```

在这个例子中，我们将消费一批消息并将它们存储在内存中。当我们积累足够多的消息后，我们再将它们批量插入到数据库中。如果我们设置offset自动提交（之前说的例子），消费将被认为是已消费的。这样会出现问题，我们的进程可能在批处理记录之后，但在它们被插入到数据库之前失败了。

为了避免这种情况，我们将在相应的记录插入数据库之后再手动提交偏移量。这样我们可以准确控制消息是成功消费的。提出一个相反的可能性：在插入数据库之后，但是在提交之前，这个过程可能会失败（即使这可能只是几毫秒，这是一种可能性）。在这种情况下，进程将获取到已提交的偏移量，并会重复插入的最后一批数据。这种方式就是所谓的"至少一次"保证，在故障情况下，可以重复。

如果您无法执行这些操作，可能会使已提交的偏移超过消耗的位置，从而导致缺少记录。 使用手动偏移控制的优点是，您可以直接控制记录何时被视为"已消耗"。

上面的例子使用commitSync表示所有收到的消息为"已提交"，在某些情况下，你可以希望更精细的控制，通过指定一个明确消息的偏移量为"已提交"。在下面，我们的例子中，我们处理完每个分区中的消息后，提交偏移量。

```java
try {
         while(running) {
             ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
             for (TopicPartition partition : records.partitions()) {
                 List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
                 for (ConsumerRecord<String, String> record : partitionRecords) {
                     System.out.println(record.offset() + ": " + record.value());
                 }
                 long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
                 consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
             }
         }
     } finally {
       consumer.close();
     }
```

> 注意：已提交的offset应始终是你的程序将读取的下一条消息的offset。因此，调用commitSync（offsets）时，你应该加1个到最后处理的消息的offset。

## 订阅指定的分区

在前面的例子中，我们订阅我们感兴趣的topic，让kafka提供给我们平分后的topic分区。但是，在有些情况下，你可能需要自己来控制分配指定分区，例如：

- 如果这个消费者进程与该分区保存了某种本地状态（如本地磁盘的键值存储），则它应该只能获取这个分区的消息。

- 如果消费者进程本身具有高可用性，并且如果它失败，会自动重新启动（可能使用集群管理框架如YARN，Mesos，或者AWS设施，或作为一个流处理框架的一部分）。 在这种情况下，不需要Kafka检测故障，重新分配分区，因为消费者进程将在另一台机器上重新启动。

要使用此模式，，你只需调用

```java
assign（Collection）
```

消费指定的分区即可：

```java
String topic = "foo";
TopicPartition partition0 = new TopicPartition(topic, 0);
TopicPartition partition1 = new TopicPartition(topic, 1);
consumer.assign(Arrays.asList(partition0, partition1));
```

一旦手动分配分区，你可以在循环中调用poll（跟前面的例子一样）。消费者分组仍需要提交offset，只是现在分区的设置只能通过调用`assign`修改，因为手动分配不会进行分组协调，因此消费者故障不会引发分区重新平衡。每一个消费者是独立工作的（即使和其他的消费者共享GroupId）。为了避免offset提交冲突，通常你需要确认每一个consumer实例的gorupId都是唯一的。

**注意，手动分配分区（即，assgin）和动态分区分配的订阅topic模式（即，subcribe）不能混合使用。**

## 消费者流量控制

如果消费者分配了多个分区，并同时消费所有的分区，这些分区具有相同的优先级。在一些情况下，消费者需要首先消费一些指定的分区，当指定的分区有少量或者已经没有可消费的数据时，则开始消费其他分区。

例如流处理，当处理器从2个topic获取消息并把这两个topic的消息合并，当其中一个topic长时间落后另一个，则暂停消费，以便落后的赶上来。

kafka支持动态控制消费流量，分别在future的poll(long)中使用pause(Collection) 和 resume(Collection) 来暂停消费指定分配的分区，重新开始消费指定暂停的分区。

## 多线程处理
