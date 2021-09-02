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



else if ((f = tabAt(tab, i = (n - 1) & hash)) == null)   =》也就是说如果这个位置没有元素的话，就把 key-value 对放进数组的这个位置。

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

这里使用的是 UnSafe和CAS操作，确保了线程安全性。同一时刻只有一个线程可以把这个 key-value 对放进数组的这个位置上去。