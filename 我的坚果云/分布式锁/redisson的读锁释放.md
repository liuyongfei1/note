### 场景

1. 假设现在有不同客户端加了读锁/同一个客户端同一个线程加了读锁：

```json
anyLock: {
	"mode" : "read",
	"UUID_01_thread_01" : 2,
	"UUID_02_thread_02" : 1,
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:2 1
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
{anyLock}:UUID_02:thread_02:rwlock_timeout:1 1
```

2. 同一个客户端的同一线程，先加写锁再加读锁

```
anyLock: {
	"mode" : "write",
	"UUID_01_thread_01:write" : 1,
	"UUID_01_thread_01" : 1,
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
```

现在想要释放锁，那释放锁是怎么处理的？

#### lua脚本

RedissonReadLock：

```lua
@Override
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    String timeoutPrefix = getReadWriteTimeoutNamePrefix(threadId);
    String keyPrefix = getKeyPrefix(threadId, timeoutPrefix);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
            "if (mode == false) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end; " +
            "local lockExists = redis.call('hexists', KEYS[1], ARGV[2]); " +
            "if (lockExists == 0) then " +
                "return nil;" +
            "end; " +
                
            "local counter = redis.call('hincrby', KEYS[1], ARGV[2], -1); " + 
            "if (counter == 0) then " +
                "redis.call('hdel', KEYS[1], ARGV[2]); " + 
            "end;" +
            "redis.call('del', KEYS[3] .. ':' .. (counter+1)); " +
            
            "if (redis.call('hlen', KEYS[1]) > 1) then " +
                "local maxRemainTime = -3; " + 
                "local keys = redis.call('hkeys', KEYS[1]); " + 
                "for n, key in ipairs(keys) do " + 
                    "counter = tonumber(redis.call('hget', KEYS[1], key)); " + 
                    "if type(counter) == 'number' then " + 
                        "for i=counter, 1, -1 do " + 
                            "local remainTime = redis.call('pttl', KEYS[4] .. ':' .. key .. ':rwlock_timeout:' .. i); " + 
                            "maxRemainTime = math.max(remainTime, maxRemainTime);" + 
                        "end; " + 
                    "end; " + 
                "end; " +
                        
                "if maxRemainTime > 0 then " +
                    "redis.call('pexpire', KEYS[1], maxRemainTime); " +
                    "return 0; " +
                "end;" + 
                    
                "if mode == 'write' then " + 
                    "return 0;" + 
                "end; " +
            "end; " +
                
            "redis.call('del', KEYS[1]); " +
            "redis.call('publish', KEYS[2], ARGV[1]); " +
            "return 1; ",
            Arrays.<Object>asList(getName(), getChannelName(), timeoutPrefix, keyPrefix), 
            LockPubSub.unlockMessage, getLockName(threadId));
}
```

#### lua脚本参数

- KEYS[1]：anyLock
- KEYS[2]：redisson_rwlock:{anyLock}
- KEYS[3]：{anyLock}:UUID_01:thread_01:rwlock_timeout
- KEYS[4]：{anyLock}
- ARGV[1]：0
- ARGV[2]：UUID_01:thread_01

#### lua脚本逻辑

```lua
local mode = redis.call('hget', KEYS[1], 'mode');
```

hget anyLock mode，返回：read。

```LUA
local lockExists = redis.call('hexists',KEYS[1], ARGV[2]);
```

hexists anyLock UUID_01:thread_01，返回1。

```lua
local counter = redis.call('hincrby', KEYS[1], ARGV[2]);
```

将这个客户端对应的加锁次数递减1，原来是2，递减1后，变为1，即 counter = 1。然后：

del {anyLock}:UUID_01:thread_01:rwlock_timeout:2 ，删除了一个 timeout key。现在的数据结构为：

```json
anyLock: {
	"mode" : "read",
	"UUID_01_thread_01" : 2,
	"UUID_02_thread_02" : 1,
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
{anyLock}:UUID_02:thread_02:rwlock_timeout:1 1
```

然后执行：

hlen anyLock > 1，就是hash里面的元素超过1个，这个判断条件成立，继续执行：

```lua
"local maxRemainTime = -3; " + 
"local keys = redis.call('hkeys', KEYS[1]); " + 
"for n, key in ipairs(keys) do " + 
                        "counter = tonumber(redis.call('hget', KEYS[1], key)); " + 
                        "if type(counter) == 'number' then " + 
                            "for i=counter, 1, -1 do " + 
                                "local remainTime = redis.call('pttl', KEYS[4] .. ':' .. key .. ':rwlock_timeout:' .. i); " + 
                                "maxRemainTime = math.max(remainTime, maxRemainTime);" + 
                            "end; " + 
                        "end; " + 
                    "end; "
```

取出现在anyLock 对应的hash结构里的所有key，并进行遍历。

counter 为每个key对应的值，并对 counter 进行递减遍历：

```lua
"local remainTime = redis.call('pttl', KEYS[4] .. ':' .. key .. ':rwlock_timeout:' .. i); " + 
"maxRemainTime = math.max(remainTime, maxRemainTime);" + 
```

pttl {anyLock}:UUID_01:thread_01:rwlock_timeout:1，获取timeout的剩余存活时间，

maxRemainTime 等于这些所有timeout里剩余存活时间最大的那个，然后，执行：

```
pexpire anyLock maxRemainTime
```

将 anyLock的剩余存活时间设置为之前计算的那个maxRemainTime。

本次释放读锁完成。

