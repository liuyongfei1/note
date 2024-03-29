## KafkaProducer初始化的时候会涉及到哪些核心的组件

### 一.客户端拉取元数据的网络通信大概过程

先看NetworkClinet类。 进行网络IO异步请求和响应，不是线程安全的。构造MetadataRequest

Sender是一个Runnable。

负责从缓冲区里获取消息，发送给broker。

Sender的run方法里，对应的NetworkClient里面有一个poll方法，poll方法里有一个MetadataUpdater组件去拉取元数据，拉取到元数据后，会更新元数据到MetaData里，然后用**notifyAll**方法唤醒主线程。

真正发送请求是调的NetworkClient组件的doSend()方法=>Selector组件（是Kafka中定义的专门用来处理网络通信的组件）。

首先确保该topic的元数据是可以使用的。

如果之前从来没有加载过topic的元数据，就会在这一步同步阻塞来等待可以去连接到broker上去拉取元数据过来，缓存到客户端。

maxBlockTimeMs参数，决定了你调用send()方法的时候，最多会被阻塞多长时间。



#### 客户端拉取Topic元数据是怎么按需加载的呢？

![客户端拉取Topic元数据是怎么按需加载的呢？](Producer-网络通信.assets/客户端拉取Topic元数据是怎么按需加载的呢？.png)



![Producer发送消息时元数据加载过程](Producer-网络通信.assets/Producer发送消息时元数据加载过程.png)



wait()方法释放锁，然后进入一个休眠等待再次被人唤醒后获取锁的状态；

此时如果有人获取锁之后，调用notifyAll()，就会把之前调用wait()方法进入休眠的线程给唤醒，让它再次尝试获取锁。

这里是一个线程同步的方法，MetaData这个类肯定是线程安全的



----------

### Kafka客户端设计是如何管理自己的内存的

往内存缓存里放入一条消息：

- 首先要找到这个分区对应的队列，这个队列核心是CopyOnWrite的数据结构；
- 如何基于内存里的数据结构构造一个缓冲区？

### 将消息放入内存缓冲

![Producer发送消息将消息放入内存缓冲](Producer-网络通信.assets/Producer发送消息将消息放入内存缓冲.png)

### RecordAccumulator

多线程如何并发安全的去创建一个分区对应的Queue？

见：RecordAccumulator.java

```java

public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        try {
            // check if we have an in-progress batch
            // 多个线程来获取或创建这个分区对应的queue，只有一个线程来创建，其它线程直接获取
            Deque<RecordBatch> dq = getOrCreateDeque(tp);
            // 对这个queue加锁，只有一个线程能进代码块
            synchronized (dq) {
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }
```

尝试将消息写入队列最近的一个batch中：

```java
private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Callback callback, Deque<RecordBatch> deque) {
    RecordBatch last = deque.peekLast();
    if (last != null) {
        FutureRecordMetadata future = last.tryAppend(timestamp, key, value, callback, time.milliseconds());
        if (future == null)
            last.close();
        else
            return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false);
    }
    return null;
}
```

第一次deque为空，直接返回null；

如果batch是存在的，就会将消息放入队列最近的一个batch中。

#### 将消息打包成batch

RecordAccumulator消息累加器，kafka发送消息时，并不是直接将消息从客户端通过网络发送给服务端，而是将消息存储在客户端的消息累加器中，当队列满了或者发送时间已到的时候才会去发送。

尽可能将消息打包成batch，减少对broker发送请求的次数。

如果这些请求是要发送到同一个分区，同一个分区对应同一个broker，则将这些请求打包成一个一个的batch。

一个broker上的多个分区对应的多个batch会被打包成一个request。

batch过小，会导致会频繁发送request；

batch过大，会在内存里缓存过多batch，浪费内存资源。

batch.size 默认16K。

默认情况下，如果仅仅是考虑batch机制的话，那么必须要等到足够多的消息打包成一个batch，才能通过request发送到broker上去。

但是有一个问题，如果你发送了一条消息，但是很久都没有达到一个batch大小，所以说要设置一个linger.ms，如果在指定时间范围内，都没凑出来一个batch，那么就必须立即把这个消息发送出去。

#### 基于NIO ByteBuffer为batch分配内存空间

RecordAccumulator.java的append()方法：

```java
// we don't have an in-progress record batch try to allocate a new batch
int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
```

一个batch对应了一块内存空间，要放一堆消息。

batchSize默认大小是16KB，如果你的消息大于了16KB，就会使用你的消息的大小来开辟内存空间；

如果你的消息小于16KB，则就使用16KB来开辟内存空间。

BufferPool.java的allocate()方法：

```java
// now check if the request is immediately satisfiable with the
// memory on hand or if we need to block
int freeListSize = this.free.size() * this.poolableSize;
```

Deque里的ByteBuffer的数量*16kb = 这时已经缓存的内存空间大小。

availableMemory 剩余的可利用的空间大小。

#### 3个线程一块过来时的场景分析

假设有3个线程，分别为：线程1，线程2，线程3。这三个线程都会获取到1个16kb的ByteBuffer内存。

假设线程2进入了synchronized代码块里面去，基于16kb的ByteBuffer构造一个batch，放入Dequeue里去了。

接着线程3进入sychronized代码块里面去，直接把消息放入Dequeue里已经存在的batch里，那么他手上的一个16kb的ByteBuffer怎么办？

在这里，会将这个16kb的ByteBuffer给放入到BytePool的池子里去，这样就可以保证内存可以复用。

![Producer发送消息将消息放入内存缓冲-double check](Producer-网络通信.assets/Producer发送消息将消息放入内存缓冲-double check.png)



其中一个线程构造了一个batch放到Dequeue里去，另外两个线程通过double check的机制就可以直接往batch里写了。那么此时这两个线程申请的ByteBuffer就没用了，就会还到BufferPool里去。

avaliableMemory = 32mb - 16KB * 3。

### 二. Kafka生产端唯一的一个IO线程到底在干什么

既然有消息在不断的往内存缓冲里写，那必须得有从内存缓存里拿消息然后往broker发送的处理机制，负责干这个事儿的就是Kafka的这个IO线程。

在创建KafkaProducer的时候初始化的：

```java
new KafkaProducer<String, String>(props);
```

#### 2.1 KafkaProducer初始化时核心源码

KafkaProducer.java：

```java
private KafkaProducer(ProducerConfig config, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
    try {
        log.trace("Starting the Kafka producer");
        Map<String, Object> userProvidedConfigs = config.originals();
        ......
        // 处理网络通信的：发送请求和接收响应  
        NetworkClient client = new NetworkClient(
                    new Selector(config.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), this.metrics, time, "producer", channelBuilder),
                    this.metadata,
                    clientId,
                    config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION),
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                    this.requestTimeoutMs,
                    time,
                    true);
            this.sender = new Sender(client,
                    this.metadata,
                    this.accumulator,
                    config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION) == 1,
                    config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                    (short) parseAcks(config.getString(ProducerConfig.ACKS_CONFIG)),
                    config.getInt(ProducerConfig.RETRIES_CONFIG),
                    this.metrics,
                    Time.SYSTEM,
                    this.requestTimeoutMs);  
        String ioThreadName = "kafka-producer-network-thread" + (clientId.length() > 0 ? " | " + clientId : "");
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
  
```

这个io线程里封装了Sender的逻辑，sender里又封装了NetworkClient。

NetworkClient是处理网络通信的：发送请求和接收响应；

IO线程一启动，Sender里面的业务逻辑就会执行。



这里先默认走到这里时，已经拉取到了topic对应的元数据信息缓存到了客户端：

topic->partition->leader/follower + isr。

#### 2.2 Sender的run()方法

1. 获取已经可以发送消息的那些partition，需要满足以下两个条件：
   - 那些partition有已经写满的batch(16kb)；
   - batch创建的时间到现在已经超过了linger.ms；
2. 如果有些topic对应的元数据没有拉取到，则标记一下，后边要重新拉取元数据；
3. 检查一下是否准备好向哪些broker发送数据了，如果还没有跟那些broker建立连接，必须在这里建立连接，基于底层的NIO来建立TCP连接；
4. 你有很多partition可以发送数据，此时检查一下有哪些partition的leader是在同一个broker上，则按broker对partition分组，找到一个broker对应的多个batch。如果一个batch在内存缓冲中停留的时间超过了60s，则超时不要；
5. 将一个broker对应的多个leader partition对应的batch组合成一个ClientRequest，形成一个请求，发送给broker；
6. 通过NetworkClient走底层的网络通信，把每个broker的ClientRequest给发送过去就可以了。poll()方法是负责实际的进行网络IO通信操作的一个核心方法，处理实际的发送请求和接收响应。

#### 2.3 如何判断内存缓存中的一个batch是可以发送的

如果有batch是可以发送的，则会找到这个batch对应的partition的 leader副本所在 broker，并将这个broker放入set集合中去。

#### 2.4 如何检查筛选出来的目标Broker可以发送数据过去

Sender.java：

```java
void run(long now) {
    Cluster cluster = metadata.fetch();
    // get the list of partitions with data ready to send
    // 这里边封装了哪些broker可以发送数据过去
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
  
  
    // remove any nodes we aren't ready to send to
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
      Node node = iter.next();
      // 判断这个broker是否可以发送数据过去
      if (!this.client.ready(node, now)) {
        iter.remove();
        notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
      }
    }
```

NetworkClient.java

```java
/**
 * Begin connecting to the given node, return true if we are already connected and ready to send to that node.
 *
 * @param node The node to check
 * @param now The current timestamp
 * @return True if we are ready to send to the given node
 */
@Override
public boolean ready(Node node, long now) {
    if (node.isEmpty())
        throw new IllegalArgumentException("Cannot connect to empty node " + node);

    if (isReady(node, now))
        return true;

    if (connectionStates.canConnect(node.idString(), now))
        // if we are interested in sending to a node and we don't have a connection to it, initiate one
        initiateConnect(node, now);

    return false;
}

/**
     * Check if the node with the given id is ready to send more requests.
     *
     * @param node The node
     * @param now The current time in ms
     * @return true if the node is ready
     */
    @Override
    public boolean isReady(Node node, long now) {
        // if we need to update our metadata now declare all requests unready to make metadata requests first
        // priority
        return !metadataUpdater.isUpdateDue(now) && canSendRequest(node.idString());
    }
```

##### 第一条线： !metadataUpdater.isUpdateDue(now)

```java
@Override
public boolean isUpdateDue(long now) {
    return !this.metadataFetchInProgress && this.metadata.timeToNextUpdate(now) == 0;
}
```

当前不是处于元数据加载的过程，而且下一次更新元数据的间隔时间为0。意思就是现在没有加载元数据，但是马上就应该要加载元数据了。

如果对上述判断条件取非的话，意思就是 要么正在加载元数据，要么还没到加载元数据的时候。

总结：**这个判断的作用是 如果当前必须要更新元数据了，则不能发送请求，必须要等待这个元数据更新了再次发送请求。**

##### 第二条线：canSendRequest(node.idString())

```java
/**
 * Are we connected and ready and able to send more requests to the given connection?
 *
 * @param node The node
 */
private boolean canSendRequest(String node) {
    return connectionStates.isReady(node) && selector.isChannelReady(node) && inFlightRequests.canSendMore(node);
}
```

1> 有一个broker连接状态的缓存，先检查一下这个broker是否已经建立过连接了，只要建立过连接，才能继续判断其它的条件；

2>selector可以理解为底层封装的就是NIO的Selector，一个Selector可以注册很多Channel，每个Channel就代表了与一个broker建立的连接；

3>inFlightRequests，有个参数可以设置，默认是对同一个broker同一时间最多容忍5个请求发送过去没有响应，多于5个都没有收到响应，则就不能继续发送了。

必须得同时满足这三个条件，才可以认为这个broker可以继续发送数据。、

#### 2.5 如果跟Broker之间没有建立连接，如何检查是否可以建立连接

NetworkClient.java

```java
/**
 * Begin connecting to the given node, return true if we are already connected and ready to send to that node.
 *
 * @param node The node to check
 * @param now The current timestamp
 * @return True if we are ready to send to the given node
 */
@Override
public boolean ready(Node node, long now) {
    if (node.isEmpty())
        throw new IllegalArgumentException("Cannot connect to empty node " + node);

    if (isReady(node, now))
        return true;
		// 主要是这个方法：判断能否建立连接
    if (connectionStates.canConnect(node.idString(), now))
        // if we are interested in sending to a node and we don't have a connection to it, initiate one
        initiateConnect(node, now);

    return false;
}
```

判断能否跟这个broker建立连接的思路：

1. 先找到这个broker的一个连接状态；
2. 如果这个连接状态是null，则表明这个broker从来没有建立过连接，直接返回ture，可以建立连接；
3. 如果这个连接状态已经存在，但当前broker的状态是断开连接，且上一次跟这个broker尝试连接的时间到现在已经超过了重试时间了（默认为100ms），则也可以建立连接。

#### 2.6 深入网络通信的起点：通过哪个核心组件与broker建立连接

NetworkClient.java

ready() -> initiateConnect(node, now) ：

```java
/**
 * Initiate a connection to the given node
 */
private void initiateConnect(Node node, long now) {
    String nodeConnectionId = node.idString();
    try {
        log.debug("Initiating connection to node {} at {}:{}.", node.id(), node.host(), node.port());
        this.connectionStates.connecting(nodeConnectionId, now);
        // 底层建立socket连接
        // 发送缓冲区大小(128KB)，接收缓冲区大小(32KB)。
        selector.connect(nodeConnectionId,
                         new InetSocketAddress(node.host(), node.port()),
                         this.socketSendBuffer,
                         this.socketReceiveBuffer);
    } catch (IOException e) {
        /* attempt failed, we'll try again after the backoff */
        connectionStates.disconnected(nodeConnectionId, now);
        /* maybe the problem is our metadata, update it */
        metadataUpdater.requestUpdate();
        log.debug("Error connecting to node {} at {}:{}:", node.id(), node.host(), node.port(), e);
    }
}
```

可以看到，**就是通过Selector组件与broker建立的socket连接**。

Selector类：

```java
A nioSelector interface for doing non-blocking multi-connection network I/O.
```

针对多个Broker的网络连接，执行非阻塞的IO操作。

发送请求和收取响应都是通过socket读取的。比较核心的两个参数就是socketSendBuffer和socketReceiveBuffer，socket的发送缓冲区和接收缓冲区。在工业级的网络通信开发里面，socketSendBuffer和socketReceiveBuffer 这连个核心参数都是必须设置的。

<img src="Producer-网络通信.assets/NetworkClient.png" alt="NetworkClient" style="zoom:80%;" />

复习一下NIO的课程：

- NIO建立socket连接，其实就是在底层初始化一个SocketChannel，发起一个连接请求；

- 然后就会把这个SocketChannel给注册到Selector上面，让Selector去监听这个连接事件；

- 如果broker返回响应说可以建立连接，Selector就会告诉你，你就可以通过API调用来完成底层的网络连接（TCP三次握手）双方都会有一个Socket（操作系统级别的概念，Socket代表了网络通信终端）。

  ```java
  SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_CONNECT);
  ```

​      发起连接之后，直接把这个SocketChannel给注册到Selector上去了，让Selector监视这个SocketChannel的OP_CONNECT事件，是否有人同意跟他建立连接。

​     会获取到一个Selectionkey，大致上可以理解为与SocketChannel是一一对应的。

#### 2.7 KafkaChannel是如何对原生的Java NIO的SocketChannel进行封装的？

broker id 对应一个网络连接，一个网络连接对应一个KafkaChannel，底层对应的是SocketChannel，SocketChannel对应的是最底层的网络通信层面的一个Socket，Socket通信其实就是基于TCP协议建立的连接，一个端口对应另一个端口，两者进行通信。

#### 2.8 Kafka封装的Selector是如何与broker建立连接的？

NetworkClient.java的initiateConnect() -> selector.connect()

Selector的connect()方法：

```java
public void connect(String id, InetSocketAddress address, int sendBufferSize, int receiveBufferSize) throws IOException {
    if (this.channels.containsKey(id))
        throw new IllegalStateException("There is already a connection for id " + id);

    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.configureBlocking(false);
    Socket socket = socketChannel.socket();
    socket.setKeepAlive(true);
    if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socket.setSendBufferSize(sendBufferSize);
    if (receiveBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socket.setReceiveBufferSize(receiveBufferSize);
    socket.setTcpNoDelay(true);
    boolean connected;
    try {
        connected = socketChannel.connect(address);
    } catch (UnresolvedAddressException e) {
        socketChannel.close();
        throw new IOException("Can't resolve address: " + address, e);
    } catch (IOException e) {
        socketChannel.close();
        throw e;
    }
    SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_CONNECT);
    KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize);
    key.attach(channel);
    this.channels.put(id, channel);

    if (connected) {
        // OP_CONNECT won't trigger for immediately connected channels
        log.debug("Immediately connected to node {}", channel.id());
        immediatelyConnectedKeys.add(key);
        key.interestOps(0);
    }
}
```

看到这里的代码是非常熟悉的，就是基于底层的Java NIO来做的。非常关键的参数：socket发送缓冲区和接收缓冲区，分别是128kb和32kb。

### Kafka Producer怎么把消息发送给Broker集群的

topic对应的数据是拆分成多个分区分布在不同的机器上的，那么producer在发送消息的时候怎么知道该把数据发送给哪个分区的呢？

1. 选择把消息发送到哪个topic去；
2. Kafka有一个提供给客户端的Partitioner组件，它的作用是生产者把消息交给Partitioner组件，然后Partitioner组件会决定把消息发送给哪个分区；
3. 默认不做任何设置直接发消息的话，会采用轮询的方式，将消息负载均衡的发给各个分区，比如：producer->send(msg)；
4. 如果发消息的时候指定分区key，那么就会根据这个key的hash值来分发到指定的分区，**这样就可以让相同的key分发到同一个分区里去**。比如用订单id或者用户id做key，这样的话同一个订单id或用户id产生的数据都会路由分发到同一个分区上去，比如：producer->send(orderId, msg)。

5. 知道要将消息发送给哪个分区之后，就可以找到这个partition的leader所在的broker，然后建立Socket连接跟那台broker建立连接，发送消息了。

Producer（生产者客户端），起码要知道两个元数据：

- 每个topic有几个分区；
- 每个分区的leader是在哪台broker上。

这两个元数据是怎么拿到的呢？

 Producer会自己从broker上拉取kafka集群的元数据，缓存在自己的client本地客户端上。

#### Partitioner组件将消息路由到分区里

##### 不指定分区key将消息负载均衡分到到各分区

取模（取余），用一个counter递增，然后对分区数进行取模。

##### 指定分区key将消息负载均衡分到到各分区

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



#### Selector类的connect()方法

本地用两个 KafkaProducer 服务试一下 多个jvm进程时的 并发问题。

#### 针对每个目标Broker构建一个很多Batch组成的Request

发送出去的请求，需要按照Kafka的二进制协议来定制数据的格式。

需要包含对应的请求头，api key，api version，acks，request timeout，接着才是请求体，里面包含了对应的多个batch数据，最后，再把数据给转化为二进制的字节数组。

ClientRequest里面就是封装了按照二进制协议格式的数据，发送到broker上去，有很多个topic，每个topic有很多个partition，每个partition对应一个batch的数据，

要发送请求的时候，会把这个请求暂存到KafkaChannel里去，同时让Selector监听他的OPT_WRITE事件，增加一种OPT_WRITE事件，同时保留了OPT_READ事件，此时Selector会同时监听这个连接的OPT_WRITE和OPT_READ事件。

发送完了请求之后，就会把OPT_WRITE事件取消监听，只保留关注OPT_READ事件。

发现对这个broker可以再次执行OPT_WRITE事件，然后再次调用SocketChannel的write方法，把ByteBuffer里剩余的数据继续往Broker去写。	

上述这个过程会重复多次，直到把这个请求对应的数据发送完毕。

#### 看看Kafka生产端的NIO编程是如何进行粘包类问题的处理的？

粘包也就是说收到了多余的数据。

怎么解决？

每个响应中间插入一个特殊的分隔符。比如在响应消息前面插入4个字节（integer类型）代表响应消息本身的数据大小的数字。

比如：

消息1 是350个字节，消息2是 212个字节。则发送过来的数据可能是这样：

350 消息1 212消息2。

分为三个阶段：

1. 先读取前四个字节，转换为一个int，获取到消息的大小；
2. 根据大小申请内存buffer；
3. 最后读取指定大小的数据到申请好的buffer。

由此，就完成了一整条数据的正式读取。

#### 看看Kafka生产端的NIO编程是如何进行拆包类问题的处理的？

拆包，也就是说收到的数据不是一条完整的数据。那么该如何组成一条完整的数据呢？

假如有这样的多条消息：

199 消息 256消息 369消息

##### 情况一

**在读取消息的时候，4个字节的size都没读完，比如就只能读到19。**

position = 0，limit = 4

现在读取一个字节，position=1；读取二个字节，position = 2，此时remaining 是2，还剩下2个字节是可以读的。	

如果说一个请求对应的ByteBuffer中的二进制字节数据一次write没有全部发送完毕，此时remainging > 0，就不会取消对OT_WRITE事件的监听。

下次请求调用poll方法，会再次运行到这里来：

Selector类的pollSelectionKeys()方法：

```java
while(networkReceive = channel.read() != null)
    addToStagedReceives(channel, networkReceive);
```



KafkaChannel类：

```java
public NetworkReceive read() throws IOException {
    NetworkReceive result = null;

    if (receive == null) {
        receive = new NetworkReceive(maxReceiveSize, id);
    }

    receive(receive);
    if (receive.complete()) {
        receive.payload().rewind();
        result = receive;
        receive = null;
    }
    return result;
}
```

由于上一次没有读完消息，因此**NetworkReceive**还是停留在那里，可以继续读取：

```java
private long receive(NetworkReceive receive) throws IOException {
    return receive.readFrom(transportLayer);
}
```

因为剩余还有2个字节，所以这次最多就只能读取2个字节到里面去，这样，4个字节的size就凑满了，此时就说明size数字是可以读取出来了。解决了size的拆包问题。

##### 情况二

**比如199个字节的消息只读取到了 172个字节，这种拆包问题怎么处理？**

下一次继续循环，执行到代码NetworkReceive类的readFromReadableChannel()方法：

```java
if (buffer != null) {
    int bytesRead = channel.read(buffer);
    if (bytesRead < 0)
        throw new EOFException();
    read += bytesRead;
}
```

buffer不为null，剩余还有27个字节，则就读取27个字节。就把剩余的数据给读出来了。



只要size或buffer没读完，**NetworkReceive**就一直放在Channel里，并且保持对OP_READ事件的监听。

```java
public NetworkReceive read() throws IOException {
    NetworkReceive result = null;

    if (receive == null) {
        receive = new NetworkReceive(maxReceiveSize, id);
    }

    receive(receive);
    // 通过判断size或buffer都没有remaining了来判断是否读完了
    if (receive.complete()) {
        receive.payload().rewind();
        result = receive;
        // 然后就将receive置为null了
        receive = null;
    }
    return result;
}
```

NetworkReceive为null了，下次就可以读取一条新的数据了。



可以看这篇文章：[java nio解决拆包粘包问题](https://www.jianshu.com/p/5c13ed1c709c?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

里边有demo代码。

---



##### Selector类的addToCompletedReceives()方法

每次循环只拿出一个客户端的一个请求放到completedReceives里去。

KSelector对于某个客户端，StagedReceives里可能有多个请求，但是completedReceives里只有一个请求。而且这个请求已经被放到RequestQueue里去了，取消了OP_WRITE事件关注，而且对于这个客户端后面不会再读取新的请求出来了。

下一次循环，只有关注了OP_WRITE事件，才会把StagedReceives里的请求放到completedReceives里去。

所以，同一时间，一个客户端只可能有一个Request在这个RequestQueue里，必须得等到这个请求处理完毕后，重新关注OP_WRITE，才能处理这个客户端的下一个请求。

#### Kafka Broker网络通信模型

![Kafka Broker网络通信模型](Producer-写磁盘文件.assets/Kafka Broker网络通信模型-4912055.png)

##### Reactor模式

IO多路复用，由一个线程来监听多路连接，同步等待一个或多个IO事件的到来，然后把事件交给对应的Handler线程来处理，这就叫Reactor模式。

基本上，只要底层的高性能网络通信就离不开Reactor模式。像Netty、Redis都是使用Reactor模式。

网络通信模型的发展如下：

单线程=》多线程=》线程池=》Reactor模型

##### Kafka所采用的的Reactor模型

<img src="Producer-写磁盘文件.assets/image-20211020233810655.png" alt="image-20211020233810655" style="zoom:50%;" />

##### Kafka网络通信模型总结

1. Broker中有一个Acceptor（mainReactor）监听新连接的到来，与新连接建立之后，轮询选择一个Processor（subReactor）管理这个连接；
2. 而Processor或监听其管理的连接，当事件到达之后，读取数据封装成Request，并将Request放入共享请求队列中；
3. 然后IO线程池不断的从该队列中取出请求，执行真正的处理。处理完之后将响应发送给Processor对应的响应队列中去，然后由Processor将Response返回给客户端。

##### 生产者-消费者模式

每个listener只有一个Acceptor线程，因为它只是作为新建连接再分发，没有过多的逻辑，很轻量，一个足矣。

Processor线程在Kafka中称之为网络线程，默认的网络线程有3个，对应的参数是num.network.threads。可以根据实际业务动态增减。

还有个IO线程池，即KafkaRequestHandlerPool，执行真正的处理，对应的参数是：num.io.threads，默认是8个。IO线程处理完之后，将Response放入对应的Processor，由Processor将响应返回给客户端。

可以看到网络线程和IO线程之间利用经典的生产者-消费者模式，不论是用于处理Request的请求队列，还是IO处理完返回的Response。

这样的好处是什么？生产者和消费者之间解耦了，可以对生产者和消费者做独立的变更或扩展，并且可以平衡两者的处理能力，例如消费不过来了，就多加一些IO线程。

如果你看过其它中间件源码，你会发现生产者-消费者模式真的是太常见了。 	

写的比较好的一篇文章：https://www.jianshu.com/p/04bae18f6b9b

### 为什么能保障高并发呢

其实主要得益于Reactor这套网络通信模型。

大量客户端疯狂的来写数据，连接就建立好，分配给默认3个Processor线程，每个线程的Selector来监听，只要监听到请求，就快速的把请求放入队列中去。这样就能保证三个线程就能非常快速的接收请求。

然后后边也是一个线程池多个线程从队列中取数据，然后再往本地磁盘写，Handler线程写磁盘文件时都是顺序写，然后写到内存里。1个线程把1个请求的数据写入多个分区对应的磁盘文件里去，都是顺序写，写内存。

1个线程处理1个写入操作只需1毫秒（因此是顺序写，且写内存，可能连1毫秒都用不了），那么1秒钟就可以处理1千个请求，8个线程1秒钟就可以处理8000个请求。

所有1秒钟处理8千个或1万个请求，几万条或上十万条数据都是没问题的。

这套机制保障有大量请求过来时，高并发也是可以扛得住的。

