## Redisson的Seamphore

### 简单实用

```java
Config config = new Config();

// 3主3从的一个redis集群
config.useClusterServers().addNodeAddress("redis://192.168.10.111:6379")
		    .addNodeAddress("redis://192.168.10.111:6380")
		    .addNodeAddress("redis://192.168.10.111:6381")
		    .addNodeAddress("redis://192.168.10.112:6379")
		    .addNodeAddress("redis://192.168.10.112:6380")
		    .addNodeAddress("redis://192.168.10.112:6381");

RedissonClient redisson = Redisson.create(config);

final RSemaphore semaphore = redisson.getSemaphore("semaphore");
        semaphore.trySetPermits(3);

        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]尝试获取semaphore锁");
                        semaphore.acquire();
                        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]成功获取到了semaphore"
                                + "锁");
                        Thread.sleep(3000);
                        semaphore.release();
                        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]释放了semaphore锁");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
```

### 源码分析

RedissonSeamphore类：

```lua
@Override
    public boolean trySetPermits(int permits) {
        return get(trySetPermitsAsync(permits));
    }
    
    @Override
    public RFuture<Boolean> trySetPermitsAsync(int permits) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "local value = redis.call('get', KEYS[1]); " +
                "if (value == false or value == 0) then "
                    + "redis.call('set', KEYS[1], ARGV[1]); "
                    + "redis.call('publish', KEYS[2], ARGV[1]); "
                    + "return 1;"
                + "end;"
                + "return 0;",
                Arrays.<Object>asList(getName(), getChannelName()), permits);
    }
```

get semaphore  不存在，返回0

set semaphore 3 将这个信号量同时能够允许获取锁的客户端的数量设置为3

### acquire()方法

```java
@Override
public boolean tryAcquire(int permits) {
    return get(tryAcquireAsync(permits));
}

@Override
    public RFuture<Boolean> tryAcquireAsync(int permits) {
        if (permits < 0) {
            throw new IllegalArgumentException("Permits amount can't be negative");
        }
        if (permits == 0) {
            return RedissonPromise.newSucceededFuture(true);
        }

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                  "local value = redis.call('get', KEYS[1]); " +
                  "if (value ~= false and tonumber(value) >= tonumber(ARGV[1])) then " +
                      "local val = redis.call('decrby', KEYS[1], ARGV[1]); " +
                      "return 1; " +
                  "end; " +
                  "return 0;",
                  Collections.<Object>singletonList(getName()), permits);
    }
```

get seamphore  = 1

1 >= 1

val = decrby seamphore 1 将信号量允许获取锁的客户端的数量递减1，变成了2

返回1。

来一个客户端，申请加锁，val就会减掉1

当 seamphore 变为0时，get seamphore 为0，if判断条件不成立，直接返回0，进入while true死循环：

```java
try {
            while (true) {
                if (tryAcquire(permits)) {
                    return;
                }

                getEntry().getLatch().acquire(permits);
            }
        } finally {
            unsubscribe(future);
        }
```

不断的尝试去获取这个 seamphore锁，直到有其它客户端释放锁，才能获取到这个锁。

### release()方法

每个客户端释放锁的时候，release()方法：

```java
@Override
    public RFuture<Void> releaseAsync(int permits) {
        if (permits < 0) {
            throw new IllegalArgumentException("Permits amount can't be negative");
        }
        if (permits == 0) {
            return RedissonPromise.newSucceededFuture(null);
        }

        return commandExecutor.evalWriteAsync(getName(), StringCodec.INSTANCE, RedisCommands.EVAL_VOID,
            "local value = redis.call('incrby', KEYS[1], ARGV[1]); " +
            "redis.call('publish', KEYS[2], value); ",
            Arrays.<Object>asList(getName(), getChannelName()), permits);
    }
```

会将 seamphore 加1

incrby seamphore 1，这样就会退出while true死循环，其它客户端就可以尝试获取这个锁了。

获取到之后再将 seamphore 递减1。