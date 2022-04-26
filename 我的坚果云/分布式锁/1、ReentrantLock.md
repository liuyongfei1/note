## ReentrantLock

ReentrantLock + AQS。

### demo

```java
 ReentrantLock lock = new ReentrantLock();
 lock.lock();
 lock.unlock();
```

### 源码

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

默认是非公平策略。为什么默认是非公平策略？

因为非公平锁的效率高呀。当一个线程请求非公平锁时，`如果在发出请求的同时该锁变成可用状态`，那么这个线程就会跳过所有的等待线程而获得锁。

有的同学会说了，这不就是插队吗？没错，这就是插队！这也就是为什么它被称为非公平锁。

之所以使用这种方式是因为：`在恢复一个被挂起的线程到该线程真正运行之间存在严重的延迟。`

当然，非公平锁**仅仅是在 当前线程请求锁，并且锁处于可用状态时**有效，当请求锁时，锁已经被其它线程占有时，就只能是老老实实的去排队了。



- **非公平策略：**在多个线程争用锁的情况下，能够最终获得锁的线程是随机的（由底层OS调度）。
- **公平策略：**在多个线程争用锁的情况下，公平策略倾向于将访问权授予等待时间最长的线程。也就是说，相当于有一个线程等待队列，先进入等待队列的线程后续会先获得锁，这样按照“先来后到”的原则，对于每一个等待线程都是公平的。

```java
public void lock() {
        sync.lock();
    }

static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

Sync是一个抽象内部类，NonfairSync集成了Sync，并重写了lock方法。

if (compareAndSetState(0, 1)) =》 相当于在尝试加锁

```java
protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

- 底层是基于UnSafe来实现的。JDK内部的API，基于CPU指令来实现原子性的CAS。

- 这段代码可以保证说，在一个原子操作中，如果发现这个值是我们期望的值，则说明没人修改过，此时可以将这个值设置为update，state如果是0的话，就修改为1，代表加锁成功了。

如果加锁成功，则执行：

```java
setExclusiveOwnerThread(Thread.currentThread());
```

设置当前线程自己是 加了一个独占锁的线程，标识出来自己是加锁的线程。
