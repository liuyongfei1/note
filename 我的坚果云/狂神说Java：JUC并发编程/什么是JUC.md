### 什么是JUC

java并发编程时用到的这几个包：

<img src="Untitled.assets/image-20211116232142444.png" alt="image-20211116232142444" style="zoom:50%;" />

### 线程和进程

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

### Lock锁

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



#### 生产者和消费者

三步曲：

判断是否等待/ 业务代码 / 通知

![image-20211117092858925](什么是JUC.assets/image-20211117092858925.png)

https://www.bilibili.com/video/BV1B7411L7tE?p=7&spm_id_from=pageDriver



> Condition可以精确的通知和唤醒线程

https://www.bilibili.com/video/BV1B7411L7tE?p=13&spm_id_from=pageDriver



#### Callable

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





