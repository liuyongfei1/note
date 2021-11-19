### 1、什么是JUC

java并发编程时用到的这几个包：

<img src="Untitled.assets/image-20211116232142444.png" alt="image-20211116232142444" style="zoom:50%;" />

### 2、线程和进程

> 线程、进程

进程：一个程序，QQ.exe，Music.exe，程序的集合。

一个进程包含多个线程。

Java默认有两个线程，一个是main线程，一个是GC线程。

线程：比如使用这个Typora这个软件，会有一个线程负责输入文字，还会有一个线程负责自动保存文字。

Java中通过Threa、Runnable、Callable去使用线程。



```java
// 查看CPU核数
System.out.println(Runtime.getRuntime().availableProcessors());
```

>  线程Thread有几个状态

```java
public enum State {
    /**
     * 新生
     */
    NEW,

    /**
     * 运行
     */
    RUNNABLE,

    /**
     * 阻塞
     */
    BLOCKED,

    /**
     * 等待，死死的等
     */
    WAITING,

    /**
     * 超时等待
     */
    TIMED_WAITING,

    /**
     * 终止
     */
    TERMINATED;
```

> wait与sleep区别

1、**来自不同的类**

wait -> Object

sleep -> Thread 类独有的

2、**关于锁的释放**

wait会释放锁；

sleep睡觉了，抱着锁睡的，不会释放锁。

3、**使用的范围是不同的**

- wait必须在同步代码块内使用；

- sleep可以在任何地方睡。

4、**是否需要捕获异常**

- wait不需要捕获异常；

- sleep需要捕获异常（因为有可能会发生超时等待）。

### 3、Lock锁

https://www.bilibili.com/video/BV1B7411L7tE?p=4&spm_id_from=pageDriver

Lock锁是一个接口，它的实现类是：

<img src="什么是JUC.assets/image-20211117123113110.png" alt="image-20211117123113110" style="zoom:50%;" />

>  Synchronized和Lock区别

1、Synchronized 内部的java关键字，Lock是一个java类；

2、Synchronized无法获取锁的状态，Lock可以判断是否获取到了锁；

3、Synchronized 会自动释放锁，Lock必须手动释放锁，如果不释放锁，会造成死锁；

4、Synchronized 线程1获取到锁如果被阻塞了，线程2会傻傻的等，Lock锁就不一定会等待下去；

5、Synchronized 可重入锁，不可以中断的、非公平锁；Lock 可重入锁、可以判断锁，非公平（可以设置为公平锁）；

6、Synchronized 适合少量的代码同步问题，Lock适合大量的同步代码。



### 4、生产者和消费者

三步曲：

判断是否等待/ 业务代码 / 通知

![image-20211117092858925](什么是JUC.assets/image-20211117092858925.png)

https://www.bilibili.com/video/BV1B7411L7tE?p=7&spm_id_from=pageDriver



> Condition可以精确的通知和唤醒线程

https://www.bilibili.com/video/BV1B7411L7tE?p=13&spm_id_from=pageDriver

### 5、8锁现象

### 6、集合类不安全

### 7、Callable

<img src="什么是JUC.assets/image-20211117225155870.png" alt="image-20211117225155870" style="zoom:50%;" />

<img src="什么是JUC.assets/image-20211117225346380.png" alt="image-20211117225346380" style="zoom:50%;" />

FutureTask是Runnable的实现类。

>  怎么使用Callable呢？

默认传统的做法是：

```java
class MyThread implements Runnable {

    @Override
    public void run() {
        System.out.printf("run()......");
    }
}
```

new Thread(new MyThread()).start(); 就可以启动一个线程。

如果使用Callable，该怎么启动一个线程呢？

```java
class MyThread implements Callable<String> {

    @Override
    public String call() throws Exception {
        System.out.println("call()");
        return "123456";
    }
}
```

我们通过查看文档可以发现，FutureTask是Runnable的一个实现类，而Future又可以调Runnable。所以可以这样做：

```java
MyThread thread  = new MyThread();
FutureTask futureTask = new FutureTask(thread);
new Thread(futureTask, "A").start();

// 获取线程执行结果的返回值(这个get方法有可能会产生阻塞，所以一般把这行代码放在最后)
// 或者使用异步通信(让这个线程跑着，处理完之后，我再去拿结果)
String result = (String) futureTask.get();
System.out.println(result);
```

**引申**

如果再启一个B线程，那么会打印几个call？

```java
MyThread thread  = new MyThread();
FutureTask futureTask = new FutureTask(thread);
new Thread(futureTask, "A").start();
new Thread(futureTask, "B").start();

// 获取线程执行结果的返回值(这个get方法有可能会产生阻塞，所以一般把这行代码放在最后)
// 或者使用异步通信(让这个线程跑着，处理完之后，我再去拿结果)
String result = (String) futureTask.get();
System.out.println(result);
```

答案：还是一个call()

原因：**结果会被缓存，提高效率。**

##### 细节

1、 Callable 有缓存；

2、结果可能要等待，会阻塞！

### 8、常用的辅助类

#### 8.1、CountDownLatch

```java
/**
 * CountDownLatch的使用demo
 *
 * 教室里有6个小学生，在6个小学生都离开教室后，保安会把教室的门锁上。
 *
 * @author Liuyongfei
 * @date 2021/11/17 23:23
 */
public class CountDownLatchDemo2 {

    public static void main(String[] args) {
        // 创建一个计数器
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "Go Out");

                // 数量减一
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }

        System.out.println("Close Door");
    }
}
```

执行结果：

```bash
1Go Out
3Go Out
2Go Out
Close Door
5Go Out
4Go Out
6Go Out
```

可以看到，会发生还有学生没出教室呢，就关门的现象。

解决：

添加： countDownLatch.await();

```java
/**
 * CountDownLatch的使用demo
 *
 * 教室里有6个小学生，在6个小学生都离开教室后，保安会把教室的门锁上。
 *
 * @author Liuyongfei
 * @date 2021/11/17 23:23
 */
public class CountDownLatchDemo2 {

    public static void main(String[] args) {
        // 创建一个计数器
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "Go Out");

                // 数量减一
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }

        // 等待计数器归零，再向下执行
        countDownLatch.await();
        System.out.println("Close Door");
    }
}
```

执行结果：

```java
4Go Out
1Go Out
2Go Out
5Go Out
6Go Out
3Go Out
Close Door
```

**原理**

`countDownLatch.countDown();` // 数量减1

`countDownLatch.await();` // 等待计数归零，然后再向下执行

- 每次有线程调用countDown()，数量就会减1；
- 假如计数器变为0，则countDownLatch.await()就会被唤醒，继续向下执行。

#### 8.2 、CyclicBarrier

是一个加法计数器。 

await会计数，达到指定计数后，会开启一条新的线程继续执行。

https://www.bilibili.com/video/BV1B7411L7tE?p=16&spm_id_from=pageDriver

#### 8.3、Semaphore

 可以理解为 抢停车位！

`限流的时候会用这个。你的流量入口只有这么大，你一次性只能进来这么多，不可能超过这个数量了。` 

**原理：**

`semphore.accure()`获得（-1），假设如果已经满了，则会等待，直到被释放为止！

`semaphore.release()`释放，会将当前的信号量释放（+1），然后唤醒等待的线程！

**作用：**

多个共享资源互斥时使用！并发限流，控制最大的线程数。

### 9、读写锁

先来看一个案例：

```java
/**
 * 读写锁的案例
 * 1.不使用读写锁
 * @author Liuyongfei
 * @date 2021/11/18 09:16
 */
public class ReadWriteLockDemo {

    public static void main(String[] args) {
        MyCache myCache = new MyCache ();

        // 启动5个线程，同时向cache中写入数据
        for (int i = 1; i <= 5 ; i++) {
            final int tmp = i;
            new Thread(() -> {
                myCache.put(tmp + "", tmp + " value");
            }, String.valueOf(i)).start();
        }

        // 启动5个线程，同时从cache中读取数据
        for (int i = 1; i <= 5; i++) {
            final  int tmp = i;
            new Thread(() -> {
                myCache.get(tmp + "");
            }, String.valueOf(i)).start();
        }
    }
}

/**
 * 自定义缓存
 */
class MyCache {
    private volatile HashMap<String, Object> cacheMap = new HashMap<>();

    /**
     * 存，写
     * @param key
     * @param value
     */
    public void put(String key, Object value) {
        System.out.println(Thread.currentThread().getName() + " 写入 " + key);
        cacheMap.put(key, value);
        System.out.println(Thread.currentThread().getName() + " 写入ok");
    }

    /**
     * 取，读
     * @param key
     */
    public void get(String key) {
        System.out.println(Thread.currentThread().getName() + " 读取 " + key);
        Object value = cacheMap.get(key);
        System.out.println(Thread.currentThread().getName() + " 读取ok：" + value);
    }
}
```

执行结果可能不会符合你的预期：

```bash
1 写入 1
3 写入 3
3 写入ok
2 写入 2
2 写入ok
4 写入 4
4 写入ok
1 写入ok
5 写入 5
5 写入ok
1 读取 1
2 读取 2
3 读取 3
3 读取ok：3 value
1 读取ok：1 value
2 读取ok：2 value
4 读取 4
4 读取ok：4 value
5 读取 5
5 读取ok：5 value
```

可以看到，1这个值还没写入到cache中，就有其它线程也在执行写入操作了。

**解决办法**

使用读写锁，保证在同一时刻写的时候 只能有一个线程能进行写；读的时候多个线程可以同时读。

```java
/**
 * 自定义缓存，使用读写锁
 */
class MyCacheLock {
    private volatile HashMap<String, Object> cacheMap = new HashMap<>();

    // private Lock lock = new ReentrantLock();
    // 声明一个读写锁：更加细粒度的控制
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();


    /**
     * 存，写
     * @param key
     * @param value
     */
    public void put(String key, Object value) {
        readWriteLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 写入 " + key);
            cacheMap.put(key, value);
            System.out.println(Thread.currentThread().getName() + " 写入ok");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }

    /**
     * 取，读
     * @param key
     */
    public void get(String key) {
        readWriteLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 读取 " + key);
            Object value = cacheMap.get(key);
            System.out.println(Thread.currentThread().getName() + " 读取ok：" + value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.readLock().unlock();
        }
    }
}
```

输出结果：

```bash
1 写入 1
1 写入ok
2 写入 2
2 写入ok
3 写入 3
3 写入ok
4 写入 4
4 写入ok
5 写入 5
5 写入ok
1 读取 1
1 读取ok：1 value
2 读取 2
2 读取ok：2 value
3 读取 3
3 读取ok：3 value
4 读取 4
4 读取ok：4 value
5 读取 5
5 读取ok：5 value
```



### 10、阻塞队列

<img src="什么是JUC.assets/队列家族.png" alt="队列家族" style="zoom:80%;" />

#### 阻塞

- 如果队列满了，此时再往里边写入，则会阻塞；

- 如果队列空了，此时再从里边进行读取元素，则会阻塞；

List、Set的祖宗类都是 Collection。

####  学会使用队列 

添加、移除

#### 四组API

1、抛出异常

2、不会抛出异常

3、阻塞等待

4、超时等待

https://www.bilibili.com/video/BV1B7411L7tE?p=20&spm_id_from=pageDriver



> SynchronousQueue同步队列

和其它的BlockingQueue是不一样的。没有容量，进去一个元素，必须等待取出来之后，才能往里边再放一个元素。

### 11、线程池（重点）

线程池：3大方法、7大参数、4种拒绝策略

>  池化技术

程序的运行，本质就是占用系统的资源，如何去优化资源的使用呢 =》池化技术

事先准备好一些资源，有人要用，就来我这里拿，用完之后还给我。

**线程池的好处**

1、降低资源的消耗；

2、提高响应的速度；

3、便于管理。

> 线程池：3大方法

https://www.bilibili.com/video/BV1B7411L7tE?p=23&spm_id_from=pageDriver

> 7大参数

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

可以发现，开启线程，本质是调用ThreadPoolExecutor。查看ThreadPoolExecutor源码：

```java
public ThreadPoolExecutor(int corePoolSize, // 核心线程池线程数大小
                          int maximumPoolSize, // 最大线程池线程数大小
                          long keepAliveTime, // 超时多久，没有人调用，就会释放(比如银行的窗口，超过1小时没有人来办理业务，就关闭窗口了)
                          TimeUnit unit, // 超时时间的单位
                          BlockingQueue<Runnable> workQueue,// 阻塞队列(比如银行的候客区，最多只能容纳3个人)
                          ThreadFactory threadFactory, // 线程工程，用来创建线程的，一般不用动
                          RejectedExecutionHandler handler) // 拒绝策略(比如银行的窗口人满了，候客区也满了，这时再有人想办理业务就会被拒绝，不处理这个人的，抛出异常)
```

可以发现这里边有7个参数。

> 阿里巴巴开发规范中提到，不要用Executors去创建，而是使用ThreadPoolExecutor的方式去创建线程池的原因是什么？

1、看一下源码，这几种方式底层都是通过使用ThreadPoolExecutor的方式去创建线程池，规避资源耗尽的风险；

2、FixedThreadPool和SingleThreadPool、CachedThreadPool和ScheduledThreadPool，允许请求的队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，导致OOM。

>  手动创建一个线程池

```java
/**
 * The default rejected execution handler
 */
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();
```

