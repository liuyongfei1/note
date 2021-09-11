## 学习NIO应该从哪个点切入

先了解基本概念，自己写一些demo感受一下。

- buffer，channel，读写文件

NIO做网络编程是怎么玩的

NIO核心的API底层

### 基本概念

#### buffer

java虚拟机要通过nio进行磁盘读写或者网络读写的时候，java虚拟机和磁盘之间交换数据的时候，必须通过buffer这个缓冲区或中转站。