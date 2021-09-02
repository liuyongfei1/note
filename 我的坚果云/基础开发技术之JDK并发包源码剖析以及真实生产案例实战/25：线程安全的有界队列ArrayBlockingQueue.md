## ArrayBlockingQueue

ArrayBlockingQueue是基于独占锁实现的线程安全的出队和入队，基于数组实现的阻塞队列。

一把独占锁，锁住整个数组，同一时间只能有一个线程可以入队或出队。

### 源码分析

- 默认情况下从 index = 0 的位置开始入队。

- count是用来保存你的队列里有多少元素。

[null,null,null,null,null,null,null,null]

index = 0, count = 0

#### 入队

主要看一下这个入队的方法：

```java
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```

将 "张三" 入队：

- items[0] = "张三";
- putIndex递增，变为1；
- 如果数组满了，则将putIndex置为0；
- count递增，等于1；
- 如果不为空，则使用Condition的signal()方法唤醒其它线程来消费。

则 index = 1, count = 1。

假如再来个新元素"赵四"入队，则 items[1] = "李四", index = 2, count =2。

#### 出队

主要看这个方法：

```java
private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```



