### ThreadLocal

对于同一个ThreaLocal变量，每个线程都会有自己线程的一个副本。

#### demo

```java
public class ThreadLocalDemo {

    private static ThreadLocal<Long> requestId = new ThreadLocal<>();

    public static void main(String[] args) {
        new Thread() {
            @Override
            public void run() {
                requestId.set(1L);
                System.out.println("线程：" + Thread.currentThread() + "输出：" + requestId.get());
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                requestId.set(2L);
                System.out.println("线程：" + Thread.currentThread() + "输出：" + requestId.get());
            }
        }.start();
    }
}
```

输出结果：

```bash
线程：Thread[Thread-0,5,main]输出：1
线程：Thread[Thread-1,5,main]输出：2
```



#### 源码分析

JDK里面的Thread类，内部就有一个ThreadLocalMap类，代表了一个Map，每个Thread对象自己内部都有一个核心数据结构是Map。

这个map只能是自己线程内部可以使用的一份数据，就是线程自己本地的一个副本数据，类似这样的结构：

```txt
Thread {

 	ThreadMap {

      ThreadLocal {

         requestId: 1L,

         txtId: 1L,  
         ......
      }
   }
}
```



get的时候，大概的流程是这样：

requestId.get() -> Thread.ThreadLocalMap -> ThreadLocal(requestId) -> 1L。

