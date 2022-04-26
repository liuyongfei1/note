## 学习NIO应该从哪个点切入

#### 传统的Socket编程存在的问题

可以看 这些案例：/Users/lyf/Workspace/www/blog-demo/nio-demo/src/main/java/SocketServer.java

在这种模式下，假如你接受的客户端的请求是百万级的，此时在你的服务端就需要构建百万个线程来支撑百万个客户端。

一台普通的4核8G的服务器，虚拟机，开启100个线程，就已经达到极限了，CPU的负载就会很高了。

#### NIO

BIO =》同步阻塞式IO，简单理解：一个线程处理一个连接，发起和处理IO请求都是同步的；
NIO  =》同步非阻塞式IO，简单理解：一个请求处理多个连接，发起IO请求是非阻塞的但处理IO请求是同步的；
AIO =》异步非阻塞式IO，简单理解：一个有效请求一个线程，发起和处理IO请求都是异步的。

先了解基本概念，自己写一些demo感受一下。

- buffer，channel，读写文件

NIO做网络编程是怎么玩的

NIO核心的API底层

#### NIO是如何基于多路复用技术支持海量客户端的

NIO里就三大核心组件：Buffer、Channel、Selector。

##### Buffer

其中Buffer是封装数据的，主要是从buffer写入数据，或者将数据写入buffer。

##### Channel

Channel是一个数据管道，负责数据传输。

##### Selector

最后一个是Selector，多路复用组件，专门在网络编程里用到。

不需要给每个socket连接都创建一个线程了，完全可以用一个Selector来多路复用监听N多个Channel是否有请求，这里是基于操作系统底层的select通知机制实现的。

这样可以避免创建大量的线程。

如果是通过传统的socket编程来解决这个问题，可能比如说，1000个客户端需要1000个线程，但是此时通过NIO网络编程模型来实现的，线程池里10个线程，就可以完美的处理这1000个客户端的请求了。

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

#### NIO核心原理之select发现通知机制

channel注册给select，如果channel注册上后，就会一直在等：

selector.select();

卡在这里。

然后如果当selector监听的channel有请求时，就会反向通知到selector。

linux select机制，可以让一个线程监听多个socket有否有请求发送过来。不同的操作系统，select机制是不一样的。

select()方法，他会感知哪些的channel底层的socket有请求进来了，可以执行IO操作来读取数据或者发送数据出去了，他就会把那些channel对应的key的集合给你返回回来了。如果没感知到哪个channel对应的socket可以执行读写操作的话，就会一直卡着，阻塞着。

#### NIO核心原理之如何通过Channel三次握手建立连接

1. 建立连接拿到一个SocketChannel，默认阻塞模式，改为非阻塞模式；

   ServerSocketChannel的accept()方法，就可以跟客户端进行tcp三次握手。如果三次握手成功了，就可以获取到一个建立好TCP连接的SocketChannel。

2. 将SocketChannel注册到selector上，并告诉select，你对这个channel感兴趣的key是什么，是要通过这个Channel写数据呢，还是要通过这个Channel读数据；

​      channel.register(selector,SelectionKey.OP_READ)

1. 如果要读数据，则必须等到客户端发数据，selector会返回这个channel对应的selectionKey给你；
2. 如果要通过Channel写数据，只要Channel ready以后，selector会返回这个channel对应的selectionKey给你。

```java
public class NIOServer {

    private static Selector selector;  
    private static LinkedBlockingQueue<SelectionKey> requestQueue;
    private static ExecutorService threadPool;
       
    public static void main(String[] args) {
    	init();
    	listen();
    }  
       
    private static void init(){  
        ServerSocketChannel serverSocketChannel = null;  
           
        try {  
        	selector = Selector.open();
        	
        	serverSocketChannel = ServerSocketChannel.open();  
        	serverSocketChannel.configureBlocking(false);  // NIO就是支持非阻塞的，项目实战的时候给大家来讲
        	serverSocketChannel.socket().bind(new InetSocketAddress(9000), 100);    
        	// ServerSocket，就是负责去跟各个客户端连接连接请求的  
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);  
            // 就是仅仅关注这个ServerSocketChannel接收到的TCP连接的请求
        } catch (IOException e) {  
            e.printStackTrace();  
        }        

        requestQueue = new LinkedBlockingQueue<SelectionKey>(500);
        
        threadPool = Executors.newFixedThreadPool(10);
        for(int i = 0; i < 10; i++) {
        	threadPool.submit(new Worker());  
        }
    }  
    
	private static void listen() {  
        while(true){  
            try{  
                selector.select();               
                Iterator<SelectionKey> keysIterator = selector.selectedKeys().iterator();  
                   
                while(keysIterator.hasNext()){  
                    SelectionKey key = (SelectionKey) keysIterator.next();  
                    // 可以认为一个SelectionKey是代表了一个请求                  
                    keysIterator.remove();
//                    requestQueue.offer(key);
                    handleRequest(key);
                }  
            }  
            catch(Throwable t){  
                t.printStackTrace();  
            }                            
        }                
    }  
	
	private static void handleRequest(SelectionKey key)  
            throws IOException, ClosedChannelException {  
    	// 假设想象一下，后台有个线程池获取到了请求
    	// 下面的代码，都是在后台线程池的工作线程里在处理和执行
        SocketChannel channel = null;  
           
        try{  
            if(key.isAcceptable()){  // 如果说这个key是个acceptable，是个连接请求的话
                ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();  
                channel = serverSocketChannel.accept(); // 调用这个accept方法，就可以跟客户端进行TCP三次握手
                // 如果三次握手成功了之后，就可以获取到一个建立好TCP连接的SocketChannel
                // 这个SocketChannel大概可以理解为，底层有一个Socket，是跟客户端进行连接的
                // 你的SocketChannel就是联通到那个Socket上去，负责进行网络数据的读写的
                channel.configureBlocking(false);  
                channel.register(selector, SelectionKey.OP_READ);  // 仅仅关注这个READ请求，就是人家发送数据过来的请求
            }  // 如果说这个key是readable，是个发送了数据过来的话，此时需要读取客户端发送过来的数据
            else if(key.isReadable()){  
                channel = (SocketChannel) key.channel();  
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int count = channel.read(buffer);  // 通过底层的socket读取数据，写入buffer中，position可能就会变成21之类的
                // 你读取到了多少个字节，此时buffer的position就会变成多少
                   
                if(count > 0){    
                	buffer.flip();   // position = 0，limit = 21，仅仅读取buffer中，0~21这段刚刚写入进去的数据
                    System.out.println("服务端接收请求：" + new String(buffer.array(), 0, count));   
                    channel.register(selector, SelectionKey.OP_WRITE);
                }                       
            } else if(key.isWritable()) {
            	ByteBuffer buffer = ByteBuffer.allocate(1024);
            	buffer.put("收到".getBytes());
            	buffer.flip();
            	
            	channel = (SocketChannel) key.channel();
            	channel.write(buffer);       
            	channel.register(selector, SelectionKey.OP_READ);
            }
        }  
        catch(Throwable t){  
            t.printStackTrace();  
            if(channel != null){  
                channel.close();  
            }  
        }        
    }
	
	static class Worker implements Runnable {
		
		@Override
		public void run() {
			while(true) {
				try {
					SelectionKey key = requestQueue.take();
					handleRequest(key);
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		}
		
		private void handleRequest(SelectionKey key)  
	            throws IOException, ClosedChannelException {  
	    	// 假设想象一下，后台有个线程池获取到了请求
	    	// 下面的代码，都是在后台线程池的工作线程里在处理和执行
	        SocketChannel channel = null;  
	           
	        try{  
	            if(key.isAcceptable()){  // 如果说这个key是个acceptable，是个连接请求的话
	            	System.out.println("[" + Thread.currentThread().getName() + "]接收到连接请求");        
	                ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();  
	                channel = serverSocketChannel.accept(); // 调用这个accept方法，就可以跟客户端进行TCP三次握手
	                System.out.println("[" + Thread.currentThread().getName() + "]建立连接时获取到的channel=" + channel);     
	                // 如果三次握手成功了之后，就可以获取到一个建立好TCP连接的SocketChannel
	                // 这个SocketChannel大概可以理解为，底层有一个Socket，是跟客户端进行连接的
	                // 你的SocketChannel就是联通到那个Socket上去，负责进行网络数据的读写的
	                channel.configureBlocking(false);  
	                channel.register(selector, SelectionKey.OP_READ);  // 仅仅关注这个READ请求，就是人家发送数据过来的请求
	            }  // 如果说这个key是readable，是个发送了数据过来的话，此时需要读取客户端发送过来的数据
	            else if(key.isReadable()){  
	                channel = (SocketChannel) key.channel();  
	                ByteBuffer buffer = ByteBuffer.allocate(1024);
	                int count = channel.read(buffer);  // 通过底层的socket读取数据，写入buffer中，position可能就会变成21之类的
	                // 你读取到了多少个字节，此时buffer的position就会变成多少
	                   
	                System.out.println("[" + Thread.currentThread().getName() + "]接收到请求");        
	                
	                if(count > 0){    
	                	buffer.flip();   // position = 0，limit = 21，仅仅读取buffer中，0~21这段刚刚写入进去的数据
	                    System.out.println("服务端接收请求：" + new String(buffer.array(), 0, count));   
	                    channel.register(selector, SelectionKey.OP_WRITE);
	                }                       
	            } else if(key.isWritable()) {
	            	ByteBuffer buffer = ByteBuffer.allocate(1024);
	            	buffer.put("收到".getBytes());
	            	buffer.flip();
	            	
	            	channel = (SocketChannel) key.channel();
	            	channel.write(buffer);       
	            	channel.register(selector, SelectionKey.OP_READ);
	            }
	        }  
	        catch(Throwable t){  
	            t.printStackTrace();  
	            if(channel != null){  
	                channel.close();  
	            }  
	        }        
	    }
		
	}
	
}
```

#### NIO核心原理之如何通过Channel读取数据

通过NIO和NIO里的channel和buffer这套机制来读取数据。

通过channel把数据读出来，然后写到缓冲区内。

接收到人家发送过来的数据或者请求之后，你应该如何来处理呢？

这里讲一下协议的概念。

TCP属于传输层协议，IP是网络层协议，以太网是链路层协议，在TCP之上，你接收到TCP给你传过来的数据只会，实际上应该有一个应用层的协议。如果是HTTP协议的话，人家给你发送的数据可能是下面这样的格式：

```
GET HTTP://1.1  home/do.jsp
Accept: xxxx
Last-Modified: xxx
{
   id: 1,
   name: xxx
}
```

此时你就需要按照应用层的协议，无论是HTTP协议，还是你自定义的协议，你就可以按照固定的格式来解析消息，进行处理。

处理完了请求之后，就需要发送响应给人家客户端，此时你可以重新注册一下Channel，把你对这个channel感兴趣的操作变成：OP_WRITE，什么时候就绪可以让我写，然后while true，又会执行 selector.select()，卡住。

#### NIO核心原理之如何通过Channel发送数据

如果当前socketChannel的状态是ok的，没有什么异常情况，那么就可以让你写数据发送出去，selector.select()就会从阻塞状态恢复过来，把对应的selectionKey告诉你，就会执行这段代码：

```java
 else if(key.isWritable()) {
	            	ByteBuffer buffer = ByteBuffer.allocate(1024);
	            	buffer.put("收到".getBytes());
                // 将position改为0，limit调成写到position的那个地方
	            	buffer.flip();
	            	
	            	channel = (SocketChannel) key.channel();
	            	channel.write(buffer);       
	            	channel.register(selector, SelectionKey.OP_READ);
	            }
```

将数据放入buffer里。此时仍然应该按照设计好的应用层的协议，比如说HTTP协议：

```
200
Gzip:xxx
Accept:xxx
// <html>
// 一大段的html代码
// </html>
```

将数据发送出去。

我现在已经发送响应回去给客户端了，接下来再次重新注册这个channel。感兴趣的是 READ事件，这时就又卡在

serverSocketChannel.accept();这里。如果又感知到客户端又给你发送数据了，这时就走READ的逻辑。





本次 Chat 的主要内容（链接：https://blog.csdn.net/valada/article/details/96040288）：

1. 什么是BIO、NIO、AIO 各自的原理；
   AIO现在用的比较少，用的比较多的还是NIO。

   AIO的网络通信原理与NIO基本是一样的，不一样的地方就是工作线程读取数据的时候，是说你提供给操作系统一个buffer，空的，然后你就可以干别的事儿了，你就把读取数据的事儿交给操作系统内核去干，操作系统内核读取到数据将数据放入buffer中，`然后回调你的一个接口`，告诉你说，ok，buffer交给你了，这个数据我给你读好了。
   https://apppukyptrl1086.pc.xiaoe-tech.com/detail/v_5e440e1daaefe_QvMnpQeT/3?from=p_5dd3ccd673073_9LnpmMju&type=6&parent_pro_id=

2. 什么是同步阻塞、同步非阻塞、异步非阻塞；

   

3. 多路复用机制是什么；

4. NIO 相关组件的含义理解。

#### BIO同步阻塞

https://apppukyptrl1086.pc.xiaoe-tech.com/detail/v_5e440e1daaefe_QvMnpQeT/3?from=p_5dd3ccd673073_9LnpmMju&type=6&parent_pro_id=
为什么BIO是同步阻塞呢？这里其实不是针对网络编程模型来说的，而是针对文件IO操作来说的，因为用BIO的流读写文件，是说你发起个IO请求后，会被直接卡死，必须等着处理完这次IO才能返回。

#### NIO同步非阻塞

`就是说通过NIO的FileChannel发起这个文件IO操作，其实发起之后就返回了，你可以干别的事儿了，这就是非阻塞。`

`为什么叫同步呢？就是说你在干其它事儿的同时，还得时不时的去主动问操作系统，数据处理好了没 （你还得不断的去轮询操作系统读取数据的状态，看看人家读好了没有。）`

#### AIO异步非阻塞

你也可以基于AIO的文件读写api去读写磁盘文件，你发起一个文件读写的操作之后，你就不用管他了，直到操作系统自己完成之后，会来回调你的一个接口，通知你说：ok，这个数据读完了。

--------------------------------------------



你的收获：

1. 彻底理解 BIO、NIO、AIO 各自的区别以及优缺点；
2. 理解多路复用机制使用场景；
3. 熟练掌握 NIO。

适用人群： Java NIO 初学者，正在准备 Java 面试的朋友。

