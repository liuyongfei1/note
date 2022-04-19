### kafka这个实战课程大纲

平时我们很有可能是自己用到Kafka的。

比如实时计算。storm -> spark streaming -> flink

是一个数据处理的通道，这里无论你用什么技术来做实时计算，大都是先从kafka里消费出数据，处理，接着再把数据写回kafka里。

**每秒钟会涌入多少数据，需要支撑多大的吞吐量，包括每天有多少数据需要落地在磁盘来存储，集群需要存储多大的数据量。**

kafka这个实战课程分为哪几个环节的步骤去讲：

1. 先明白kafka的工作原理；
2. 生产环境怎么部署；
3. 集群的日常管理；
4. 怎么来用；
5. 生产环境会遇到的问题；
6. 接下来就可以玩各种数据的采集了。用户行为日志怎么来采集，怎么埋点，离线导入hdfs，实时导入kafka；
7. 数据库业务数据怎么采集；
8. 爬虫，基于java自研一套分布式爬虫系统，分布式是个关键点，N多个爬虫系统协作起来分布式抓取数据，离线导入hdfs，实时导入kafka。

数据采集这个大的项目就做完了。

### 什么是吞吐量和延迟

#### 吞吐量

比如kafka每毫秒可以处理1条数据，每秒可以处理1000条数据，这个单位时间内可以处理多少条数据，就叫做吞吐量。

算吞吐量的时候还有另外一个单位，比如1000条数据，每条数据10kb，1000条数据相当于10mb的数据，吞吐量相当于是每秒处理10mb的数据。

#### 延迟

写数据请求发送给kafka一直到它处理成功，假如是1毫秒，这个就说明性能很高，延迟是1毫秒。

### Kafka高吞吐低延迟是怎么做到的

<img src="README.assets/Kafka低延迟高吞吐原理分析.png" alt="Kafka低延迟高吞吐原理分析" style="zoom:80%;" />



2、Kafka低延迟高吞吐原理分析(非零拷贝)



![Kafka低延迟高吞吐原理分析(README.assets/Kafka低延迟高吞吐原理分析(非零拷贝)-8728987.png)](../../../自己画的流程图/Kafka/Kafka低延迟高吞吐原理分析(非零拷贝).png)

3、Kafka低延迟高吞吐原理分析(零拷贝)

![Kafka低延迟高吞吐原理分析(README.assets/Kafka低延迟高吞吐原理分析(零拷贝)-8729065.png)](../../../自己画的流程图/Kafka/Kafka低延迟高吞吐原理分析(零拷贝).png)

零拷贝并不是不需要拷贝，而是减少拷贝次数。通常是说在IO读写过程中。

传统的IO流程，比如读取文件，再通过socket发送出去，传统方式实现：

1、从磁盘中读取文件，读取到操作系统内核缓冲区；

2、将操作系统内核缓冲区的数据拷贝到 application应用程序的buffer里；

3、将应用程序的buffer拷贝到socket网络发送缓冲区；

4、将socket buffer 的数据，拷贝到网卡，由网卡进行网络传输。

可以看到，传统方式，读取磁盘文件再到进行网络发送，经历过的四次数据拷贝是非常繁琐的，需要CPU上下文切换。

kafka将这个过程简化为两步了：

1、网络数据持久化到磁盘；（Producer 到 Broker）

2、磁盘文件通过网络发送（Broker到 Consumer）



结论：

https://www.jianshu.com/p/7863667d5fa7

1、顺序写，顺序读；

kafka将来自Prodcuer的数据，顺序写入 partition，partition就是一个文件，以此实现顺序写入；

Consumer从broker消费数据时，因为自带了偏移量，接着上次读取的位置继续读，以此实现顺序读。

2、零拷贝sendfile（in，out）；

数据直接在内核完成输入输出，不需要拷贝到用户空间再写出去；

3、mmap文件映射

简单描述其作用：将磁盘文件映射到内存，用户通过修改内存就能修改磁盘文件。它的工作原理是直接利用操作系统的Page来实现文件到物理内存的映射。完成映射之后你对物理内存的操作就会被同步到磁盘上。（操作系统在适当的时候）。



### Kafka的底层数据结构：日志文件与offset

可以认为写到kafka里的数据，一条一条的数据都是写到日志文件里。每条数据都有offset，代表了这条数据在日志文件里是第几条。

消费的时候也有一个offset的概念，意思是说某个消费者在这个日志文件里当前消费到了第几条消息。

### Kafka是如何通过设计消息格式节约磁盘空间占用开销的

crc32， magic，attribute，时间戳，key长度，key，value长度，value

Kafka是直接通过NIO的ByteBuffer以二进制的方式来保存消息的。

### 如何实现TB量级的数据再Kafka集群中分布式存储

每个Kafka应用进程 叫 Broker，多个Broker 就组成一个集群。

每个机器上都有partiton，是topic的一个数据分片，在底层就是一个日志文件，写数据的时候就可以把数据均匀的写到不同机器的不同partition上，这样的话，就可以通过多台机器来存储一个topic对应的大量数据。

### 如何基于多副本冗余机制保证Kafka宕机时还具备高可用性

多副本冗余机制，在partiiton里会选举出来一个leader，一个follower。

### 如何保证写入Kafka的数据不丢失呢？

如果写入的数据还没来得及向副本里同步的时候，所在的机器就宕机了。

然后副本被选举为leader；

然后这时来一个消费者消费数据，就消费不到之前写入的数据了。

怎么解决？引入ISR机制。in sync replica，就是跟leader partition保持同步的follower partition的数量。

只有处于ISR列表中的follower的才可以在leader宕机之后被选举为新的leader，因为这个ISR列表里代表他的数据跟leader是同步的。

Kafka会自动维护这个ISR列表，代表有哪个follower的数据跟leader是同步的。起码ISR列表里有一个，这时写数据才会写成功。`如果ISR列表里连一个follower也没有，这时就不允许写。`

这样的话，只要你写成功了一条数据，就保证数据不会丢了。

### Kafka集群处理请求的时候如何实现负载均衡的效果

整个Kafka的架构原理：

- Kafka的数据是分布式存储的，通过partition来把一个topic里的数据进行分布式存储；
- 写数据的时候，不停的把请求发送给各个Kafka Broker里的leader partition；
- 然后写完数据后就把leader partition数据同步给其它机器上的follower partition，做一个热备；
- 同时还自动维护一个ISR列表，会记录哪些follower partition与leader partition保持着同步。

#### 各个Kafka Broker如何实现负载均衡

Kafka会自动的把各个leader partition均匀的分布在各个Broker所在的机器上。

请求leader partition的时候，是均匀的发送请求给各个Broker。

### 基于Zookeeper实现Kafka无状态可伸缩的架构设计思路

将Kafka Broker相关的元数据信息，比如包含了哪些partition，哪些topic等，存入到Zookeeper集群。

### Partition的几个核心offset

#### LEO

每次leader接收到一条消息，都会更新自己的LEO，也就是log end offset。

接着各个follower会从leader请求同步数据。

offset = 0  ~~~ offset = 4 =》 LEO = 5，代表了最后一条数据后面的offset，也就是下一次将要写入数据的offset。

### 磁盘上的日志文件是按照什么策略定期清理腾出空间的？

大家可以想，不可能说每天涌入的数据都一直留在磁盘上，本质Kafka是一个流式数据的中间件，不需要跟离线存储系统一样保存全量的大数据，所以Kafka是会定期清理掉数据的，这里有几个清理策略。

Kafka默认保留最近7天的数据，每天都会把7天以前的数据给清理掉，包括.log，.index和.timeindex 这几个文件。log.retention.hours参数，可以设置数据要保留几天。

只要你的数据保留在Kafka里，就可以通过offset的指定，随时可以从kafka 搂出来几天之前的数据，进行数据回放。

比如下游的消费者消费了数据之后，数据丢失了，你需要从Kafka里搂出来3天前的数据，重新来回放处理一遍。

### Kafka是如何自定义TCP之上的通信协议以及使用长连接通信的？

Kafka的通信主要发生在 生产端和broker之间，broker和消费端之间，broker和broker之间，这些通信都是基于TCP协议进行连接和传输数据。

然后基于TCP之上的通信协议，是Kafka自定义的，是属于网络层里面的应用层的。

**所谓自定义的协议，说白了就是定义好互相之间发送请求和响应的数据格式，大家都按照这个格式来发送和响应数据。**

### Broker是如何基于Reactor模式进行多路复用请求处理的？

每个broker上都有一个 acceptor 线程和 很多个 processor 线程。可以用 num.network.threads 参数设置 processor 线程的数量，默认是3，client 跟一个broker 之间只会创建一个 socket 长连接，会复用。

Kafka 使用的是 Reactor 模式。特别适合应用与处理多个客户端并发向服务端发送请求的场景。

<img src="README.assets/Kafka Broker基于Reactor模式进行多路复用请求处理.png" alt="Kafka Broker基于Reactor模式进行多路复用请求处理" style="zoom:80%;" />



acceptor线程只用于请求分发，不涉及具体的逻辑处理，非常轻量级，因此具有很高的吞吐量表现。



### 如何对Kafka集群进行整体控制？Controller是个什么东西？

不知道大家有没有思考过一个问题，就是Kafka集群中某个broker宕机之后，是谁负责感知到他的宕机？ 

加一个Broker时是谁负责把集群里的数据进行负载均衡的迁移？创建topic时leader，follower怎么分配这些?

需要有一个总控组件来对集群进行整体的控制，这就是Controller。

所有的Broker会进行选举，然后某一个Broker会做为Controller的角色。

### 如何基于Zookeeper实现Controller的选举以及故障转移

在Kakfa集群启动的时候，会自动选举一台broker出来承担controller的责任，然后负责管理集群。

这个过程就是说每个broker都会尝试在zookeeper上创建一个/controller临时节点。

但是zookeeper会保证只有一个Broker会创建临节点成功，这个创建成功的broker就会选举为Controller。

一旦controller所在broker宕机了，这个临时节点就会消失，集群里的其它broker会一直监听这个临时节点，发现临时节点消失了，就再次争抢去创建临时节点，保证有一台新的broker会成为controller角色。

### 创建Topic时Kafka Controller是如何完成partition的Leader选举的呢

创建topic的时候，就会在zk里创建这个topic对应的节点。这个时候就会在zk里边写入，我创建的topic有几个partition，每个partition有几个副本，每个副本的状态。此时状态都是：NonExistentReplica。

Broker会监听zk里的topic变化的（topic的partition变化或者topic的数量变化），接着就会从zk中加载所有的kafka元数据信息到内存里，把这些partition副本的状态更改为：**NewReplica**。

比如说你创建一个topic，order_topic，设置3个partition，有2个副本，会写入到zk里去：

创建一个节点：

/topics/order_topic

partiton = 3， replica_factor = 2

[partition0_1,partition0_2]

[partition1_1,partition1_2]

[partition2_1,partition2_2]

Controller会感知到topic下面新增了一个子节点，就会把这些元数据加载到内存里边。

然后从每个partition的副本列表里取出来第一个做为leader，其他的就是follower，把这些给放到partition对应的ISR列表里去。

每个partition的副本在哪台机器上呢？kafka会做一个均匀的分配，把partition分散在各个机器上面，通过算法来保证，尽可能的把leader partition均匀分配在各个机器上，读写请求流都是打在leader partition上。

同时还会设置整个partition的状态：**OnlinePartition**。

接着Controller会把这个partition和副本所有的信息（包括谁是leader，谁是follower，ISR列表）都发送给所有broker，让他们知晓。

在kafka集群里，Controller负责集群的整体控制，但是每个broker都有一份元数据。

### 删除Topic时又是如何通过Kafka Controller控制数据清理？

如果要删除某个topic，则是如下的过程：

1. Controller会发送请求给这个Topic所有的partition所在的Broker机器，通知设置这个Topic的所有partition副本状态为：OfflineReplica，也就是让副本全部下线；
2. 接着Controller继续将这个Topic的所有partition副本状态修改为：ReplicaDeleteStarted；
3. 然后Controller还要给Broker发送请求，将各个partition副本的数据给删掉，就是对应磁盘上的那些文件，删除成功后，副本状态变为：ReplicaDeletionSuccessful，接着再变为 NonExistenReplica；
4. 最后设置这个Topic的各个分区状态为：Offline。

### Kafka Controller是如何基于ZK感知Broker的上线以及崩溃的？

1. Broker b在启动的时候会在zk里边生成一个临时节点，当Broker b宕机后，zk里的这个临时节点会消失；

2. Controller知道这个宕机的Broker上的leader partition有哪些，Controller会感知到这个临时节点消失了就会找到这个leader partition对应的follower partition所在的机器，然后从这些follower里边重新进行选举一个新的leader，然后更新元数据，并且通知所有的Broker，leader发生变更了。

如果集群中添加更多的Broker，Controller也能感知到。后边如果你给Topic设置更多的分区时，kafka会尽量分配给新的Broker。



kafka_2.11-1.0.0.tgz

集群新增Broker：

- Broker只要一启动，就会往zk注册节点；
- Controller就会感知到这个集群有新的Broker进来了；
- 然后呢会把当前集群有哪些Broker，这些Broker相关的元数据信息（比如在哪台机器上），广播和推送给集群内其他所有的Broker。



验证数据写到哪个机器的哪个分区上：

test-topic-0  => partition0的一个副本

test-topic-1  => partition1的一个副本



日志是分段存的，有索引文件。

### 通过监控解决线上集群的资源过载、负载不均、异常问题

- 监控资源的使用率

  - 当前集群保存了多大的数据量；

  - 每台机器的吞吐量：通过jmx来看，broker接收数据的速率，每秒接收多少条消息，接收多少字节的数据。

  - 如果扛不住这么多的数据了或并发，就需要进行broker机器的扩容，多增加一些broker机器。然后需要对所有的partition在扩容后的机器上进行负载均衡。

    

- 集群的负载倾斜

  如果某几台broker承载的数据量特别大，或者是承载的吞吐量特别大，此时你就应该手动进行一下数据迁移，可以把这些broker上的partition迁移一些到负载较轻的机器上去。

- 异常报错、JVM GC 频繁

  解决报错的问题，那是非常考验你的技术功底的，需要对Kafka源码有比较深入的研究。

  

### 如何对Kafka集群进行动态扩容加入更多的Broker节点

新增了一台broker节点，把配置文件什么的都修改好之后，直接使用命令启动。

然后Controller就会感知到，这时就会通知其它的Broker节点，有新的Broker加进来了。

Controller会把当前集群的元数据信息，比如当前有Broker节点的存在，有哪些topic，每个topic有哪些分区，有几个副本，ISR列表的信息同步给这个新的Broker。

### Kafka集群扩容之后如何迁移parititon保证集群负载均衡

这时如果要实现资源负载均衡，需要手动进行迁移，将原来Broker上的partition迁移一些到这个新的Broker上面。

vi topics-to-move.json

 

{“topics”: [{“topic”: “test01”}, {“topic”: “test02”}], “version”: 1} // 把你所有的topic都写在这里



bin/kafka-reassgin-partitions.sh --zookeeper hadoop03:2181,hadoop04:2181,hadoop05:2181 --topics-to-move-json-file topics-to-move.json --broker-list "5,6" --generate // 把你所有的包括新加入的broker机器都写在这里，就会说是把所有的partition均匀的分散在各个broker上，包括新进来的broker

 5,6 即为新增的broker的broker的id。

执行这个命令后， --generate 会生成一个迁移方案，可以保存到一个json文件里去，类似这样：

```json
bin/kafka-reassign-partitions.sh --zookeeper hadoop01:2181,hadoop02:2181,hadoop03:2181 --reassignment-json-file expand-cluster-reassignment.json --execute

bin/kafka-reassign-partitions.sh --zookeeper hadoop01:2181,hadoop02:2181,hadoop03:2181 --reassignment-json-file expand-cluster-reassignment.json --verify

```

然后再手动执行这个json文件里的这些指令。这个过程是非常耗费资源的，因为涉及到partition数据的迁移，很耗资源，涉及大量的网络带宽的传输，所以最好在网络低峰时做。

### Kafka Producer怎么把消息发送给Broker集群的？

producer需要指定把消息发送给哪个topic去。

producer.send(msg); // 用类似这样的方式去发送消息，就会把消息给你均匀的分不到各个分区上去；

producer.send(key,msg); // 用这种方式，比如订单id/用户id做key，会根据这个key的hash值去分发到某个分区上去，可以保证相同的key会路由分发到同一个分区上去。

生产者客户端起码要知道两个元数据：

- 每个topic有几个分区；
- 每个分区的leader是在哪台broker上。

然后跟那个机器上的Broker通过sokcet建立连接来进行通信，发送Kafka自定义的协议格式的请求过去，把消息带过去就行了。

这两个元数据从哪里来的呢？

**producer会自己从 broker 上拉取kafka集群的元数据，缓存在自己的client本地客户端上。**

### Kafka Producer发送消息的内部实现原理

每次发送消息，都会必须把数据封装成一个ProducerRecord对象，里面包含了要发送的topic，发送给哪个分区，分区key，消息内容，timestamp时间戳，然后将这个对象交给序列化器，变成自定义协议格式的数据。

接着把数据交给partitioner分区器，对这个数据选择合适的分区，默认就轮询所有分区，或者根据key来hash路由到某个分区，这个topic对应的分区消息，都会在客户端缓存的（当然提前回从broker中获取）。

接着会把这个数据发送到producer内部的一块缓冲区里。

然后producer内部有一个sender后台线程，会从缓冲区里提取消息，将这些数据封装成一个一个的batch，然后每个batch发送给分区的leader所在的broker机器。

<img src="README.assets/Kafka Producer发送消息的内部实现原理.png" alt="Kafka Producer发送消息的内部实现原理" style="zoom:80%;" />



### Kafka基于Consumer Group的消费者组

每个consumer都要属于一个consumer.group，就是一个消费组。

topic的一个分区只会分配给一个消费组内的一个consumer来消费消息。

每个consumer可能分配多个分区，也有可能某个consumer没有分配到任何分区。

分区内的消息都是由offset的，因此同一分区内的数据是可以保证顺序性的。

如果consumer group中的某个consumer挂了，则此时会自动把分配给他的分区交给其他的消费者；

如果他又重启了，那么又会把一些分区重新交给他，这个就是所谓的消费者 rebalance 的过程。

### Consumer Group Coordinator是什么以及主要负责什么

每个consumer group 都会选择一个broker 作为自己的coordinator，它是负责监控这个消费组里的各个消费者的心跳、以及判断是否宕机，然后开启rebalance，那么这个如何选择呢？

每个consumer都有自己的后台线程，不断发送心跳

coordinator 会尽可能均匀的分配分区给各个 consumer 来消费。

coordinator 能感知到 group 里的 consumer 发生了变化，会进行 rebalance。

### 为消费者选择 coordinator的算法是如何实现的

每个kafka consumer 不停的消费数据后，会维护一个自己当前消费每个分区数据消费到哪个offset了，然后会定期提交这个offset。新版本比如0.10，0.11以后，提交这个offset直接往kafka内部的一个topic: __consumer_offsets 去写。

那么写的时候，是怎么决定往 __consumer_offsets 的哪个分区（默认有50个分区）去提交的呢？

每个kafka consumer 都会拿自己的 group id进行hash，接着对__consumer_offsets的分区数据进行取模，就可以找到你的这个 consumer group 的offset 要提交到__consumer_offsets的哪个分区了。



你只要能找到你这个consumer group 是往 __consumer_offsets 的哪个分区去提交，你就可以找到这个分区对应的leader分区所在broker，这个broker就是这个 consumer group对应的 coordinator。

每个consumer内部维护着一个socket，会跟对应的coordinator（kafka broker） 建立一个连接，通信。

### Coordinator和Consumer leader如何协作制定分区方案

<img src="README.assets/Coordinator和Consumer Leader如何协作制定分区方案-3657349.png" alt="Coordinator和Consumer Leader如何协作制定分区方案" style="zoom:80%;" />



### rebalance的三种策略分别有哪些优劣势

三种rebalance策略：range、round-robin、sticky。

#### range

consumer1: partition0~2；

consumer2: partition3~5；

consumer3: partition6~8；

#### round-robin

consumer1: partition0，partition3，partition6；

consumer2: partition1，partition4，partition7；

consumer3: partition2，partition5，partition8；

#### sticky

最新的一个sticky策略，会尽可能的保障在rebalance的时候，让原本属于这个consumer的分区还属于他们，然后把多余的分区再均匀的分配过去，这样尽可能维持原来的分区分配的策略。

假如consumer3挂了，会这样分配：

consumer1: partition0~2 + partition6~7

consumer2: partition3~5 + partition8；

默认是 range 策略。基本上用的也都是range策略。

### 自动提交offset语义以及导致消息丢失和重复消费的问题

默认offset 是自动提交的。

auto.commit.inetrval.ms：5000，默认是5秒提交一次。

如果提交了消费到的offset之后，kafka broker就可以感知到了。比如你消费到了offset = 56987，下次你的consumer再次重启的时候，就会自动从 56987 这个位置往后去消费。

他的语义是一旦消息给你poll到了之后，这些消息就认为处理完了，后续就可以提交offset了。所以这里有两种问题：

#### 第一：消息丢失

如果你刚刚poll到消息，然后还没来得及处理消息，这时offset提交了，如果此时consumer宕机了，再次重启，则数据会丢失。因为上一次消费的那条数据其实你还没来得及处理，但kafka认为你已经处理了。

例如，本次poll到了一批数据，offset=65510~65532，刚好到了时间提交了offset，offset=65532 这个地方已经提交给kafka broker了，接着你准备对这批数据进行消费，结果不巧的是，刚要消费consumer宕机了。

其实你消费到的数据是没有处理的，但是消费offset已经提交给kafka了，consumer下次重启之后，会从offset=65533 这个位置开始消费，因此之前的那批数据就丢失了。

#### 第二：重复消费

如果你poll到了一批数据，offset=65510~65532，然后消费数据，并且都处理完了，但是offset还没来得及提交，consumer却宕机了，再次重启consumer，就会重新消费offset=65510~65532这一批消息，那么就会存在消息重复消费的问题。

### 同时可以接受有几个发送到Broker的请求没有响应

Sender线程 给broker 发送请求后后，不用等待这个broker有响应，就会接着发送下一批消息。

那么最多可以接受有几个请求没有响应呢？

max.in.flight.requests.per.connetction:5

这个参数默认值是5，默认情况下，**每个Broker最多只能有5个请求是发送出去但是还没接收到响应的**，**所以这种情况下是有可能导致顺序错乱的。**



比如，连续发送了5个请求，前两个请求都成功了，第三个请求失败了，第四第五的请求也成功了，然后把第三个请求又重试，重新发送了一遍，会导致第三个请求的数据在插入broker分区的时候排在第四第五条消息的后面。

所以如果要严格保证消息顺序，就需要将该参数值设置为1。也就是说给这个broker发送请求后，这个broker必须有响应回来后，才会继续向该broker发送下一条消息。

只有这样，才能保证，这个顺序完全是按照发送的顺序来走的。

### Kafka Broker内部有哪些延时的任务

#### 延时任务

比如 ack = -1，那么必须等待 leader和follower 都写完数据才能返回响应。而且有一个超时时间，默认是30秒，也就是 request.timeout.ms = 30。

那么在写入一条数据到磁盘之后，就必须有一个延时任务，到期时间是30秒。

- 数据写入leader所在分区的磁盘，如果所有follower都写入到本地磁盘了，那么就会自动触发苏醒，返回响应结果给客户端；

- 如果follower超过30秒还没有来拉取数据，那么就直接超时返回异常。

#### 延时拉取任务

这种延时拉取任务，就是说follower往leader拉取消息的时候，如果发现是空的，那么此时会创建一个延时拉取任务，然后延时时间到了之后，就会再次读取一次消息，如果过程中leader写入了消息，那么也会自动执行这个拉取任务。

### Kafka全链路数据丢失风险分析

#### 生产端丢数据的场景

1. producer生产的数据，还没有写入缓冲区，结果producer就挂了；

2. acks = 1（那么只要leader写入成功了，就可以认为是成功）时，但是如果数据刚刚写入leader虽然返回成功，但follower还没有同步数据呢，此时leader宕机了；

3. acks =  -1，leader和follower都写成功了才算是成功，此时任何一个副本数据丢失，都不会导致数据的丢失。但问题是如果min.insyc.replicas = 1，此时如果follower先宕机了，导致ISR列表里就一个leader了。

   但 acks =  -1，ISR里的副本都写入成功了，leader写入成功了就算成功了，结果此后leader也挂了，此时由于没有副本，数据还是会丢失的；

#### 消费端丢数据的场景

1. 消费到数据，还没来得及处理，刚好offset自动提交了，此时kafka就认为你已经成功处理了这批数据，但是此时 consumer宕机了。

### 为什么线上数据总是莫名奇妙出现数据重复

#### 生产端

生产端有重试机制，当出现一些网络抖动，在底层的网络环节，其实消息是发送出去了，结果收到了一个NetworkException，此时会重复发消息

#### 消费端

消费端重复消费的问题很频繁。如果你是用offset自动提交，那么在每次重启consumer服务的时候，一定会重复消费消息。为什么呢？

重启之前收到的一批消息，处理完毕了，但是offset还没来得及提交offset就重启了consumer，此时重启就会从上一次提交offset的地方重新拉取消息，导致消息重复。

