### Consumer端主要知识点

有个consumer-group的概念。

#### 1.消费过程

1. 每个consumer初始化启动完毕，都会发送请求到某一个broker的consumer group coordinator上，加入group；
2. 如果coordinator确认一个consumer group组建完毕，此时就会选择一个consumer leader出来；
3. consumer leader接到通知说自己是leader了，接着就会制定一份分区消费的方案（每个分区只会分配给group内的一个consumer）但一个consumer可以消费多个分区，返回给coordinator，下发给所有的consumer知道；
4. 然后大家就开始消费消息了，单线程，一次拉取一批数据过来，进行消费。

#### 2.offset作用

默认是后台自动提交offset，写入内存__consumer_offsets 这个topic里去。作用是知道每个consumer group对一个kafka的各个分区都消费到哪里了。

当你的一个consumer停机了，重启后，此时就会接着上一次提交的offset继续进行消费。

#### 3.rebalance

- 如果consumer挂掉，coordinator感知到了之后，就会rebalance，把那个挂掉的consumer消费的分区分配给其它的consumer。

- 如果新增consumer，此时会考虑将已有consumer的分区转移给新增的consumer来进行消费。新的consumer直接会接着那个分区被提交过的offset继续进行消费。