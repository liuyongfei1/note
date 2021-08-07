在分布式系统开发当中，分布式锁的使用场景还是很常见的。

### Redis哨兵模式

![Redis哨兵](Redis分布式锁.assets/Redis哨兵.png)

#### Redis哨兵的作用

- 通过发送命令，让Redis服务器返回其监控的运行状态，包括主服务器和从服务器；
- 当哨兵监测到master宕机，会自动将slave切换为master；
- 然后通过发布订阅模式通知其它的从服务器，修改配置文件，让它们切换主机。

### Redis分布式锁

#### 什么是分布式锁

在分布式系统里面，如果多个机器上的服务要同时对一个共享资源进行操作，比如修改数据库里的一份数据，此时的话，某台机器就需要先获取一个针对那个资源（比如数据库里的某一条数据）的分布式锁。

获取到了分布式锁之后，就可以任由你去查询那条数据，修改那条数据，或者做其它的什么操作。在这个期间，没有任何其它的客户端可以来修改这条数据。

获取了一个分布式操作之后，就对某个共享的数据获取了一定时间范围内的独享操作。

#### 源码分析

使用redisson客户端来进行redis分布式锁的源码分析。

```java
Config config = new Config();

// 这里本地没有搭建redis集群
config.useClusterServers().addNodeAddress("localhost:6379");

RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("anyLock");
lock.lock();
```

##### getLock()

1. 这里getLock方法获取到的是RedissonLock对象，它里面封装了一个commandExecutor，可以执行一些redis的底层命令，比如set，get的一些操作；

2. 还有一个internalLockLeaseTime，是跟watch dog有关的，默认是30秒。

   

#### 可重入锁

如果是在一个客户端的一个线程内，先对一个Lock进行了加锁，然后后面又加了一次锁，这样就形成了一个叫做可重入锁的概念。

就是同一个线程对一个lock可以反复的重复加锁多次，每次加锁和一次释放锁必须是配对的。

