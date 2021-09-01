## LinkedBlockingQueue

### 源码分析

#### 1.属性

```java
/**
 * 节点类，用于存储数据
 */
static class Node<E> {
	E item;
  Node<E> next;
  Node(E x) {
    item = x; 
  }
}

/**
 * 阻塞队列的大小，默认为Integer.MAX_VALUE
 */
private final int capacity;

/**
 * 当前阻塞队列中元素的个数
 */
private final AtomicInteger count = new AtomicInteger();

/**
 * 阻塞队列的头节点
 */
transient Node<E> head;

/**
 * 阻塞队列的尾节点
 */
transient Node<E> last;

/**
 * 获取并移除元素时使用的锁，如take,poll
 */
private final ReentrantLock takeLock = new ReentrantLock();

/**
 * 当队列没有数据时，用于挂起 "执行删除" 的线程
 */
private final Condition notEmpty = takeLock.newCondition();

/**
 * 添加元素时使用的锁，如put,offer
 */
private final ReentrantLock putLock = new ReentrantLock();

/**
 * 当队列已满时，用于挂起 "执行添加" 的线程
 */
private final Condition notFull = putLock.newCondition();
```

从上面的属性我们可以看出：

- 与ArrayBlockingQueue不同的是，LinkedBlockingQueue内部分别使用了takeLock和putLock两把锁对并发进行控制，也就是说，**添加和删除操作并不是互斥操作，可以同时进行，这样可以大大提高吞吐量**。
- 如果不指定容量的大小，则默认为Integer.MAX_VALUE，如果存在速度大于删除速度的时候，有可能会内存溢出。这一点在使用之前要慎重考虑。
- 对每一个lock锁都使用了Condition用来挂起和唤醒其它线程。



是一个有界队列，有大小限制。如果超过了限制，你往队列里塞数据就会被阻塞。

好处就在于说可以限制内存队列的大小，避免说内存队列无限制的增长，最后撑爆内存。



同一时间只能有一个线程可以入队。

队列满时，是如何实现阻塞线程的？



while(coun.get == capacity) {

​	// 直接会调用put锁对应的condition队列进行阻塞

}

take掉一个元素后，队列不满了，是如何唤醒线程的？



队列为空时，是如何实现阻塞线程的？



### size

size直接通过CAS+ volatile，拿到的基本是比较准备的一个值。

#### iterator

是直接锁了整个队列，直接把两把锁都给锁掉了。

遍历的时候不允许入队或出队了。

