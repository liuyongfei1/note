## redisson的可重入锁源码学习之watchdog

### 问题

假如你的某个客户端上锁了之后，过了5分钟，10分钟，都没释放掉这个锁，那么你觉得会怎么样呢？

锁对应的key刚开始的生存周期其实就是30秒而已，难道是默认情况下30秒后这个锁就自动释放？

肯定不是这样的，如果分布式锁是这样设计的话，那肯定是没办法用的。

### watchdog的原理

判断一下客户端是否还持有这把锁，如果还持有这把锁的话，会去判断该锁的可存活时间，如果是快要过期的话，就会去执行一段脚本，给这个锁进行一个延期。

后台有一个定时调度的一个watchlog任务，只要你的这个lock没有被释放掉，这个任务就会每隔10秒钟去延长一下这个锁的生存周期，重新延长为30秒。

因此，客户端就可以长期的持有这个key对应的分布式锁。

### 加锁成功后，后台是如何每隔10秒钟去延长生存时间的

#### lua脚本执行完触发监听器

接着上一篇，在执行lua脚本加锁成功后，会返回一个RFture：

```java
private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, final long threadId) {
    if (leaseTime != -1) {
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    // 执行lua脚本，进行加锁，返回锁的剩余生存时间ttlRemainingFuture
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    ttlRemainingFuture.addListener(new FutureListener<Long>() {
        @Override
        public void operationComplete(Future<Long> future) throws Exception {
            if (!future.isSuccess()) {
                return;
            }

            Long ttlRemaining = future.getNow();
            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        }
    });
    return ttlRemainingFuture;
}
```

只要lua脚本执行完成，返回ttlRemainingFuture后，这个RFture的监听器就会被触发执行：

- 如果lua脚本执行失败，future.isSuccess()是false，则直接返回，什么也不做；
- 如果lua脚本执行成功，且ttlRemaining等于null，说明锁已经获取到了（源码已经给出注释 // lock acquired），这时会开启一个定时调度的任务。

**备注：**

可能看到这里，会存在一个疑问：

ttlRemaining不应该是锁的剩余存活时间吗，为什么ttlRemaining等于null就说明锁已经获取到了？

要搞明白这个原因，需要去继续看一下CommandAsyncService类里的evalAsync方法:

```java
private <T, R> RFuture<R> evalAsync(NodeSource nodeSource, boolean readOnlyMode, Codec codec, RedisCommand<T> evalCommandType, String script, List<Object> keys, Object... params) {
    RPromise<R> mainPromise = createPromise();
    List<Object> args = new ArrayList<Object>(2 + keys.size() + params.length);
    args.add(script);
    args.add(keys.size());
    args.addAll(keys);
    args.addAll(Arrays.asList(params));
    async(readOnlyMode, nodeSource, codec, evalCommandType, args.toArray(), mainPromise, 0, false, null);
    return mainPromise;
}
```

async()方法会与redis进行通信，这是比较偏底层了，我这里就不深入分析了。

#### 定时调度的任务干了什么事儿

```java
private void scheduleExpirationRenewal(final long threadId) {
        if (expirationRenewalMap.containsKey(getEntryName())) {
            return;
        }

        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                
                // 延长过期时间,下边会详细介绍
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                
                future.addListener(new FutureListener<Boolean>() {
                    @Override
                    public void operationComplete(Future<Boolean> future) throws Exception {
                        expirationRenewalMap.remove(getEntryName());
                        if (!future.isSuccess()) {
                            log.error("Can't update lock " + getName() + " expiration", future.cause());
                            return;
                        }
                        
                        if (future.getNow()) {
                            // reschedule itself
                            scheduleExpirationRenewal(threadId);
                        }
                    }
                });
            }

        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);

        if (expirationRenewalMap.putIfAbsent(getEntryName(), new ExpirationEntry(threadId, task)) != null) {
            task.cancel();
        }
    }
```

上一篇我们已经介绍过internalLockLeaseTime的值为30000毫秒，也就是说在执行lua脚本成功后，10秒后开始执行这个定时任务，且以后会每隔10秒去执行一次。

##### 延长存活时间

我们来看renewExpirationAsync方法：

```java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                "return 1; " +
            "end; " +
            "return 0;",
        Collections.<Object>singletonList(getName()), 
        internalLockLeaseTime, getLockName(threadId));
}
```

该方法会执行一段lua脚本，解释一下这段lua脚本干了什么事儿：

- ARGV[2]就是一个由UUID和线程id拼接的字符串，执行

  ```bash
  hexists testLock jk6b27a7-5346-483a-b9b5-0957c690c27f:1
  ```

  判断是否存在jk6b27a7-5346-483a-b9b5-0957c690c27f:1，如果还存在**则说明这个客户端的这个线程当前还持有这把锁**；

- 则延长存活时间，重新设置存活时间为30秒。

##### 递归执行延长存活时间的方法

在scheduleExpirationRenewal方法里，有这样的代码：

```java
if (future.getNow()) {
                            // reschedule itself
                            scheduleExpirationRenewal(threadId);
                        }
```

因此只要这个客户端当前还持有这把锁，**它会递归执行scheduleExpirationRenewal方法，就会延长锁的存活时间。**

否则，就会停止定时调度任务。

