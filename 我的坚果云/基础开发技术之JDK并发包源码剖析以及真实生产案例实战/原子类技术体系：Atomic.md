### i++和AtomicInteger之间的差别分析

```java
public class AtomicIntegerDemo {

    private static long i = 0;

    private static AtomicInteger j = new AtomicInteger(0);

    public static void main(String[] args) {
//        synchronizedAdd();
        atomicIntegerAdd();
    }

    public static void synchronizedAdd() {
        for (int i = 0; i < 10; i++) {
            new Thread() {
                @Override
                public void run() {
                    synchronized (AtomicIntegerDemo.class) {
                        System.out.println(AtomicIntegerDemo.i++);
                    }
                }
            }.start();
        }
    }

    public static void atomicIntegerAdd() {
        for (int i = 0; i < 10; i++) {
            new Thread() {
                @Override
                public void run() {
                    System.out.println(AtomicIntegerDemo.j.incrementAndGet());
                }
            }.start();
        }
    }
}
```

Atomic一些列的类可以来优化加锁的性能。

### Atomic原理图

<img src="原子类技术体系：AutomicInteger.assets/Atomic原理.png" alt="Atomic原理" style="zoom:50%;" />

Atomic原子类体系的底层核心的原理是CAS。Compare And Set。

无锁化。

- 每次尝试修改值的时候，就对比一下，有没有人修改过这个值，没有人修改的话，就自己修改；
- 如果有人修改过，就重新查出来最新的值；
- 再次重复这个过程。

#### 源码分析

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

- 这段代码是在类初始化的时候执行的；
- valueOffset：可以理解为value这个字段在AtomicInteger这个类中的位置。在底层，这个类是有自己对应的结构的，无论是在磁盘的.class文件里，还是在JVM内存中；
- 在类初始化的时候，就会完成这个操作，final的，一旦初始化完毕，就不会再变更了。

```java
/**
 * Atomically increments by one the current value.
 *
 * @return the updated value
 */
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

- var1：当前的AtomicInteger对象实例；

- var2：valueOffset，就是那个位置偏移量；

- var5：就是刚刚根据getIntVolatile()方法拿到的值，比如i；

- var5 + var4：i + 1；

- getIntVolatile方法：从AtomicInteger对象实例，通过valueOffset偏移量，知道了value这个字段的位置，去获取当前value的值；

- compareAndSwapInt方法：这个就是CAS方法，比较底层了，会有一些指针的操作。他会拿你刚刚获取到的那个值，去跟当前AtomicInteger实例中的value值进行比较：

  

  - 如果是一样的话，就会执行set的过程，将value的值递增+1；
  - 如果不一样的话，此时compareAndSwapInt就会返回false；
  - 如果while循环里拿到的是false，就会自动进入下一轮循环。
  - 如果成功的话，就会返回一个i的值，然后incrementAndGet()方法里会加1，得到当前累加1后的最新的值。

  #### AtomicInteger源码分析图

  <img src="原子类技术体系：Atomic.assets/AtomicInteger源码分析.png" alt="AtomicInteger源码分析" style="zoom:50%;" />

