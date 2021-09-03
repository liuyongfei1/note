## ConcurrentHashMap

分段加锁，将一份数据拆分为多个segment，对每个segment设置一把小锁。比如put操作，仅仅是锁定一个segment而已，锁一部分的数据，其它的线程操作segment的数据，跟你是没有竞争的。

可以提高多线程并发操作hashmap的效率。

### 源码分析

#### put()方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

// 使用spread算法，获取到一个hash值
int hash = spread(key.hashCode());

static final int spread(int h) {
        // 相当于是把高低16位都考虑到后面的hash取模算法里，可以降低hash冲突的概率。
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

- spread算法，通过高低16位异或的操作，降低hash冲突的概率；
- 然后hash值再跟数组的大小做了一个取模算法，定位到数组的一个位置。

else if ((f = tabAt(tab, i = (n - 1) & hash)) == null)   =》也就是说如果这个位置没有元素的话，就把 key-value 对放进数组的这个位置。

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

这里使用的是 UnSafe和CAS操作，确保了线程安全性。同一时刻只有一个线程可以把这个 key-value 对放进数组的这个位置上去。

#### 分段加锁

其实这里已经体现了分段加锁的机制，它并没有对整个数组进行synchronized加锁，这样，线程会串行化起来，并发效率会比较低。

仅仅是对数组的同一个位置的元素赋值的时候，多个线程会基于CAS（隐式的锁），对那个元素进行加锁。

数组大小是16，有16个元素，同时可以允许最多是16个线程同时并发的来操作这个数组，如果16个线程操作的是数组的不同位置的元素的话，此时16个线程之间是没有任何锁的关系，只有竞争同一个位置的元素的多个线程，才会对一把锁进行争用的操作，大概可以这么来理解。