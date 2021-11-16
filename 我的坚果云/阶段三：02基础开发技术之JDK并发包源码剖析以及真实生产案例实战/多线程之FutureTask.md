### FutureTask概念

- FutureTask一个可取消的异步计算，FutureTask实现了Future的基本方法。

- 可以查询计算是否已经完成，并且可以获取计算的结果。结果只可以在计算完成之后获取。
- get方法会阻塞（当计算没有完成的时候），一旦计算完成，那么计算就不能再次启动或是取消。

一个FutureTask可以用来包装一个Callable或是一个runnable对象。因为FutureTask实现了Runnable方法，所以一个FutureTask可以提交（submit）给一个Executor执行（execution）。

#### Callable与Runnable的区别

Java发布的第一个版本，就可以很方便的编写多线程的应用程序，并在设计中引入了异步处理。Thread类、Runnable接口和Java内存管理模型使得多线程编程简单直接。

但Thread类和Runnable接口都不允许声明检查类型异常，也不能定义返回值。

比如这样的例子：

```java
/**
 * 多线程发生异常，无法使用try ...catch捕获异常
 *
 * @author Liuyongfei
 * @date 2021/11/15 23:35
 */
public class NoCaughtThread implements Runnable{
    @Override
    public void run() {
        System.out.println( 3 / 2);
        System.out.println( 3 / 1);
        System.out.println( 3 / 0);
    }

    public static void main(String[] args) {
        try {
            Thread thread = new Thread(new NoCaughtThread());
            thread.start();
        } catch (Exception e) {
            System.out.println("==Exception: " + e.getMessage());
        }
    }
}
```

运行，报错：

```bash
1
3
Exception in thread "Thread-0" java.lang.ArithmeticException: / by zero
	at exception.NoCaughtThread.run(NoCaughtThread.java:25)
	at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 0
```

**显然这并非程序设定异常捕获，此时try...catch无法捕获线程的异常。**

此时，如果线程因为异常而终止运行，将无法检测到异常。

究其原因Threa类run()方法是不抛出任何检查型异常的，而自身可能因为一个异常而终止。

这种情况的解决方式有两种。

#### 在run()中主动捕获异常

Thread类API中提供UncaughtExceptionHandler接口捕获异常，要求检测线程异常。发生异常设置为重复调用三次之后结束线程。

```java
/**
 * 主动捕获线程异常
 *
 * ① 在run()中设置对应的异常处理，主动方法来解决未检测异常
 *
 * @author Liuyongfei
 * @date 2021/11/16 08:15
 */
public class InitiativeCaught {

    public static void main(String[] args) {
        InitialtiveThread thread = new InitialtiveThread();

        ExecutorService exec = Executors.newCachedThreadPool();
        exec.execute(thread);
        exec.shutdown();
    }
}

class InitialtiveThread implements Runnable {

    @Override
    public void run() {
        Throwable throwable = null;
        try {
            System.out.println( 3 / 2);
            System.out.println( 3 / 1);
            System.out.println( 3 / 0);
        } catch (Exception e) {
            throwable = e;
        } finally {
            threadDeal(throwable);
        }
    }

    public void threadDeal(Throwable  t) {
        System.out.println("==Exception: " + t.getMessage());
    }
}
```



#### 使用Thread类API中提供UncaughtExceptionHandler接口捕获异常

```java
import java.lang.Thread.UncaughtExceptionHandler;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadMonitor implements Runnable {
    private int data;                    // 可设置通过构造传参
    private int control = 0;
    private static final int MAX = 3;    // 设置重试次数

    public ThreadMonitor(int i) {
        this.data = i;
    }
    public ThreadMonitor() {
        // TODO Auto-generated constructor stub
    }

    @Override
    public void run() {
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread arg0, Throwable e) {
                // TODO Auto-generated method stub
                System.out.println("==Exception: " + e.getMessage());
                String message = e.getMessage();
                if( control==MAX ){
                    return ;
                }else if( "ok".equals(message) ){
                    return ;
                }else if ( "error".equals(message) ) {
                    new Thread() {
                        public void run() {
                            try {
                                System.out.println("开始睡眠。");
                                Thread.sleep(1 * 1000);
                                control++ ;
                                System.out.println("睡眠结束，control: "+ control);
                                myTask(data) ;
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        };
                    }.start();
                }else{
                    return ;
                }
            }
        });

        myTask(data) ;

    }

    @SuppressWarnings("finally")
    public void myTask(int data){
        boolean flag = true ;
        try {
            System.out.println(4 / data);
        } catch (Exception e) {
            flag = false ;
        } finally {
            if( flag ){
                throw new RuntimeException("ok");
            }else{
                throw new RuntimeException("error");
            }
        }
    }

    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        ThreadMonitor threadMonitor = new ThreadMonitor(0);
        exec.execute(threadMonitor);
        exec.shutdown();
    }
}
```

运行结果：

```bash
==Exception: error
开始睡眠。
睡眠结束，control: 1
==Exception: error
开始睡眠。
睡眠结束，control: 2
==Exception: error
开始睡眠。
睡眠结束，control: 3
==Exception: error

Process finished with exit code 0
```

此时，可以正常捕获线程因除数为零造成的中断。

其中：

(1) 在Thread类API中提供**Interface**接口UncaughtExceptionHandler，该接口包含一个uncaughtException方法，它能检测出某个由于未捕获的异常而终结的情况。定义如下：

UncaughtExceptionHandler接口:  **public static interface Thread.UncaughtExceptionHandler**

uncaughtException方法： **public void uncaughtException(Thread t, Throwable e)**

(2) uncaughtException方法会捕获线程的异常，此时需要覆写该方法设定自定义的处理方式。

(3) 设置UncaughtExceptionHandler异常处理：

方式一：通过Thread提供的静态static方法，设置**默认**异常处理：**public static void setDefaultUncaughtExceptionHandler(Thread.UncaughtExceptionHandler ux)**

方式二：通过方法：**public void setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)**

(4) UncaughtExceptionHandler异常处理需要设置在run()方法内，否则无法捕获到线程的异常。

参考链接：https://www.cnblogs.com/xiaoxing/p/6230056.html



https://www.bilibili.com/video/BV13E411N7pp?from=search&seid=14773070716069031092&spm_id_from=333.337.0.0