## zookeeper的Semaphore实现原理

### Semaphore总结

可以指定同时有多个线程获取到锁。

### Semaphore简单使用

```java
InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, "/semaphores/semaphore_01", 3);
Lease lease = semaphore.acquire();
Thread.sleep(3000);
semaphore.returnLease(lease);
```

#### 先创建一个普通的lock锁

点击InterProcessSemaphoreV2()构造方法，查看

```java
private InterProcessSemaphoreV2(CuratorFramework client, String path, int maxLeases, SharedCountReader count)
{
    this.client = client;
    // 创建一个普通的锁
    lock = new InterProcessMutex(client, ZKPaths.makePath(path, LOCK_PARENT));
    this.maxLeases = (count != null) ? count.getCount() : maxLeases;
    leasesPath = ZKPaths.makePath(path, LEASE_PARENT);

    if ( count != null )
    { ......}
```

客户端如果尝试获取semaphore锁，会自动创建一个自动生成的锁，path是：/semaphores/semaphore_01/locks。

这时，其它的客户端如果再来尝试获取Semaphore锁，会生成临时顺序节点，按照这个顺序排队。

#### acquire()方法

##### 获取lease锁

```java
private InternalAcquireResult internalAcquire1Lease(ImmutableList.Builder<Lease> builder, long startMs, boolean hasWait, long waitMs) throws Exception {
  ......
    PathAndBytesable<String> createBuilder = client.create().creatingParentsIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL);
            String path = (nodeData != null) ? createBuilder.forPath(ZKPaths.makePath(leasesPath, LEASE_BASE_NAME), nodeData) : createBuilder.forPath(ZKPaths.makePath(leasesPath, LEASE_BASE_NAME));
            String nodeName = ZKPaths.getNodeFromPath(path);
            // 在创建Lease的时候，实现了一个Lease的close方法
            builder.add(makeLease(path));
}
```

这里会创建一个临时顺序节点，比如：

/semaphores/semaphore_01/leases/_c_a72fd130-0e97-427d-8cd9-d4619513064b-lease-0000000000

创建了一个lease，其实就代表了这个客户端已经成功的获取到了一个 semaphore 的锁。

比如，semaphore 一共允许最多有3个客户端可以加锁。

在创建Lease的时候，实现了一个Lease的close方法：

```java
private Lease makeLease(final String path)
{
    return new Lease()
    {
        @Override
        public void close() throws IOException
        {
            try
            {
                client.delete().guaranteed().forPath(path);
            }
            catch ( KeeperException.NoNodeException e )
            {
                log.warn("Lease already released", e);
            }
            catch ( Exception e )
            {
                throw new IOException(e);
            }
        }

        @Override
        public byte[] getData() throws Exception
        {
            return client.getData().forPath(path);
        }
    };
}
```



##### 释放semaphore锁	

```java
semaphore.returnLease(lease);
```

在释放semaphore锁的时候，会调Lease的close()方法。

```java
client.delete().guaranteed().forPath(path);
```

path:

/semaphores/semaphore_01/leases/_c_c3a48fc5-846f-4091-b751-4249bf9de0bb-lease-0000000003

删除这个临时顺序节点，同时会触发客户端D的一个监听器。

##### 使用监听器

InterProcessSemaphoreV2类的 356行：

```java
                for(;;)
                {
                    // 使用了一个监听器watcher
                    List<String> children = client.getChildren().usingWatcher(watcher).forPath(leasesPath);
                    if ( !children.contains(nodeName) )
                    {
                        log.error("Sequential path not found: " + path);
                        return InternalAcquireResult.RETRY_DUE_TO_MISSING_NODE;
                    }

                    if ( children.size() <= maxLeases )
                    {
                        break;
                    }
                    if ( hasWait )
                    {
                        long thisWaitMs = getThisWaitMs(startMs, waitMs);
                        if ( thisWaitMs <= 0 )
                        {
                            return InternalAcquireResult.RETURN_NULL;
                        }
                        wait(thisWaitMs);
                    }
                    else
                    {
                        // 当前线程被阻塞
                        wait();
                    }
                }


```

监听器的逻辑：

```java
private final Watcher watcher = new Watcher()
    {
        @Override
        public void process(WatchedEvent event)
        {
            notifyFromWatcher();
        }
    };

private synchronized void notifyFromWatcher()
    {
        notifyAll();
    }
```



##### 子节点变化的监听器逻辑

监听器会执行Object.notifyAll()方法唤醒阻塞的当前进程。

那么之前线程是在哪被阻塞的呢？=》wait()方法（可以详见上面注释）

唤醒阻塞的当前进程，则会继续会到for死循环中，再次去判断leases目录下的节点数量是否<=3。

如果 <= 3，则客户端D就可以加semaphore锁了。

如果 > 3，则就处于阻塞状态（wait），继续等待。

semaphore锁的实现机制，内部借助了一个普通的锁，保证所有的客户端都在保障顺序去获取semaphore锁了。

##### 释放locks锁

由于已经成功的获取到了一个 semaphore的锁，这时就会释放掉之前 自动加的那个lock锁（/semaphores/semaphore_01/locks）。

这时其它客户端就会来尝试加锁。

