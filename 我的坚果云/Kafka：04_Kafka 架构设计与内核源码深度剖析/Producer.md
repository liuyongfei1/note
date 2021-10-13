## KafkaProducer初始化的时候会涉及到哪些核心的组件

### RecordAccumulator：将消息打包成batch

RecordAccumulator消息累加器，kafka发送消息时，并不是直接将消息从客户端通过网络发送给服务端，而是将消息存储在客户端的消息累加器中，当队列满了或者发送时间已到的时候才会去发送。

尽可能将消息打包成batch，减少对broker发送请求的次数。

如果这些请求是要发送到同一个分区，同一个分区对应同一个broker，则将这些请求打包成一个一个的batch。

一个broker上的多个分区对应的多个batch会被打包成一个request。

batch过小，会导致会频繁发送request；

batch过大，会在内存里缓存过多batch，浪费内存资源。

batch.size 默认16K。

默认情况下，如果仅仅是考虑batch机制的话，那么必须要等到足够多的消息打包成一个batch，才能通过request发送到broker上去。

但是有一个问题，如果你发送了一条消息，但是很久都没有达到一个batch大小，所以说要设置一个linger.ms，如果在指定时间范围内，都没凑出来一个batch，那么就必须立即把这个消息发送出去。

#### 拉取元数据的网络通信大概过程

先看NetworkClinet类。 进行网络IO异步请求和响应，不是线程安全的。构造MetadataRequest

Sender是一个Runnable。

负责从缓冲区里获取消息，发送给broker。

Sender的run方法里，对应的NetworkClient里面有一个poll方法，poll方法里有一个MetadataUpdater组件去拉取元数据，拉取到元数据后，会更新元数据到MetaData里，然后用notifyAll方法唤醒主线程。

真正发送请求是调的NetworkClient组件的doSend()方法=>Selector组件（是Kafka中定义的专门用来处理网络通信的组件）。

首先确保该topic的元数据是可以使用的。

如果之前从来没有加载过topic的元数据，就会在这一步同步阻塞来等待可以去连接到broker上去拉取元数据过来，缓存到客户端。

maxBlockTimeMs参数，决定了你调用send()方法的时候，最多会被阻塞多长时间。



### Partitioner组件将消息路由到分区里

#### 不指定分区key将消息负载均衡分到到各分区

取模（取余），用一个counter递增，然后对分区数进行取模。

#### 指定分区key将消息负载均衡分到到各分区

会通过封装的一个工具类murmur2，实现一个算法。将一个字节数组转换为一个int类型的hash值，因此，只要分区key是一样的，比如说是同样的订单id，此时就一定会生成相同的hash值，那么用hash值对分区数量进行取模，就可以保证路由到的分区也是相同的。

### 1.将消息写入内存缓冲的运行流程值得学习的地方



#### 1.如何基于CopyOnWriteMap实现线程安全的分区队列构建



### 不断申请内存空间的情况下导致可用内存耗尽了怎么办

如果申请不到新的内存空间，就会阻塞；

如果有batch底层对应的空间腾出来，就会唤醒阻塞的线程，再次申请一个新的ByteBuffer来构造一个Batch；

如果等一段时间还是没有可用的内存空间，就会报timeout超时。



### 总结： RecordAccumulator 这个组件的内存缓存机制

主要是  RecordAccumulator 这个组件的内存缓存机制，一个分区的消息才能缓冲到一个Deque对应的batch里去：

1. 不断往里面写消息，像每个分区写消息时多线程并发；
2. 从BufferPool里拿ByteBuffer出来复用，然后拿ByteBuffer去构建 batch，后面其它线程都往batch里去写；
3. 写满一个batch，就会再拿一个ByteBuffer再去构建一个batch，然后往里边写；
4. 各个分区都是这个流程；
5. 如果batch占的ByteBuffer的内存空间超过了可用的内存空间，其它线程就会卡在这里，阻塞；
6. 如果有空间腾出来，就会唤醒阻塞的线程，再次申请一个新的ByteBuffer来构建一个batch，然后线程可以继续往里写消息了，否则，就会抛出超时的错误。

基于缓冲池，复用内存，避免不断的申请内存、垃圾回收。



#### Kafka生产端唯一的一个IO线程到底在干什么？

就是Kafka生产端的一个IO线程，里面封装了 sender 逻辑 =》 NetworkClient 发送网络请求。

每个Broker上有多个partition，把同一个broker上的leader partition聚合分组打包为batch，然后把这些leader partition所对应的batch，放在一个ClientRequest里一次性发送给broker。

然后通过NetWorkClient走底层的网络通信，把每个broker的clientRequest给发送过去。

NetWorkClient的poll方法是负责进行网络IO通信操作的一个核心方法，负责发送数据出去和读取响应。



元数据加载的请求是如何通过网络通信发送出去的？

元数据加载的响应是如何来处理的？



