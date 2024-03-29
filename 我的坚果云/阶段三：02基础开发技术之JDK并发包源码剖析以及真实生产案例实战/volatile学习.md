### 多线程共用一个变量遇到的问题

多个线程，如果共用一个变量，有一个线程负责修改这个变量，有一个线程负责读取这个变量，这个时候会出现什么问题吗？

先来看一个demo，这里开启了两个线程，我们的本意是其中一个线程修改变量后，另外一个线程可以里面感知到变量发生了变化：

```java
public class VolatileDemo {
    private static int flag = 0;
    public static void main(String[] args) {
        new Thread(){
            @Override
            public void run() {
                int localFlag = flag;
                while (true) {
                    if (localFlag != flag) {
                        System.out.println("读取到了修改后的标志位：" + flag);
                        localFlag = flag;
                    }
                }
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                int localFlag = flag;
                while (true) {
                    System.out.println("标志位被修改了：" + ++localFlag);
                    flag = localFlag;
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();
    }
}
```

执行程序，却发现实际输出的跟我们期望的有差别：

```bash
标志位被修改了：1
读取到了修改后的标志位：1
标志位被修改了：2
标志位被修改了：3
标志位被修改了：4
标志位被修改了：5
标志位被修改了：6
标志位被修改了：7
标志位被修改了：8
......
```

我们发现，除了第一次的修改被感知到外，剩下的变量修改，另一个线程无法感知。

### 怎么解决？使用volatile

我们只需要在定义 flag变量的时候加上 volatile 关键字即可：

```java
private volatile static int flag = 0;
```

重新运行程序：

```bash
标志位被修改了：1
读取到了修改后的标志位：1
标志位被修改了：2
读取到了修改后的标志位：2
标志位被修改了：3
读取到了修改后的标志位：3
标志位被修改了：4
读取到了修改后的标志位：4
标志位被修改了：5
读取到了修改后的标志位：5
```

### 总结

- Thread1已经将 flag = true 设置好了；
- Thread0，比如说在一段时间内还是读到 flag = false。在一小段时间内范围，可能Threa0会感知不到Thread1对Threa0的修改。
- volatile这个东西，没那么高深，其实很简单，他是并发编程里很常见的一个，只要开了多个线程，一定会有这种问题，某个线程修改一个变量值，其他线程要立刻感知这个值的变化，这时就得用volatile。

#### volatile是如何实现保证可见性

**lock前缀指令 + MESI缓存一致性协议**

- 对于volatile修饰的变量，如果执行写操作，则JVM会给CPU发送一个lock前缀指令，会将更新后的数据写入主存；
- 同时因为有MESI一致性协议，各个CPU会都会对总线进行嗅探自己本地缓存中的数值是否发生了变化，如果发生了变化，则CPU会将自己本地工作内存的变量缓存刷为过期；
- 然后这个CPU上执行的线程在读取这个变量的时候就会直接从主内存加载最新的数据了。

#### volatile是如何保证有序性

##### 内存屏障+禁止排序

第二周27讲，需要再复习一下

volatile的写和读前后会加一些屏障，保证很多指令在一定规则下不会重排。

比如：**LoadLoad屏障**

```
int age = this.age => 相当于一个 load

LoadLoad屏障

int name = this.name => 相当于一个load
```

确保load1数据的装载会优先于load2后所有装载指令，意思就是load1和load2对应的代码，是不能指令重排的。   

#### double check的单例模式安全吗

第二周28讲（double check的缺陷），需要再复习一下

常见的doublie check单例模式：

```java
public class DoubleCheckSingleton {
    private static DoubleCheckSingleton instance;

    private DoubleCheckSingleton() {}

    public DoubleCheckSingleton getInstance() {
        if (instance == null) {
            // 多个线程会卡在这里
            synchronized (DoubleCheckSingleton.class) {
                // 第一个线程进来，发现instance为空，就创建一个实例
                // 这是第二个线程会进来，如果不加上这个if判断，就会再创建一遍实例。
                if (instance == null) {
                    DoubleCheckSingleton.instance = new DoubleCheckSingleton();
                }
            }
            // 第一个线程执行完毕，退出synchronized代码块，会是否synchronized锁
        }
        return instance;
    }
}
```

这种单例模式有什么缺陷？

~~拿Java的内存模型来讲：~~

- ~~线程1创建一个实例后，会缓存在自己的工作内存里，并没有强制刷回到主存；~~
- ~~线程2进来之后，获取instance时依然从线程2对应的工作内存里去取，由于instance依然是null，所以，又创建了一遍instance，写入到自己的工作内存里。~~

~~解决办法：给 instance加上 volatile 关键字即可，保证线程的可见性。~~

synchronized能保证原子性，所以本身也能保证可见性。这里的缺陷主要还是 指令重排的问题：

- 给 instance 加锁 volatile 关键字后，会解决指令重排的问题。前后的读写操作，就会保证一定的顺序。
- 对加了`volatile`的这个变量，只有都写成功了之后呢，才会允许其它的线程来读。不会说一个写还没全部执行结束完毕就自动让另一个先来执行读了。

可以见 /Users/lyf/Workspace/www/blog-demo/thread-demo/src/main/java/volatiledemo/Singleton.java

底层的细节，会有很多jvm指令，没必要深扣。

#### 实战场景1

register-client项目 优雅的关闭微服务

Register-client里有个 shutdown方法：

```java
/**
 * 关闭registerClient组件
 */
public void shutdown() {
    this.finishedRunning = false;
    heartbeatWorker.interrupt();
}
```

这个地方就是很典型的多线程操作共享变量的场景：

- 有一个线程执行shutdown方法，修改finishedRunning；

- 有一个heartbeat线程，读finishedRunning：

- ```java
  /**
   * 心跳线程
   */
  private class HeartbeatWorker extends Thread {
      @Override
      public void run() {
          HeartbeatRequest request = new HeartbeatRequest();
          request.setServerInstanceId(serviceInstanceId);
  
          HeartbeatResponse response = null;
          // 读finishedRunning
          while (finishedRunning) {
              try {
                  response = httpSender.heartbeat(request);
                  System.out.println("心跳的结果为：【" + response.getStatus() + "】......");
                  Thread.sleep(30 * 1000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
      }
  }
  ```

​    既有写，又有读，这时如果不用volatile，就有可能会发生 一个线程执行shutdown方法后，heartbeat线程却一直感知不到finishedRunning被修改为false，照样还一直在执行心跳请求。

##### 解决办法

给 finishedRunning 添加volatile关键字即可：

```java
/**
 * 服务是否在运行
 */
private volatile Boolean finishedRunning;
```

#### 实战场景2

Register-server项目：

- 一个线程接收心跳请求，会服务进行续约，实际是对ServerInstance里的Lease的latestHeartbeatTime变量进行写操作；
- 同时又有监控服务是否存活的后台线程读ServerInstance里的Lease的latestHeartbeatTime变量来判断服务是否存活。

这就存在多个线程对一个共享变量进行读写的问题。

##### 解决办法

同样是给latestHeartbeatTime变量加上volatile关键字即可。

```java
private class Lease {
    private volatile Long latestHeartbeatTime = System.currentTimeMillis();
```

