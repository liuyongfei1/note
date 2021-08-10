## redisson的MultiLock源码学习

### Redisson的MultiLock简单实用

```java
RLock lock1 = redisson.getLock("locak1");
RLock lock2 = redisson.getLock("locak2");
RLock lock3 = redisson.getLock("locak3");

RedissonMultiLock lock = new RedissonMultiLock(lock1,lock2,lock3);
lock.lock();;
lock.unlock();
```

### 总结

默认情况下，你包裹了几把锁，则baseWaitTime = 锁的数量 * 1500毫秒。

获取所有的锁必须在这个时间内结束：

- 如果成功获取到锁，则会启动一个lock watchdog不断的刷新你的锁key的生存时间为3000毫秒。
- 如果超时，则会释放掉所有已经获取到的锁，重新走while true死循环重新尝试获取锁，使用的是各个锁的tryLock方法。