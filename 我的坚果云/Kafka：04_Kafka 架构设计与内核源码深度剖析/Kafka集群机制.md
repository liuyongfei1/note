#### 大数据技术栈

分布式存储（HDFS）、分布式资源调度（YARN）、分布式计算（MapReduce/Spark）、分布式数据仓库（Hive）、分布式NoSQL数据库（HBase）、分布式即席查询（Presto/Impala）、分布式OLAP分析引擎（Kylin）、分布式流式引擎（Storm/Spark Stearming/Flink）。

Zookeeper 是分布式系统中基石型组件。大数据本身是分布式系统，所以会大量依赖于Zookeeper。

#### 创建topic的时候是如何将副本均匀的分配给各个Broker的？

客户端创建topic  -》从zk里读取集群的元数据 -》找到Controller =》 请求会发给Controller

基于均匀分配的算法，将副本均匀分配给各个Broker。

Controller会注册针对节点 "broker/topics" 的监听

#### 扩容Kafka集群时如何触发副本重平衡

**进行迁移的流程**

Controller会注册监听器，监听zk的指定节点，比如："/admin/reassign_partitions"。如果有变更，执行重分配方案，从哪个broker迁移到其它broker。

1，2，3，4，5，6

- 一旦在zk中更新了4，5，6三个副本之后，Controller一定会感知到，一定会推送元数据到各个follower所在的broker上；
- 4，5，6三个broker感知到自己负责了新的follower之后，一定会启动Fetcher线程去leader那拉取数据；
- 在同一时间可能会出现一个分区有6个副本的情况，但是新分配的副本其实都是follower；
- 这些follower其实都在跟leader进行sync同步，等待，一直到这些follower全部跟leader同步为止；
- 新的follower的LEO大于leader的HW，则会被假如ISR中；
- 在6个副本中重新选举一个leader出来，是从4，5，6三个副本中来选举的；
- 从ISR列表中删除1，2，3 三个副本；
- 从zk中也删除1，2，3三个副本，就不再是follower了；
- 最后就是zk里是4，5，6三个副本，ISR里也是4，5，6三个副本，leader是4。

#### 总结：Kafka集群感知

1. 整个集群都启动以后，所有的broker会竞争，选举出来一个Controller，会将集群的元数据推送给所有的broker；
2. 使用客户端脚本创建topic，走分区副本分配方案，写入ZK，Controller会感知到，再次将元数据推送给所有的broker，broker会知道自己是leader还是follower；
3. 客户端发送消息时，会拉取元数据，从而会知道这个topic有几个分区，分区副本的leader在哪台broker上，然后就开始向这些broker上发送数据，写到各个分区对应的batch里，把属于同一个分区的batch打包成Request，给broker发送过去。

#### 总结：新增一个Broker(rebalance)

1. 需要进行重reblance，重平衡，生成重平衡方案，写入ZK；
2. Controller感知到后，会在新的Broker上起follower，同步数据。数据同步完后，加入ISR列表，重新选举leader，并将原来的副本从磁盘和zk里干掉。
3. 这样就实现了数据的迁移。

#### 总结：挂掉一个Broker（failover机制）

1. 当某个broker挂掉后，Controller会感知到，就会重新选举副本的leader，下发给各个broker；
2. Producer客户端如果还在向之前挂掉的broker写数据，会发现写入失败，然后就会重新拉取元数据，客户端就会感知到新的leader是谁，然后就会向leader所在的broker发送数据了。



