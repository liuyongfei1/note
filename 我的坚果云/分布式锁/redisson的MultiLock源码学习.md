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

