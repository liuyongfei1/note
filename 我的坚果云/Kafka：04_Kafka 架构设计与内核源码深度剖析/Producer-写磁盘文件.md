### 写磁盘文件

#### 稀疏索引

Broker 端参数` log.index.interval.bytes` 值，默认4KB，即4KB的消息建一条索引。

往指定的磁盘目录下面写文件，类似这种格式，topic名称-分区序号目录（比如：order-0目录）下面会有这样的文件：

****.log

****.index

每次在****.log里写入了 4096字节的数据，就会同时在****.index中创建一个稀疏索引，类似这样：

****.index：

offset = 23345 物理位置 380

offset = 23670 物理位置 421

offset = 24137 物理位置 495

****.log：

asdadasda offset=23765 ...



也就是说在****.index中，每隔一段时间会将数据的逻辑索引与物流位置进行对应。

假设要查找的是offset=23689的这条数据：

1. 就会通过二分查找，先定位到这条数据在offset = 23345和offset = 23670之间，那么就会定位到物理位置为380；
2. 直接去.log文件中定位到物理位置为380这里往后扫，可能扫几条数据后就找到了offset=23765这条数据。

#### 基于mmap机制实现的索引文件内存映射以及内存写入

MappedByteBuffer如何将.index文件映射到内存里，这样向磁盘写文件其实是在向内存写文件，这样效率就会很高了。

#### FileChannel优先写入OS Cache

以后基于NIO写磁盘文件时，需要写一个while循环，要不停的调用write方法，直到把ByteBuffer中的数据全部写完为止。

基于FileChannel从ByteBuffer中读取数据，然后写入OS Cache（操作系统管理的内存），然后每隔一段时间，将OS Cache中的数据写入磁盘。

