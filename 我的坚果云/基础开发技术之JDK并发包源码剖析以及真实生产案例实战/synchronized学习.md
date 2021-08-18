### synchronized介绍

synchronized加锁，一旦加了锁之后，其它的线程就没法去读取和修改这个变量了。

1. 同一时间，只有一个线程可以读取和修改这个数据，别的线程都会卡在那里尝试获取锁；

2. synchronized锁两种东西，一种是对某个实例对象来加锁，另外一种是对这个类进行加锁；

3. 对类加锁，也是在对某一个对象实例进行加锁，他的意思就是对这个类的Class对象进行加锁；
4. 你直接在一个方法上加synchronized，那么就是对当前这个对象实例在加锁，访问对象实例的synchronized方法，同一时间只有一个线程可以做到。

### 加锁方式

下面这几种加锁方式也是可以的。

```java
synchronized(myObject) {

}

synchronized(this) {

}

synchronized(MyObject.class) {

}
```

### synchronized底层原理

其实synchronized的底层原理是跟jvm指令和monitor有关系的。

你如果用到了synchronized关键字，在底层编译后的指令中，会有monitorenter和monitorexit两个指令。

```
monitorenter指令
// 代码对应的指令
monitorexit执行
```

那么monitorenter指令执行的时候会干什么呢？

每个对象都有一个关联的monitor，比如一个对象实例就有一个monitor，一个类的Class对象也有一个monitor，如果要对这个对象加锁，那么必须获取到这个对象关联的monitor的lock锁。

原理大概是这样的：

- monitor里有一个计数器，是从0开始的；

- 如果一个线程要获取monitor的锁，就看看他的计数器是不是0，如果是0的话，那么就说明还没有人获取锁，那么这个线程就可以获取锁了，然后对计数器加1。

- 这个monitor锁是支持重入加锁的，比如下面的代码：

  ```
  synchronized(myObject) {
  	// 一大堆的代码
  	synchronized(myObject) {
  		// 一大堆的代码
  	}
  }
  ```

  - 如果一个线程获取myObject对象的monitor锁，计数器会加1；

  - 然后第二次还是这个线程来获取myObject的锁，这个就是重入加锁了，计数器会再次加1，变成2。
  - 这时候，其它线程在外层的synchronized那里，会发现myObject的monitor锁的计数器大于0，意味着别人已经加过锁了，然后此时该线程就会进入block阻塞状态，什么都干不了，就是等着获取锁。
  - 接着如果出了synchronized修改的代码片段范围，就会有一个monitorexit的指令，在底层，此时获取锁的线程就会对那个对象的monitor锁的计数器减1，如果有多次重入加锁，就多次减1，直到计数器为0。
  - 然后后边block的线程，会再次尝试获取锁，但是只有一个线程可以获取到锁。