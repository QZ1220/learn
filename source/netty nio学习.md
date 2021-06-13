# netty nio学习 

标签（空格分隔）： 线程模型 零拷贝

---

参考链接
----
https://segmentfault.com/a/1190000011883632

https://www.cnblogs.com/200911/articles/10432551.html

零拷贝
---
所谓**零拷贝**，其实就是减少了数据在用户态与核心态之间的拷贝次数。

 - 传统方式实现文件传输

传统方式的文件传输的一般代码如下：
```java
File.read(file, buf, len);
Socket.send(socket, buf, len);
```
上面的代码，看似简单，实际会进行4次用户态--核心态的上下文切换，以及4次数据拷贝。

4次上下文切换过程如下入所示：

![此处输入图片的描述][1]

4次数据拷贝过程：
![此处输入图片的描述][2]

![此处输入图片的描述][3]

其中，切换的步骤如下：

下面的文字来自于https://segmentfault.com/a/1190000011883632

 1. **read()** 调用导致了一次用户态到内核态的上下文切换，在内部，一个 **sys_read()**（或等价函数）被执行来从文件中读取数据。第一次拷贝是由 DMA 引擎将数据从磁盘文件存储到内核地址空间缓冲区。
 2. 被请求长度的数据从内核的读缓冲区拷贝到用户缓冲区，并且 **read()**调用返回。这个返回导致又一次从内核态到用户态的上下文切换。现在数据是存储在用户地址空间缓冲区。
 3. **send()**调用引起了一次从用户态到内核态的上下文切换。第三次拷贝又一次将数据放进内核地址空间缓冲区，尽管这一次是放进另一个不同的缓冲区，和目标socket联系在一起。
 4. **send()** 系统调用返回，产生了第四次上下文切换。第四次拷贝由 DMA 引擎独立异步地将数据从内核缓冲区传递给协议引擎。

下面的文字来自于https://www.cnblogs.com/200911/articles/10432551.html

1.首先，调用read方法，文件从user模式拷贝到了kernel模式；（用户模式->内核模式的上下文切换，在内部发送sys_read() 从文件中读取数据，存储到一个内核地址空间缓存区中）

2.之后CPU控制将kernel模式数据拷贝到user模式下；（内核模式-> 用户模式的上下文切换，read()调用返回，数据被存储到用户地址空间的缓存区中）

3.调用write时候，先将user模式下的内容copy到kernel模式下的socket的buffer中（用户模式->内核模式，数据再次被放置在内核缓存区中，send（）套接字调用）

4.最后将kernel模式下的socket buffer的数据copy到网卡设备中；（send套接字调用返回）

**read()** 函数为什么不**直接**将数据拷贝到用户地址空间的缓冲区，而要经内核地址空间的缓冲区转一次手，这不是白白多了一次拷贝操作吗？

对IO函数有了解的童鞋肯定知道，在IO函数的背后有一个缓冲区 buffer ，我们平常的读和写操作并不是直接和底层硬件设备打交道，而是通过一块叫缓冲区的内存区域缓存数据来间接读写。我们知道，和CPU、高速缓存、内存比，磁盘、网卡这些设备属于慢速设备，交换一次数据要花很多时间，同时会消耗总线传输带宽，所以我们要尽量降低和这些设备打交道的频率，而使用缓冲区中转数据就是为了这个目的。（也就是说，缓冲区的存在是很重要的）

从图中看2，3两次copy是多余的，数据从kernel模式到user模式走了一圈，浪费了2次copy。

 - 零拷贝实现
上面的步骤中，2、3步是多余的。数据可以直接从read buffer 读缓存区传输到套接字缓冲区，也就是省去了将操作系统的read buffer 拷贝到程序的buffer，以及从程序buffer拷贝到socket buffer的步骤，直接将read buffer拷贝到socket buffer。JDK NIO中的的transferTo() 方法就能够让您实现这个操作，这个实现依赖于操作系统底层的sendFile（）实现的：
```java
public void transferTo(long position, long count, WritableByteChannel target);
```
在UNIX/Linux系统上， transferTo() 实际会调用 sendfile() 这个系统函数，将数据从一个文件描述符传输到另一个。
```c++
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
![此处输入图片的描述][4]

![此处输入图片的描述][5]

transferTo()方法，整个过程如下：

1.transferTo()方法使得文件的内容直接copy到了一个read buffer（kernel buffer）中

2.然后数据（kernel buffer）copy到socket buffer中

3.最后将socket buffer中的数据copy到网卡设备（protocol engine）中传输；

这个显然是一个伟大的进步：这里上下文切换从4次减少到2次，同时把数据copy的次数从4次降低到3次.只有1次 CPU 参与，另外2次 DMA 引擎完成），那么我们可不可以把这唯一一次CPU参与的数据拷贝也省掉呢？

答案是肯定的。

如果网卡支持 gather operations 内核就可以进一步减少数据拷贝。在 Linux kernels 2.4 及更新的版本，socket 描述符已经为适应这个需求做了变化。现在这个方法不仅减少了上下文切换，而且消除了CPU参与的数据拷贝。API接口是一样的，但是实质已经发生了变化：

![此处输入图片的描述][6]

改进后的transferTo()的数据拷贝步骤如下：

 1. transferTo() 方法引起 DMA 引擎将文件内容拷贝到内核缓冲区。
 2. 没有数据从内核缓冲区拷贝到socket缓冲区，只有携带位置和长度信息的描述符被追加到socket缓冲区上， DMA引擎直接将内核缓冲区的数据传递到协议引擎，全程无需CPU拷贝数据。

Zero-Copy技术的使用场景有很多，比如Kafka, 又或者是Netty等，可以大大提升程序的性能。

线程模型
----

参考链接：

 1. https://www.infoq.cn/article/netty-threading-model
 2. https://www.jianshu.com/p/38b56531565d

个人认为，链接2讲解的比较通俗易懂一点。其实，简单来说，netty的线程模型，就是参考reactor线程模型，将网络请求的监听，以及后续的处理，分成两个单独的线程池进行出路，同时处理的过程中，采用线程串行处理的方式，避免了线程的上下文的切换，从而进一步的提升了性能。

下面将链接2的大部分内容摘抄如下：

 1. NIO Reactor模型

一个连接里完整的网络处理过程一般分为accept、read、decode、process、encode、send这几步。

Reactor模式将每个步骤映射为一个Task，服务端线程执行的最小逻辑单元不再是一次完整的网络请求，而是Task，且采用非阻塞方式执行。

每个Task对应特定网络事件。当Task准备就绪时，Reactor收到对应的网络事件通知，并将Task分发给绑定了对应网络事件的Handler执行。

Reactor：负责响应事件，将事件分发给绑定了该事件的Handler处理；

Handler：事件处理器，绑定了某类事件，负责执行对应事件的Task对事件进行处理；

Acceptor：Handler的一种，绑定了connect事件。当客户端发起connect请求时，Reactor会将accept事件分发给Acceptor处理。

 - 单线程Reactor

![此处输入图片的描述][7]

这种模型，结构简单，实现也较为容易，因为不需要进行并发点控制，但是**缺点**也是显而易见的，因为单线程必然无法发挥多核处理器点优势，必然的随着访问请求的增长，肯定无法满足性能要求。一旦**reactor线程**意外跑飞或者进入死循环，会导致整个系统通信模块不可用。

 - 多线程Reactor

![此处输入图片的描述][8]

如上图所示，相对于单线程reactor，其实就是将网络请求链接的  建立  处理  分开来。

a)有专门一个reactor线程用于监听服务端ServerSocketChannel，接收客户端的TCP连接请求；

b)网络IO的读/写操作等由一个worker reactor线程池负责，由线程池中的NIO线程负责监听SocketChannel事件，进行消息的读取、解码、编码和发送。

c)一个NIO线程可以同时处理N条链路，但是一个链路只注册在一个NIO线程上处理，防止发生并发操作问题。

 - 主从多线程

![此处输入图片的描述][9]

其实，参考文档里，把这个称为主从多线程，个人觉其实就是acceptor和handler都变成了多线程。

a)服务端用于接收客户端连接的不再是个1个单独的reactor线程，而是一个boss reactor线程池；

b)服务端启用多个ServerSocketChannel监听不同端口时，每个ServerSocketChannel的监听工作可以由线程池中的一个NIO线程完成。


Netty线程模型
---------
![此处输入图片的描述][10]

netty线程模型采用“服务端监听线程”和“IO线程”分离的方式，与多线程Reactor模型类似。

抽象出NioEventLoop来表示一个不断循环执行处理任务的线程，每个NioEventLoop有一个selector，用于监听绑定在其上的socket链路。

1、串行化设计避免线程竞争
netty采用串行化设计理念，从消息的读取->解码->处理->编码->发送，始终由IO线程NioEventLoop负责。整个流程不会进行线程上下文切换，数据无并发修改风险。

一个NioEventLoop聚合一个多路复用器selector，因此可以处理多个客户端连接。

netty只负责提供和管理“IO线程”，其他的业务线程模型由用户自己集成。

时间可控的简单业务建议直接在“IO线程”上处理，复杂和时间不可控的业务建议投递到后端业务线程池中处理。

2、定时任务与时间轮
NioEventLoop中的Thread线程按照时间轮中的步骤不断循环执行：

a)在时间片Tirck内执行selector.select()轮询监听IO事件；

b)处理监听到的就绪IO事件；

c)执行任务队列taskQueue/delayTaskQueue中的非IO任务。

NioEventLoop与NioChannel类关系
--------------------------
![此处输入图片的描述][11]

 注意图中点箭头指向
 
 一个NioEventLoopGroup下包含多个NioEventLoop

每个NioEventLoop中包含有一个Selector，一个taskQueue，一个delayedTaskQueue

每个NioEventLoop的Selector上可以注册监听多个AbstractNioChannel.ch

每个AbstractNioChannel只会绑定在唯一的NioEventLoop上

每个AbstractNioChannel都绑定有一个自己的DefaultChannelPipeline

NioEventLoop线程执行过程
------------------
1、轮询监听的IO事件
1）netty的轮询注册机制

netty将AbstractNioChannel内部的jdk类SelectableChannel对象注册到NioEventLoopGroup中的jdk类Selector对象上去，并且将AbstractNioChannel作为SelectableChannel对象的一个attachment附属上。

这样在Selector轮询到某个SelectableChannel有IO事件发生时，就可以直接取出IO事件对应的AbstractNioChannel进行后续操作。

2）循环执行阻塞selector.select(timeoutMIllis)操作直到以下条件产生

a)轮询到了IO事件（selectedKey != 0）

b)oldWakenUp参数为true

c)任务队列里面有待处理任务（hasTasks()）

d)第一个定时任务即将要被执行（hasScheduledTasks()）

e)用户主动唤醒（wakenUp.get()==true）

3）解决JDK的NIO epoll bug

该bug会导致Selector一直空轮询，最终导致cpu 100%。

在每次selector.select(timeoutMillis)后，如果没有监听到就绪IO事件，会记录此次select的耗时。如果耗时不足timeoutMillis，说明select操作没有阻塞那么长时间，可能触发了空轮询，进行一次计数。

计数累积超过阈值（默认512）后，开始进行Selector重建：

a)拿到有效的selectionKey集合

b)取消该selectionKey在旧的selector上的事件注册

c)将该selectionKey对应的Channel注册到新的selector上，生成新的selectionKey

d)重新绑定Channel和新的selectionKey的关系

4）netty优化了sun.nio.ch.SelectorImpl类中的selectedKeys和publicSelectedKeys这两个field的实现

netty通过反射将这两个filed替换掉，替换后的field采用数组实现。

这样每次在轮询到nio事件的时候，netty只需要O(1)的时间复杂度就能将SelectionKey塞到set中去，而jdk原有field底层使用的hashSet需要O(lgn)的时间复杂度。

2、处理IO事件
1）对于boss NioEventLoop来说，轮询到的是基本上是连接事件（OP_ACCEPT）：

a)socketChannel = ch.accept()；

b)将socketChannel绑定到worker NioEventLoop上；

c)socketChannel在worker NioEventLoop上创建register0任务；

d)pipeline.fireChannelReadComplete();

2）对于worker NioEventLoop来说，轮询到的基本上是IO读写事件（以OP_READ为例）：

a)ByteBuffer.allocateDirect(capacity)；

b)socketChannel.read(dst);

c)pipeline.fireChannelRead();

d)pipeline.fireChannelReadComplete();

3、处理任务队列
1）处理用户产生的普通任务

NioEventLoop中的Queue<Runnable> taskQueue被用来承载用户产生的普通Task。

taskQueue被实现为netty的mpscQueue，即多生产者单消费者队列。netty使用该队列将外部用户线程产生的Task聚集，并在reactor线程内部用单线程的方式串行执行队列中的Task。

当用户在非IO线程调用Channel的各种方法执行Channel相关的操作时，比如channel.write()、channel.flush()等，netty会将相关操作封装成一个Task并放入taskQueue中，保证相关操作在IO线程中串行执行。

2）处理用户产生的定时任务

NioEventLoop中的Queue<ScheduledFutureTask<?>> delayedTaskQueue = new PriorityQueue被用来承载用户产生的定时Task。

当用户在非IO线程需要产生定时操作时，netty将用户的定时操作封装成ScheduledFutureTask，即一个netty内部的定时Task，并将定时Task放入delayedTaskQueue中等待对应Channel的IO线程串行执行。

为了解决多线程并发写入delayedTaskQueue的问题，netty将添加ScheduledFutureTask到delayedTaskQueue中的操作封装成普通Task，放入taskQueue中，通过NioEventLoop的IO线程对delayedTaskQueue进行单线程写操作。

3）处理任务队列的逻辑

a)将已到期的定时Task从delayedTaskQueue中转移到taskQueue中

b)计算本次循环执行的截止时间

c)循环执行taskQueue中的任务，每隔64个任务检查一下是否已过截止时间，直到taskQueue中任务全部执行完或者超过执行截止时间。

Netty中Reactor线程和worker线程所处理的事件
------------------------------

 1. Server端NioEventLoop处理的事件
![此处输入图片的描述][12]
  2. Client端NioEventLoop处理的事件
![此处输入图片的描述][13]
  


  [1]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/%E4%BC%A0%E7%BB%9F%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2.png?raw=true
  [2]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/%E4%BC%A0%E7%BB%9F%E7%9A%84%E6%95%B0%E6%8D%AE%E6%8B%B7%E8%B4%9D%E6%96%B9%E6%B3%95.png?raw=true
  [3]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/%E4%BC%A0%E7%BB%9F%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A22.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/%E9%9B%B6%E6%8B%B7%E8%B4%9D%E4%BC%A0%E8%BE%93%E6%96%B9%E6%B3%95.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/transferToContext.png?raw=true
  [6]: https://github.com/Audi-A7/learn/blob/master/source/image/netty/zeroCopy.png?raw=true
  [7]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/single_reactor.webp?token=ABXI2UM2N7Q3RT454DKR3IC5QOHFS
  [8]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/multi_thread_reactor.webp?token=ABXI2UO6JTOMACOVZBK2SIC5QOHZK
  [9]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/master_slave_reactor.webp?token=ABXI2UM4NNXYSEM6D3YV7AK5QOIII
  [10]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/netty_thread.webp?token=ABXI2UJ5W3TQ2WRR4BYDHDS5QIAU6
  [11]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/NioEventLoop&NioChannel.webp?token=ABXI2UIP25DK6NBMLCGXZ3S5QOL4U
  [12]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/server_NIO.webp?token=ABXI2ULNFSINBL5JMNCJ7B25QOMJ2
  [13]: https://raw.githubusercontent.com/Audi-A7/learn/master/source/image/netty/clientNio.webp?token=ABXI2UJNBADO5I3G2KWC4QS5QOMNI