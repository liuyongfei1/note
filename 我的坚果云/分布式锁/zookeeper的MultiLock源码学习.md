## zookeeper的MultiLock源码学习

multiLock 无非是把多个小锁给合并成一个大锁。

### 简单使用

```java
// ***************MultiLock *********************
InterProcessLock lock1 = new InterProcessMutex(client,"/locks/lock_01");
InterProcessLock lock2 = new InterProcessMutex(client,"/locks/lock_02");
InterProcessLock lock3 = new InterProcessMutex(client,"/locks/lock_03");

List<InterProcessLock> locks = new ArrayList<>();
locks.add(lock1);
locks.add(lock2);
locks.add(lock3);

InterProcessMultiLock multiLock = new InterProcessMultiLock(locks);
multiLock.acquire();
multiLock.release();
// ***************MultiLock end******************
```

### 源码分析

```java
/**
 * {@inheritDoc}
 */
@Override
public void acquire() throws Exception
{
    acquire(-1, null);
}
```

```java
/**
 * {@inheritDoc}
 */
@Override
public boolean acquire(long time, TimeUnit unit) throws Exception
{
    Exception                   exception = null;
    List<InterProcessLock>      acquired = Lists.newArrayList();
    boolean                     success = true;
    for ( InterProcessLock lock : locks )
    {
        try
        {
            if ( unit == null )
            {
                // 加锁
                lock.acquire();
                acquired.add(lock);
            }
            else
            {
                if ( lock.acquire(time, unit) )
                {
                    acquired.add(lock);
                }
                else
                {
                    success = false;
                    break;
                }
            }
        }
        catch ( Exception e )
        {
            // 如果有报错
            success = false;
            exception = e;
        }
    }

    if ( !success )
    {
        for ( InterProcessLock lock : reverse(acquired) )
        {
            try
            {
                lock.release();
            }
            catch ( Exception e )
            {
                // ignore
            }
        }
    }

    if ( exception != null )
    {
        throw exception;
    }
    
    return success;
}
```

### 总结

- 遍历locks，对每个lock进行加锁；
- 在这个过程中，一旦有一个加锁失败，就依次遍历之前已经加过的锁，并释放掉。