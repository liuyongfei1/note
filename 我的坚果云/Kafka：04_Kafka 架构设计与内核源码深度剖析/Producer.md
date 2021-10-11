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

