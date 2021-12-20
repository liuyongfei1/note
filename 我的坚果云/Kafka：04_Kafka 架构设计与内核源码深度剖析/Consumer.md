### Consumer端主要知识点

#### consumer group coordinator

每个consumer group都会选择一个broker做为自己的coordinator，它负责监控这个消费组里的各个消费者的心跳，判断是否宕机，以及开启rebalance的。

它主要负责rebalance的，说白了你的consumer group中的每个consumer启动完毕后就会根据选举出来的这个coordinator所在的broker的机器进行通信，然后由coordinator分配分区给consumer来进行消费。

##### 那么是怎么选哪一个broker做为coordinator

1. 首先对group-id进行hash，得出一个数字；
2. 接着对__consumer_offsets的分区数量取模，默认是50个分区；
3. 就可以找到你的这个consumer group的offset要提交到__consumer_offsets哪个分区了；
4. __consumer_offsets分区的副本数量默认来说是1，也就是只有1个leader，然后对这个分区找到对应的leader所在的broker就是这个consumer group的coordinator了；
5. 接着这个consumer group里的所有消费者就会与这个coordinator建立Socket连接，发起心跳，coordinator就会对这个consumer进行监控。

有个consumer-group的概念。

#### Coordinator和Consumer Leader如何协作指定分区方案

1. 一个Consumer group里的consumer都启动后，会找到coordinator；

2. 每个consumer会发出 joinGroup请求，coordinator会等待一段时间，确保group里的每个consumer都进来发送 joinGroup请求后，就可以知道这个组里都有哪些consumer了；
3. 在这个consumer group里选择一个 consumer当做leader；
4. 然后通知这个leader，让leader来负责指定分区分配方案（每个consumer要消费哪些分区）；
5. 这个leader制定好以后，会发送SyncGroup请求，告诉coordinator，分区分配方案指定好了；
6. 接着coordinator就会把分区方案下发给你各个cosumer，consumer就会知道自己要从哪个分区去消费数据了，然后与指定分区的leader broker开始进行socket连接以及消费消息。

#### 1.消费过程

1. 每个consumer初始化启动完毕，都会发送请求到某一个broker的consumer group coordinator上，加入group；
2. 如果coordinator确认一个consumer group组建完毕，此时就会选择一个consumer leader出来；
3. consumer leader接到通知说自己是leader了，接着就会制定一份分区消费的方案（每个分区只会分配给group内的一个consumer）但一个consumer可以消费多个分区，返回给coordinator，下发给所有的consumer知道；
4. 然后大家就开始消费消息了，单线程，一次拉取一批数据过来，进行消费。

#### 2.offset作用

默认是后台自动提交offset，写入内存__consumer_offsets 这个topic里去。作用是知道每个consumer group对一个kafka的各个分区都消费到哪里了。

当你的一个consumer停机了，重启后，此时就会接着上一次提交的offset继续进行消费。

##### 几个offset

内存里会记录以下几个offset：

- 上一次提交的offset：consumer向coorinator提交offset；

- 当前消费到的offset：不断的在消费消息，不停的更新当前消费到的offset；

- LEO：leader partition 已经更新到的一个offset（比如写入一条数据，此时LEO=1，但是写入第一条数据的offset=0，LEO永远是大于最后一条数据的offset）。但是HW在前面，follower只能拉取到HW之前的数据。因为HW后面的数据，不是所有的follower都写入进去了，所以消费者是不能读取的。

- HW：

  在ISR列表里可能会有多个副本，比如一个leader对应的几个follower，leader和follower之间最小的LEO就是 HW。假设你要拉取一个分区的数据，只有HW之前的数据才能拉取到。

#### 3.rebalance

- 如果consumer挂掉，coordinator感知到了之后，就会rebalance，把那个挂掉的consumer消费的分区分配给其它的consumer。

- 如果新增consumer，此时会考虑将已有consumer的分区转移给新增的consumer来进行消费。新的consumer直接会接着那个分区被提交过的offset继续进行消费。

### 一.consumer初始化是如何加入group的

Kafka源码：server/KafkaApis里的JoinGroup =》kafka.coordinator.group.GroupCoordinator类：

1. 一开始各个consumer是：PreparingRebalance

过来一个组成员，就加入group。

2. 都加入完了之后，AwaitingSync状态。这个状态下会进行选举，选举一个leader出来

3. 然后是：CompletingRebalance

<img src="Consumer.assets/Consumer Coordinator是如何选举出来Consumer leader.png" alt="Consumer Coordinator是如何选举出来Consumer leader" style="zoom:80%;" />

#### AutoCommitTask线程是如何执行提交offset任务的？

```java
org.apache.kafka.clients.consumer.internals
  
```

ConsumerCoordinator.java类：

```java
private final AutoCommitTask autoCommitTask;
```

就是提交给coordinator所在机器。

### 实际场景

在移山系统的-实时服务功能里，有很多地方都用到了Kafka，还是数据库变更订阅服务（perceptor）里，也是client client，去消费kafka里的数据，并将数据解析，存入hbase或mysql。 



**总结移山实时同步，消费kafka消息时的乱序处理机制：**

**解决方案：**

将数据主键做为目标表的rowkey，利用binlog里的日志时间戳和偏移量，借助HBase-client API来完成对HBase表的操作，只有服务端对应rowkey的列数据与预期的值符合期望条件（大于、小于、等于）时，才会将put操作提交至服务端。

主要逻辑为：

1. 如果update_info（列族）execute_time（列）不存在，说明这是一条新数据，交本次put操作，插入数据；

2. 如果存在则会返回false，不会提交本次put操作，则去比较执行时间：

- 如果本次传递的执行时间小于HBase中的执行时间，则证明这条消息是重复消息或乱序的消息，直接丢弃 ；

- 如果本次传递的执行时间大于HBase中的执行时间，则插入put对象；

- 执行时间相等时，则去比较偏移量:
  - 如果本次传递的偏移量小于HBase中的存储的偏移量，则证明这条消息是重复消息，直接丢弃；
  - 如果本次传递的偏移量大于HBase中的存储的偏移量，则插入put对象，更新数据。