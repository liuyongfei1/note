## CountDownLatch

### demo

如果你的一个线程启动了多个线程来执行一些任务，且你的这个线程需要同步阻塞等待其它线程都执行完毕了，才可以继续往下走，此时就可以用CountDownLanch。

```java
public class CountDownLatchDemo {

    public static void main(String[] args){
        CountDownLatch cdt = new CountDownLatch(3);

        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2 * 1000);
                    System.out.println("线程一执行完毕");
                    cdt.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2 * 1000);
                    System.out.println("线程二执行完毕");
                    cdt.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2 * 1000);
                    System.out.println("线程三执行完毕");
                    cdt.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                try {
                    System.out.println("main线程被阻塞");
                    cdt.await();
                    System.out.println("main线程被唤醒，可继续向下执行");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
    }
}
```

执行结果：

```java
main线程被阻塞
线程一执行完毕
线程二执行完毕
线程三执行完毕
main线程被唤醒，可继续向下执行
```

### 源码

```java
public CountDownLatch(int count) {
   if (count < 0) throw new IllegalArgumentException("count < 0");
   this.sync = new Sync(count);
}

/**
 * The synchronization state.
 */
private volatile int state;
Sync(int count) {
    setState(count);
}

protected final void setState(int newState) {
    state = newState;
}
```

#### awit

```java
public void await() throws InterruptedException {
   sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  if (tryAcquireShared(arg) < 0)
    doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    for (;;) {
      final Node p = node.predecessor();
      if (p == head) {
        int r = tryAcquireShared(arg);
        if (r >= 0) {
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          failed = false;
          return;
        }
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        throw new InterruptedException();
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

此操作会将当前线程放入AQS的等待队列中，入队去等待。

此时的话，main线程会直接通过park操作挂起，阻塞住，不能干任何事儿了，就等待别人把他从队列里来唤醒。

