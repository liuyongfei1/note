## redisson的写锁互斥

### 客户端A加写锁

假设客户端A加了一个写锁：

```
anyLock: {
	"mode": "write",
	"UUID_01:thread_01:write": 1
}

```

### 客户端B也来尝试加写锁

RedissonWriteLock类的tryLockInnerAsync方法：

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

#### 参数

- KEYS[1]： anyLock
- ARGV[1]： 30000毫秒
- ARGV[2]：UUID_02:thread_02:write

#### 脚本解释

由于客户端A已经加了写锁，因此执行：

```
hget anyLock mode
```

会返回 write，

```lua
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
```

执行：

```
hexists anyLock UUID_02:thread_02:write
```

由于此前是客户端A加的写锁，因此这里返回false，if判断条件不成立。

```lua
return redis.call('pttl', KEYS[1]);
```

直接返回 anyLock 的剩余生存时间，导致加写锁失败。

#### 结论

- 不同的客户端之间加写锁，是会互斥的；
- 不同的客户端/线程之间，读锁与读锁不互斥，读锁与写锁互斥，写锁与写锁互斥。

