## redisson的公平锁源码学习之排队加锁

### 可重入非公平锁

前面讲到的是非公平可重入锁。所谓的非公平可重入锁是什么意思呢？多个胡乱的争抢，根本没有任何公平性和顺序性而言。

### 可重入公平锁

使用公平锁，可以保证说：客户端获取锁的顺序就跟他们请求获取锁的顺序是一致的。

会把各个客户端对加锁的请求进行排队处理，保证说先申请加锁的，就可以先得到这把锁，实现所谓的公平性。

可重入非公平锁、公平锁，他们在整体的技术实现上都是一样的，只不过唯一不同的点就是在加锁逻辑那。

### 源码学习

RedissonFairLock是RedissonLock的子类，整体锁的技术框架实现都是跟之前讲解的RedissonLock是一样的，无非就是重载了一些方法，加锁和释放锁的lua脚本逻辑比之前复杂了。

直接看RedissonFairLock类的tryLockInnerAsync方法的这段lua脚本：

```
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    // ......
    
}
```

<img src="redisson的公平锁源码学习之排队加锁.assets/可重入公平锁lua加锁脚本.png" alt="可重入公平锁lua加锁脚本" style="zoom:80%;" />

#### lua脚本所需参数

```java
Arrays.<Object>asList(getName(), threadsQueueName, timeoutSetName), 
            internalLockLeaseTime, getLockName(threadId), currentTime + threadWaitTime, currentTime);
```

对应lua脚本中：

- KEYS[1]：getName()，就是锁的名字，比如：testLock；
- KEYS[2]：threadsQueueName（在redissonFairLock类构造函数中定义的），它是基于redis的数据结构实现的一个队列。比如：redisson_lock_queue:{testLock}；
- KEYS[3]：timeoutSetName（在redissonFairLock类构造函数中定义的），它是基于redis的数据结构实现的一个有序的Set数据集合。比如：redisson_lock_timeout:{testLock}；
- ARGV[1]：internalLockLeaseTime（在redissonLock类构造函数中定义的），值为 30000毫秒；
- ARGV[2]：getLockName(threadId)，查看代码可以知道值为：UUID:threaId；
- ARGV[3]：currentTime + threadWaitTime，查看代码可以知道值为：当前时间+5000毫秒。

在搞清楚脚本所需的每个参数代表什么意义后，**我们以客户端的当前线程在10点整使用"testLock"这个key首次尝试加锁为例**，来分析这段lua脚本。

#### 第一部分

首先是一个while true的死循环，执行：

```bash
lindex redisson_lock_queue:{testLock} 0
```

从redisson_lock_queue:{testLock} 这个队列中弹出第一个元素，由于是首次加锁，队列是空的，所以这里什么都获取不到，firstThreadId2等于false，直接退出死循环。

#### 第二部分

这个大判断条件包含三个小的判断条件：

```lua
if (redis.call('exists', KEYS[1]) == 0) and ((redis.call('exists', KEYS[2]) == 0) 
    or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then
```

第一个小条件必须成立，然后第二个和第三个小条件只要其中一个成立，则这个大判断条件就成立了。

执行：

```
exists testLock
exists redisson_lock_queue:{testLock}
lindex redisson_lock_queue:{testLock} 0
```

- 由于是第一次加锁，肯定不存在testLock 这个key，因此第一个小判断条件成立。
- 由于是第一次加锁，队列还不存在，所以这个条件肯定也是成立的；
- 或者说队列存在，且弹出的第一个元素等于当前的UUID:threaId，也就是说队列中的第一个排队的就是当前线程。

因此这个大判断条件成立，则执行：

