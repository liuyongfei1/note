## LinkedBlockingQueue

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

