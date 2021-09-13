## 学习NIO应该从哪个点切入

先了解基本概念，自己写一些demo感受一下。

- buffer，channel，读写文件

NIO做网络编程是怎么玩的

NIO核心的API底层

### 基本概念

#### buffer

java虚拟机要通过nio进行磁盘读写或者网络读写的时候，java虚拟机和磁盘之间交换数据的时候，必须通过buffer这个缓冲区或中转站。支持各种不同的类型，包括ByteBuffer、ChartBuffer、DoubleBuffer、IntBuffer、FloatBuffer。

4个核心概念：capacity，limit，position，mark。

demo可以看 blog-demo工程里nio-demo模块下面。

#### 基于FileChannel将数据写入磁盘

channel，数据通道。

<img src="NIO.assets/NIO里的Channel.png" alt="NIO里的Channel" style="zoom:80%;" />

channel.write(buffer) 是直接在末尾追加写，如果要实现随机写，需要借助：

buffer.rewind(); =》buffer中的position复位

channel.position();   => 调整 channel的position

```java
public class FileChannel1 {

    public static void main(String[] args) throws Exception{

        // 构造一个传统的文件输出流
        FileOutputStream outputStream = new FileOutputStream("/Users/lyf/Downloads/nioTest/test1.txt");

        // 获取该文件输出流的FileChannel，以NIO的方式来写文件
        FileChannel channel = outputStream.getChannel();

        // 创建一个缓冲区
        ByteBuffer buffer = ByteBuffer.wrap("hello world".getBytes());

        channel.write(buffer);

        channel.close();
        outputStream.close();
    }
}

```



### 从磁盘文件中读取数据到Buffer缓冲区

```java
public class FileChannelDemo3 {

    public static void main(String[] args) throws Exception{
        FileInputStream stream = new FileInputStream("/Users/lyf/Downloads/nioTest/test1.txt");
        FileChannel channel = stream.getChannel();

        // 将数据写入buffer，所以完成后buffer的position = 16
        ByteBuffer buffer = ByteBuffer.allocateDirect(11);
        channel.read(buffer);
        // 将buffer的position复位，position = 0, limit = 16
        buffer.flip();

        // 将buffer中的数据写入到data中
        byte[] data = new byte[11];
        buffer.get(data);

        System.out.println(new String(data));

        channel.close();
        stream.close();

    }
}
```

### 利用FileChannel对文件上共享锁限制文件只读

在一个jvm内，是可以通过多个线程共用一个FileChannel来写磁盘文件，是线程安全的。

那如果是多个jvm呢，此时就没办法保证多线程按照顺序来写文件了，并发写，可能会出问题。

FileChannel提供了文件锁，分为共享锁和独占锁。

#### 共享锁

如果你对文件上了一个共享锁，此时你可以读文件，别人也可以上共享锁，也可以读文件，但是包括你在内，没有人可以写文件。

```java
public static void main(String[] args) throws Exception {
		FileInputStream in = new FileInputStream("F:\\development\\tmp\\hello.txt");
		FileChannel channel = in.getChannel();
		
		channel.lock(0, Integer.MAX_VALUE, true);
		Thread.sleep(60 * 60 * 1000);  
		
		channel.close();
		in.close();
	}
```

#### OS CACHE

利用FileChannel写数据时，不是立即写入磁盘，需要先写入到 OS CACHE。操作系统会不定时的将数据从OS CACHE写入到磁盘文件里。

可以使用force，强制将数据写入到磁盘。

```java
public static void main(String[] args) throws Exception {
		FileInputStream in = new FileInputStream("F:\\development\\tmp\\hello.txt");
		FileChannel channel = in.getChannel();
		
		channel.lock(0, Integer.MAX_VALUE, true);
    // 强制将channel对应的文件直接写入到磁盘文件，避免停留在OS Cache。
		channel.force(true);
		
		channel.close();
		in.close();
	}
```

#### NIO核心原理剖析之FileChannel的初始化

```java
 // 构造一个传统的文件输出流
FileOutputStream outputStream = new FileOutputStream("/Users/lyf/Downloads/nioTest/test1.txt");

// 获取该文件输出流的FileChannel，以NIO的方式来写文件
FileChannel channel = outputStream.getChannel();
```

sun.nio.ch.FileChannelImpl 这个类，是非常底层的一个类，是jdk内核里的东西，所以对这块源码就没必要深究了。

FileOutputStream类：

```java
public FileChannel getChannel() {
        synchronized (this) {
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, false, true, append, this);
            }
            return channel;
        }
    }
```

可以看到，是基于synchronized做的线程同步，他可以保证多个线程调用同一个的getChannel()方法的时候，是可以保证多线程并发的安全。

#### FileChannel数据写入到磁盘

FileChannel有一个position的概念。指向了当前文件的某个位置，写数据的时候是从当前position的位置开始写的，写入文件之后，文件会不断变大，写了多少byte的数据，文件的position就会向前推移多少位。

