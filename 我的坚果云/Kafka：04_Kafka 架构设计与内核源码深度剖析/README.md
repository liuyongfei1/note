### Kafka的集群架构设计

- 各个broker启动的时候，如何组成一个集群；
- 集群的主控节点是如何选举出来的；
- 后续如何监控集群的各个broker是否正常运行，是否故障宕机，如何对故障的broker找备用的来进行替代；
- 集群的元数据（topic，partition，leader/follower，isr）；
- 负责均衡，是如何保证数据均匀的分布在集群的各个broker机器上，如何进行topic的partition扩容，让一个broker通过partition扩容来使用集群里更多的broker的机器资源；
- 伸缩架构，broker的扩容，如何通过加更多的broker机器来扩容集群的存储资源以及网络资源。

### 生产端源码剖析

最核心的两块：

- Kafka客户端是如何去设计一个非常优秀的生产级的保证高吞吐的一个缓冲机制；
- Send线程网络通信的模块：必须要搞清楚kafka客户端是如何通过网络通信把一批消息发送到broker上去的。网络通信的设置的一些对应的参数，应对网络故障，人家是怎么来做的。
- 深入学习磁盘读写这块是如何来实现的，消息是如何写入磁盘的，存储结构，怎么去使用os page cache，怎么实现磁盘文件的顺序写。
- 副本冗余以及高可用的架构设计，leader和follower是如何同步的，副本是如何传输的，offset是如何变更的，如果leader所在的broker故障了，是如何进行leader，follower切换的。

### 消费端源码剖析

- 每个消费组的主控节点是如何来选择的，group coordinator 如何选择，consumer group leader 如何选择；
- 分区分配的方法；
- 分布式消费的实现机制；
- 拉取消息的原理；
- offset提交的原理。