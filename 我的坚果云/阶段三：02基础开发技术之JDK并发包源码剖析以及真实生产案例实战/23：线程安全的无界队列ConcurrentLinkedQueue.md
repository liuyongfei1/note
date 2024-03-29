## ConcurrentLinkedQueue

哪怕是线程安全的集合包，使用的时候也要考虑到多线程并发的一些数据不一致的问题。

为了优化多线程并发性能，牺牲掉了一些数据一致性的。为了保障多线程并发写队列的性能，大量的采用了CAS无锁化的操作，很多读操作，尤其是常见的size，不涉及任何锁。

### 无界队列

没有大小限制，是一个单向的链表，无限制的往里面去存放数据，可能会导致内存溢出。

### 对哪些操作是并发安全的？

#### size()

size方法源码，就是看一下队列的大小，并没有任何的锁机制，直接从头结点开始遍历，遍历每个链表中的节点，count++。

如果在遍历的过程中，有人执行入队或是出队的操作，此时会怎么样？

##### 入队

入队最核心的操作，就是设置队列尾部节点的next指针，设置好了以后，新的节点就入队了，使用的volatile写。

你在遍历的时候，立马就可以看得到。

##### 出队

如果是从头开始遍历的时候，遍历到了一半儿，此时有人从队头出队，会把队头的一个节点给摘掉。这个就没办法了，此时是感知不到的。



size之类的操作，都是一瞬间的快照看到的数量，不一定是准确的。比如你拿到了一个size，可能你刚刚拿到一个队列的size是10，结果立马队列大小就变成了5。

#### Iterator()方法

interator机制，是基于副本快照机制来实现的，创建了一个迭代器之后，只会用当时的一个副本快照来完成遍历。因此也有可能会存在数据不一致的问题。

