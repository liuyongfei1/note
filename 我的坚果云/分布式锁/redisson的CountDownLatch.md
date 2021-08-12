## redisson的CountDownLatch

### 用法

```java
 RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
        latch.trySetCount(3);
        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]设置了必须有3个线程进行countDown，进入等待中");
        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]在做一些操作，请耐心等待");
                        Thread.sleep(3000);
                        RCountDownLatch localLatch = redisson.getCountDownLatch("anyCountDownLatch");
                        localLatch.countDown();
                        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]执行countDown操作");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }

        latch.wait();
        System.out.println(new Date() + ": 线程[" + Thread.currentThread().getName() + "]收到通知，有3个线程都执行了countDown操作，可以继续向下执行");
```

### trySetCount()方法

1. trySetCount(3) 方法时，

   ```
   set anyCountDownLatch 3
   ```

   会向redis中设置一个key， 存入一个值 3 。

### await()方法

```java
public void await() throws InterruptedException {
    RFuture<RedissonCountDownLatchEntry> future = subscribe();
    try {
        commandExecutor.syncSubscription(future);

        while (getCount() > 0) {
            // waiting for open state
            RedissonCountDownLatchEntry entry = getEntry();
            if (entry != null) {
                entry.getLatch().await();
            }
        }
    } finally {
        unsubscribe(future);
    }
}
```

这里其实会进入一个while 死循环里，不断的去执行：

```
get anyCountDownLatch
```

如果值大于0，则表明还没有到达指定数量的客户端执行countDown的操作，就继续陷入这个死循环。

### countDown()方法

```lua
@Override
public RFuture<Void> countDownAsync() {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                    "local v = redis.call('decr', KEYS[1]);" +
                    "if v <= 0 then redis.call('del', KEYS[1]) end;" +
                    "if v == 0 then redis.call('publish', KEYS[2], ARGV[1]) end;",
                Arrays.<Object>asList(getName(), getChannelName()), zeroCountMessage);
}
```

每执行一次countDown()方法，就会执行：

```
decr anyCountDownLatch
```

将 anyCountDownLatch 递减1，如果值等于0，则删除anyCountDownLatch：

```
del anyCountDownLatch
```

