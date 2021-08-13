## zookeeper的非重入锁源码学习

### 简单实用

```java
InterProcessSemaphoreMutex lock = new InterProcessSemaphoreMutex(client, "/locks/lock_01");
lock.acquire();
Thread.sleep(3000);
lock.release();
```

就是基于semaphore实现的，只不过将semaphore允许获取锁的客户端的数量设置为了1，同一时间只能有一个客户端获取到一把锁。

InterProcessSemaphoreMutex类：

```java
/**
 * @param client the client
 * @param path path for the lock
 */
public InterProcessSemaphoreMutex(CuratorFramework client, String path)
{
    this.semaphore = new InterProcessSemaphoreV2(client, path, 1);
}
```

