## redisson的可重入锁学习之watchdog

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

要搞明白这个原因，需要去继续看一下CommandAsyncService类里的evalAsync方法，与redis进行通信这是比较偏底层了，我这里就不深入分析了。

#### 定时调度的任务干了什么事儿

