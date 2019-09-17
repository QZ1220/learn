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
 

 
 
 
 


  [1]: https://github.com/Audi-A7/learn/blob/master/image/netty/%E4%BC%A0%E7%BB%9F%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2.png?raw=true
  [2]: https://github.com/Audi-A7/learn/blob/master/image/netty/%E4%BC%A0%E7%BB%9F%E7%9A%84%E6%95%B0%E6%8D%AE%E6%8B%B7%E8%B4%9D%E6%96%B9%E6%B3%95.png?raw=true
  [3]: https://github.com/Audi-A7/learn/blob/master/image/netty/%E4%BC%A0%E7%BB%9F%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A22.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/netty/%E9%9B%B6%E6%8B%B7%E8%B4%9D%E4%BC%A0%E8%BE%93%E6%96%B9%E6%B3%95.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/image/netty/transferToContext.png?raw=true
  [6]: https://github.com/Audi-A7/learn/blob/master/image/netty/zeroCopy.png?raw=true