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