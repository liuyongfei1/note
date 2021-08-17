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