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
- Sender线程网络通信的模块：必须要搞清楚kafka客户端是如何通过网络通信把一批消息发送到broker上去的。网络通信的设置的一些对应的参数，应对网络故障，人家是怎么来做的。
- 深入学习磁盘读写这块是如何来实现的，消息是如何写入磁盘的，存储结构，怎么去使用os page cache，怎么实现磁盘文件的顺序写。
- 副本冗余以及高可用的架构设计，leader和follower是如何同步的，副本是如何传输的，offset是如何变更的，如果leader所在的broker故障了，是如何进行leader，follower切换的。

### 消费端源码剖析

- 每个消费组的主控节点是如何来选择的，group coordinator 如何选择，consumer group leader 如何选择；
- 分区分配的方法；
- 分布式消费的实现机制；
- 拉取消息的原理；
- offset提交的原理。

#### Kafka自定义的基于TCP/IP的二进制协议

kafka自定义了一组二进制协议，现在是一共包含了43种协议类型，每种协议都有对应的请求和响应，其实可以认为是几十个接口，每个接口对应一种协议。

每一种协议都对应一个Request，一个Response。每个Request和Response都有header和body。

每个协议的Request都有相同的请求头（RequestHeader），不同的请求体。

请求头包含了：

- api_key：就类似于"PRODUCE"，"FETCH"，可以认为是接口的名字；
- api_version：就是这个API的版本号；
- correlation_id :类似客户端生成的一次请求的唯一标识号；
- client_id：就是客户端id；

每个协议的Response也有相同的响应头，就是一个correlation_id，就是对某个请求的响应。

比如说发送消息，就是ProduceRequest和ProduceResponse，代表"PRODUCE"这个接口的请求和响应。

##### ProduceRequest的RequestBody包含：

- transactional_id（事务id）；
- asks（会告诉broker是只写到leader以后就可以返回响应，还是要写到leader以后还要等所有follower也需要都写完才返回响应）；
- timeout（如果在指定时间范围比如timeout这个参数设置为30s，30s内不能返回响应，那就得超时，返回一个响应）；
- topic_data(topic,data(partition,record_set))

​        发给你的数据里会包含很多的batch，batch可能是属于不同topic的，所以会按topic来分组，每个topic又对应了哪些partition，每个partition对应的数据。

按照这种格式，把数据组织好，以字节数组流的形式传过去。

##### ProduceResponse的ResponseBody包含：

broker收到请求做出处理后，会返回响应。包含：

- topic，partition_response(partition,error_code,base_offset,log_append_time,log_start_offset),throttle_time_ms 每个分区对应的响应（有没有异常，写入的时候是从哪个offset开始写的，写入时间）

