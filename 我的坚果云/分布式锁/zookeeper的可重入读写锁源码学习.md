## zookeeper的可重入读写锁源码学习

### 简单使用

```java
InterProcessReadWriteLock lock = new InterProcessReadWriteLock(client, "/locks/lock_01");
lock.readLock().acquire();
Thread.sleep(3000);
lock.readLock().release();

lock.writeLock().acquire();
lock.writeLock().release();
```

### 读锁+读锁

```java
/**
 * @param client the client
 * @param basePath path to use for locking
 */
public InterProcessReadWriteLock(CuratorFramework client, String basePath)
{
    writeMutex = new InternalInterProcessMutex
    (
        client,
        basePath,
        WRITE_LOCK_NAME,
        1,
        new SortingLockInternalsDriver()
        {
            @Override
            public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
            {
                return super.getsTheLock(client, children, sequenceNodeName, maxLeases);
            }
        }
    );

    readMutex = new InternalInterProcessMutex
    (
        client,
        basePath,
        READ_LOCK_NAME,
        Integer.MAX_VALUE,
        new SortingLockInternalsDriver()
        {
            @Override
            public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
            {
                return readLockPredicate(children, sequenceNodeName);
            }
        }
    );
}
```

```java
private PredicateResults readLockPredicate(List<String> children, String sequenceNodeName) throws Exception {
		if ( writeMutex.isOwnedByCurrentThread() )
        {
            return new PredicateResults(null, true);
        }

        int         index = 0;
        int         firstWriteIndex = Integer.MAX_VALUE;  // 2147483647
        int         ourIndex = Integer.MAX_VALUE;
}
```

参数：

- children： _c_8f701105-0e7b-414a-9a53-982abf3af1ab___READ__0000000004
- sequenceNodeName：_c_8f701105-0e7b-414a-9a53-982abf3af1ab__READ__0000000004

#### 逻辑总结

firstWriteIndex: 2147483647

加读锁的逻辑非常简单：

- 每创建一个读锁都会在/locks/lock_01目录下创建一个顺序节点；

- 获取这个顺序节点在目录下索引的位置，只要这个位置 < 2147483647，都能加读锁成功。

- ```java
  boolean     getsTheLock = (ourIndex < firstWriteIndex);
  ```

  因此，N多个客户端同时加读锁，肯定不是互斥的。

### 读锁+写锁

假如现在在locks/lock_01目录下面，已经有了N个读锁的顺序节点：

_c_82fe2ab8-3a7b-45a6-b5ff-044372979fa4-__READ__0000000010

加写锁的时候，会在locks/lock_01目录下面直接创建一个__WRIT__的顺序节点：

_c_82fe2ab8-3a7b-45a6-b5ff-044372979fa4-__WRIT__0000000011

接着来看internalLockLoop方法：

```java
PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
```

此时children存储的数据为：

[	

0 = "_c_4ad5c44d-cf2d-45ce-ad55-a7040080256b-__READ__0000000017"
1 = "_c_e4219946-2857-4c56-a6ba-022270a67c5a-__WRIT__0000000018"

]

sequenceNodeName：_c_e4219946-2857-4c56-a6ba-022270a67c5a-__WRIT__0000000018

maxLeases：1（见InterProcessReadWriteLock类）

```java
/**
 * @param client the client
 * @param basePath path to use for locking
 */
public InterProcessReadWriteLock(CuratorFramework client, String basePath)
{
    writeMutex = new InternalInterProcessMutex
    (
        client,
        basePath,
        WRITE_LOCK_NAME,
        1,
        new SortingLockInternalsDriver()
        {
            @Override
            public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
            {
                return super.getsTheLock(client, children, sequenceNodeName, maxLeases);
            }
        }
    );
```

StandardLockInternalsDriver类：

```java
@Override
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
{
    int             ourIndex = children.indexOf(sequenceNodeName);
    validateOurIndex(sequenceNodeName, ourIndex);

    boolean         getsTheLock = ourIndex < maxLeases;
    // 如果getsTheLock为false，则会通过outerIndex-maxLeases 获得上一个节点
    String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

    return new PredicateResults(pathToWatch, getsTheLock);
}
```

ourIndex = 1，maxLeases也为1，ourIndex < maxLease 不成立，所以getsTheLock为false：

LockInternals类：

287行：

```java
PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
if ( predicateResults.getsTheLock() )
{
    haveTheLock = true;
}
else
{
    // 获取到前一个节点：/locks/lock_01/_c_4ad5c44d-cf2d-45ce-ad55-a7040080256b-__READ__0000000017
    String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

    synchronized(this)
    {
        // 给前一个节点加上监听器
        Stat stat = client.checkExists().usingWatcher(watcher).forPath(previousSequencePath);
      ......
```

- 给前一个节点加上监听器，如果前面一个节点释放了锁，他就会被唤醒，重新去判断自己是不是处于children中的第一个节点；
- 如果是第一个节点的话，才能加锁成功。

#### 总结

除非前面的读锁释放成功了，写锁才能被唤醒后尝试加写锁成功。

### 写锁+读锁

#### 简单使用

先加写锁，再尝试加读锁：

```java
InterProcessReadWriteLock lock = new InterProcessReadWriteLock(client, "/locks/lock_01");
lock.writeLock().acquire();
lock.readLock().acquire();
```

children：

[

0 = "_c_e1dee7a7-c820-47b2-8abd-bb41079b057f-__WRIT__0000000031"
1 = "_c_38094ad5-61be-4931-9494-2c9f291ad2b5-__READ__0000000032"

]

InterProcessReadWriteLock类：

```java
private PredicateResults readLockPredicate(List<String> children, String sequenceNodeName) throws Exception
{
    if ( writeMutex.isOwnedByCurrentThread() )
    {
        return new PredicateResults(null, true);
    }
```

如果要加写锁的是同一个客户端，则允许加锁。同一个客户端写锁和读锁不互斥。

如果是不同的客户端，则：

```java
for ( String node : children )
{
    if ( node.contains(WRITE_LOCK_NAME) )
    {
        firstWriteIndex = Math.min(index, firstWriteIndex);
    }
    else if ( node.startsWith(sequenceNodeName) )
    {
        ourIndex = index;
        break;
    }

    ++index;
}
```

如果children里包含写锁，则：

firstWriteIndex = 0，ourIndex =1。

```java
boolean     getsTheLock = (ourIndex < firstWriteIndex);
```

1 < 1，肯定不成立，加写锁失败。

##### 总结

- 如果是同一个客户端，先加写锁，再加读锁是非互斥的，可以加锁；

- 不同的客户端，先加写锁，再加读锁是互斥的，此时一律不让加读锁，加锁失败了，就wait等待。

### 写锁+写锁

```java
private PredicateResults readLockPredicate(List<String> children, String sequenceNodeName) throws Exception
{
    if ( writeMutex.isOwnedByCurrentThread() )
    {
        return new PredicateResults(null, true);
    }

    int         index = 0;
    int         firstWriteIndex = Integer.MAX_VALUE;
    int         ourIndex = Integer.MAX_VALUE;
    for ( String node : children )
    {
        if ( node.contains(WRITE_LOCK_NAME) )
        {
            firstWriteIndex = Math.min(index, firstWriteIndex);
        }
        else if ( node.startsWith(sequenceNodeName) )
        {
            ourIndex = index;
            break;
        }

        ++index;
    }
    StandardLockInternalsDriver.validateOurIndex(sequenceNodeName, ourIndex);

    boolean     getsTheLock = (ourIndex < firstWriteIndex);
    String      pathToWatch = getsTheLock ? null : children.get(firstWriteIndex);
    return new PredicateResults(pathToWatch, getsTheLock);
}
```

由于第一个加锁的是写锁，第二个加锁的也是写锁，所以

```java
if ( node.contains(WRITE_LOCK_NAME) )
else if ( node.startsWith(sequenceNodeName) )
        {
            ourIndex = index;
            break;
        }
```

这两个判断都不成立，ourIndex（Integer.MAX_VALUE） < firstWriteIndex （Integer.MAX_VALUE）不成立，返回false，加写锁失败。

#### 总结

写锁 + 写锁 是互斥的。