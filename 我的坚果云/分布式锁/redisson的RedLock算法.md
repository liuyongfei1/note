## Redisson的RedLock

### RedLock算法原理总结

假设有一个redis cluster，有3个redis master实例。

然后执行如下步骤，获取一把分布式锁：

1. 获取当前时间戳，时间是毫秒；
2. 轮流尝试在每个master实例上进行加锁；
3. 尝试在大多数master实例上进行加锁，比如一共是3个节点，则需要加锁的数量是2个节点：n / 2 +1；
4. 客户端建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了；
5. 只要锁建立失败，就会依次删除所有已经创建的锁；
6. 只要一个客户端创建了一把分布式锁，其他客户端就得不断轮询去尝试获取锁。

### 源码分析

RedissonRedLock锁的实现非常简单，是RedissonMutilLock的一个子类。主要是通过方法的重载，改变了MutilLock中的几个特殊的行为。

```java
public RedissonRedLock(RLock... locks) {
    super(locks);
}

@Override
protected int failedLocksLimit() {
    return locks.size() - minLocksAmount(locks);
}

protected int minLocksAmount(final List<RLock> locks) {
    return locks.size()/2 + 1;
}

@Override
protected long calcLockWaitTime(long remainTime) {
    return Math.max(remainTime / locks.size(), 1);
}
```

假如有个redis cluster，3个 master/salve实例：

- minLocksAmount：给大多数节点加锁的算法，最少的加锁数量： 3 / 2 +1 = 2
- failedLocksLimit：允许加锁失败的数量：3 - minLocksAmount = 3 -2 = 1
- calcLockWaitTime：加锁等待时间 ，就是说对每个lock进行加锁的时候，有一个尝试获取锁的超时时间，原来的是默认的 4500毫秒，4500 / 3 = 1500毫秒，则每个小lock获取锁的超时时间就是 1500毫秒。



#### 总结

1. 之前的MultiLock，只要有一个小lock加锁失败，则已经加的锁都要被释放，此时这个MultiLock就会标记为失败。

2. 但是现在使用RedLock的话，可以容忍failedLocksLimit个加锁失败的（如果小锁的数量是3，则可以容忍1个加锁失败的）；

​       如果第二个lock又加锁失败，则failedLocksLimit-- 就等于0，那么就会标记为加锁失败，RedLock加锁失败。

3. 也就是说针对多个lock进行加锁，每个lock都有一个 1500毫秒的加锁超时时间，如果在4500毫秒内，成功的对 n/2 +1 个lock加锁成功了，就算这个RedLock加锁成功了。

### 读写锁

多个客户端同时加读锁，是不会互斥的。多个客户端可以同时加这个读锁。

加读锁的lua脚本逻辑

.. 就是做字符串的拼接