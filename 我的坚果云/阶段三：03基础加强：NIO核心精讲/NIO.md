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

#### FileChannel

channel，数据通道。

<img src="NIO.assets/NIO里的Channel.png" alt="NIO里的Channel" style="zoom:80%;" />

channel.write(buffer) 是直接在末尾追加写，如果要实现随机写，需要借助：

buffer.rewind(); =》buffer中的position复位

channel.position();   => 调整 channel的position

### 从磁盘文件中读取数据到Buffer缓冲区

