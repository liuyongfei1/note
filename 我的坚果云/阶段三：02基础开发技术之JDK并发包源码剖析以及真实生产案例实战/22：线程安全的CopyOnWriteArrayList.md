## CopyOnWriteArrayList

CopyOnWriteArrayList，写时复制机制的ArrayList。可以保证线程并发的安全性。

### 源码分析

```java
/**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

/** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;

 /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();
```

#### 

1. 从这个构造函数的代码，可以看出，CopyOnWriteArrayList的底层是基于数组来实现的；

2. array使用volatile关键字，保证多线程读写的可见性，只要有一个线程修改了这个数组，其它线程是立马可以读到的；

3. 每个CopyOnWriteArrayList底层除了对应一个Object[]外，还对应一个ReentrantLock独占锁。独占锁保证了只有一个线程可以获取到锁，修改底层数组里的数据。

4. 增删改查操作的时候，都必须先获取一把ReentrantLock独占锁，保证同一时间只可以有一个线程操作底层的数组。

  

### 总结

并发写CopyOnWriteArrayList的性能是比较差的，基本上所有的线程都要串行起来写，一个线程先写完，下一个线程才能写。