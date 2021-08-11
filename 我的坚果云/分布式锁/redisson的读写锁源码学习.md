## redisson的读写锁源码学习

### 读写锁

- 多个客户端同时加读锁，是不会互斥的；
- 如果有人加了读锁，则这时就不能加写锁了，任何人都不能加写锁，读锁和写锁时互斥的；
- 如果有人加了写锁，此时任何人都不能再加写锁了，写锁和写锁也是互斥的。

#### 客户端A（UUID_01:thread_01）来加读锁

```java
@Override
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                            "if (mode == false) then " +
                              "redis.call('hset', KEYS[1], 'mode', 'read'); " +
                              "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                              "redis.call('set', KEYS[2] .. ':1', 1); " +
                              "redis.call('pexpire', KEYS[2] .. ':1', ARGV[1]); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                            "end; " +
                            "if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then " +
                              "local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                              "local key = KEYS[2] .. ':' .. ind;" +
                              "redis.call('set', key, 1); " +
                              "redis.call('pexpire', key, ARGV[1]); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                            "end;" +
                            "return redis.call('pttl', KEYS[1]);",
                    Arrays.<Object>asList(getName(), getReadWriteTimeoutNamePrefix(threadId)), 
                    internalLockLeaseTime, getLockName(threadId), getWriteLockName(threadId));
}
```

##### 参数

- KEYS[1]：anyLock
- KEYS[2]：{anyLock}:UUID_01:threadId_01:rwlock_timeout
- ARGV[1]：30 * 1000毫秒
- ARGV[2]：UUID_01:threadId_01
- ARGV[3]：UUID_01:threadId_01:write

lua脚本的执行这里不再详细解释。客户端A加锁成功后，会有这样的两个数据：

```
anyLock: {
	"mode": "read",
	"UUID_01:threadId_01": 1
}
{anyLock}:UUID_01:threadId_01:rwlock_timeout:1 1
```

同时，会开启一个lock watchdog，每隔10秒去判断当前客户端是否还持有这个锁，如果持有这个锁，则重新设置存活时间为30秒，保持redis的锁key和java代码中的持有的锁是同步的。

#### watchdog逻辑

主要看lua脚本：

RedssionRedLock类里的renewExpirationAsync方法：

```java
@Override
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    String timeoutPrefix = getReadWriteTimeoutNamePrefix(threadId);
    String keyPrefix = getKeyPrefix(threadId, timeoutPrefix);
    
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "local counter = redis.call('hget', KEYS[1], ARGV[2]); " +
            "if (counter ~= false) then " +
                "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                
                "if (redis.call('hlen', KEYS[1]) > 1) then " +
                    "local keys = redis.call('hkeys', KEYS[1]); " + 
                    "for n, key in ipairs(keys) do " + 
                        "counter = tonumber(redis.call('hget', KEYS[1], key)); " + 
                        "if type(counter) == 'number' then " + 
                            "for i=counter, 1, -1 do " + 
                                "redis.call('pexpire', KEYS[2] .. ':' .. key .. ':rwlock_timeout:' .. i, ARGV[1]); " + 
                            "end; " + 
                        "end; " + 
                    "end; " +
                "end; " +
                
                "return 1; " +
            "end; " +
            "return 0;",
        Arrays.<Object>asList(getName(), keyPrefix), 
        internalLockLeaseTime, getLockName(threadId));
```

参数：

- KEYS[1]：anyLock
- KEYS[2]：{anyLock}
- ARGV[1]：30000毫秒
- ARGV[2]：UUID_01:threadId_01

##### lua脚本解释

1. 判断当前线程是否已经加过锁了，由于客户端A已经成功的加了读锁，因此这里返回true；
2. pexpire anyLock 30000，刷新一下anyLock锁key的生存时间为30000毫秒；
3. hlen anyLock >1，就是判断说anyLock的hash 数据结构内的key-value队是否超过1个，这个条件肯定是成立的；
4. local keys = redis.call('hkeys', KEYS[1]);  取出键值对的所有的key，并遍历：
   - hget anyLock key，拿到每个key对应的值，比如：hget anyLock UUID_01:threadId_01 的值是1，赋给counter（就是加锁次数）；
   - 如果值是数字，则对counter进行递减循环；
   - pexpire {anyLock}:UUID_01:threadId_01:rwlock_timeout:10 30000，给这个设置存活时间为30秒；
   - pexpire {anyLock}:UUID_01:threadId_01:rwlock_timeout:9 30000，给这个设置存活时间为30秒；
   - .....

##### 总结

watchdog主要干了两件事儿：

- 判断当前线程，如果还持有这把锁，则刷新生存时间为30秒；
- 同时还会遍历加锁次数，对那个锁key的每次加锁对应的一个rwlock_timeout也重新设置存活时间为30秒。