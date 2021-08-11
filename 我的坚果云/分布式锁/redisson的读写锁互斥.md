## redisson的读写锁互斥

### 读写锁互斥

#### 先读锁后写锁如何互斥

客户端A和客户端B 加了读锁，此时的数据如下：

```
anyLock: {
	"mode": "read",
	"UUID_01:thread_01": 1,
	"UUID_02:thread_02": 1
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
{anyLock}:UUID_02:thread_02:rwlock_timeout:1 1
```

此时客户端C（UUID_03:thread_03）来加写锁。

#### lua脚本

RedissonWriteLock类：

```java
@Override
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                        "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                        "if (mode == false) then " +
                              "redis.call('hset', KEYS[1], 'mode', 'write'); " +
                              "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                          "end; " +
                          "if (mode == 'write') then " +
                              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                                  "local currentExpire = redis.call('pttl', KEYS[1]); " +
                                  "redis.call('pexpire', KEYS[1], currentExpire + ARGV[1]); " +
                                  "return nil; " +
                              "end; " +
                            "end;" +
                            "return redis.call('pttl', KEYS[1]);",
                    Arrays.<Object>asList(getName()), 
                    internalLockLeaseTime, getLockName(threadId));
}
```

##### 参数

- KEYS[1]：anyLock
- ARGV[1]：30000毫秒
- ARGV[2]：UUID_03_thread_03:write

##### 解释

执行

```
hget anyLock mode
```

由于之前已经加过读锁了，因此不会返回false，也等于write，因此这两个if判断都不成立，直接走：

```lua
return redis.call('pttl', KEYS[1]);
```

返回 anyLock这个锁key的剩余生存时间。

会导致客户端C加锁失败，进入while true死循环，除非原先加读锁的人释放了读锁，他这个锁才能够重新加上去。

#### 2、如果先有写锁再有人加读锁是如何互斥的

假设客户端A加了一个写锁：

```
anyLock: {
	"mode": "write",
	"UUID_01:thread_01:write": 1
}
```

假设此时客户单B来进行加读锁，参数为：

- KEYS[1]：anyLock
- KEYS[2]：{anyLock}:UUID_02:thread_02:rwlock:timeout
- ARGV[1]：30000毫秒
- ARGV[2]：UUID_02_thread_02
- ARGV[3]：UUID_02_thread_02:write

##### lua脚本解释

RedissonReadLock类的tryLockInnerAsync方法：

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

```
hget anyLock mode
```

因为此时客户端A已经加了一个写锁，所以此时mode等于write，进入第二个if判断：

```lua
if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then 
```

执行

```
hexists anyLock UUID_02_thread_02:write
```

由于**之前加写锁的是客户端A，不是客户端B**，所以这里不存在，返回false，然后直接返回：

```
pttl anyLock
```

的剩余生存时间，导致加锁失败，然后进入死循环不断的重试。因此，读锁的写锁是互斥的。

备注：

**如果是客户端A加写锁，然后客户端B再加读锁是可以的。**

