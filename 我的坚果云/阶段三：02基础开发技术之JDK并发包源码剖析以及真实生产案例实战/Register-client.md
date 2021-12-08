### 微服务的服务注册案例

#### 知识点

1. 创建线程有几种方法；
2. Daemon线程和非Daemon线程；
3. 线程join()方法和interrupt()方法

##### join()方法

main线程里开启了一个线程，如果对这个线程启用了join方法，那么就会导致main线程阻塞住，main线程会等待这个线程执行完毕后，才会继续往下走。

##### interrupt()方法

- interrupt()方法打断一个线程，其实就是修改线程里的一个interrupt标志位，打断以后，interrupt的标志位就变为true，这时可以在线程内部，根据这个标志位isInterrupted，去判断要不要继续执行。
- 并不是说直接interrupt一下就不让线程执行了。

如果系统要尽快停止，那么就应该应用interrupt打断各个工作线程的休眠，让他们判断是否运行标志位为false，就立刻退出。

实战可以看：

微服务注册中心的register-client服务中，为RegisterClient提供了一个优雅关闭心跳线程的方法：

```java
/**
 * 关闭registerClient组件
 */
public void shutdown() {
    this.finishedRunning = false;
    heartbeatWorker.interrupt();
}

/**
     * 心跳线程
     */
    private class HeartbeatWorker extends Thread {
        @Override
        public void run() {
            HeartbeatRequest request = new HeartbeatRequest();
            request.setServerInstanceId(serviceInstanceId);

            HeartbeatResponse response = null;
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

一旦执行 .shutdown()方法：

- 则将finishedRunning置为false；
- 并调用interrupt()方法；
- 然后会打断正在休眠的Heartbeat线程，然后继续执行while死循环时，由于finishedRunning已经被修改为false，因此就退出执行。然后整个jvm进程就会退出了。

