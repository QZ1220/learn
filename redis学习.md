# redis学习

标签（空格分隔）： redis 数据结构

---

参考链接
----
https://www.toutiao.com/i6731690432675185165/

redis数据结构
=========

redisObject
-----------
理论上来说，所有redis存储的value都是一个redisObject，它的结构如下：
```c++
typedef struct redisObject {
 unsigned [type] 4;
 unsigned [encoding] 4;
 unsigned [lru] REDIS_LRU_BITS;
 int refcount;
 void *ptr;
} robj;
```
简单介绍一下这几个字段：

 - type：数据类型，就是我们熟悉的string、hash、list等。
 - encoding：内部编码，其实就是本文要介绍的数据结构。指的是当前这个value底层是用的什么数据结构。因为同一个数据类型底层也有多种数据结构的实现，所以这里需要指定数据结构。
 - REDIS_LRU_BITS：当前对象可以保留的时长。这个我们在后面讲键的过期策略的时候讲。
 - refcount：对象引用计数，用于GC。
 - ptr：指针，指向以encoding的方式实现这个对象的实际地址。

![此处输入图片的描述][1]

string
------
在Redis内部，string类型有两种底层储存结构。Redis会根据存储的数据及用户的操作指令自动选择合适的结构：

 - int：存放整数类型；
 - SDS(简单动态字符串 simple dynamic string)：存放浮点、字符串、字节类型；

SDS的内部数据结构：
```c++
typedef struct sdshdr {
 // buf中已经占用的字符长度
 unsigned int len;
 // buf中剩余可用的字符长度
 unsigned int free;
 // 数据空间
 char buf[];
}
```
可见，其底层是一个char数组。buf最大容量为512M，里面可以放字符串、浮点数和字节。它为什么没有直接使用数组，而是包装成了这样的数据结构呢？

因为buf会有动态扩容和缩容的需求。如果直接使用数组，那每次对字符串的修改都会导致重新分配内存，效率很低。

buf的扩容过程如下：

 - 如果修改后len长度将小于1M,这时分配给free的大小和len一样,例如修改过后为10字节,
   那么给free也是10字节，buf实际长度变成了10 + 10 + 1 = 21byte (下图展示的就是该种情况)
 - 如果修改后len长度将大于等于1M,这时分配给free的长度为1M,例如修改过后为30M,那么给free是1M.buf实际长度变成了30M + 1M + 1byte

![此处输入图片的描述][2]
 
 需要指出的是，与扩容相反，当缩容时，并不会进行真正的缩容，「惰性空间释放指的是当字符串缩短时，并没有真正的缩容，而是移动free的指针。这样将来字符串长度增加时，就不用重新分配内存了。但这样会造成内存浪费，Redis提供了API来真正释放内存。」
 
 

list
----
list底层有两种数据结构：链表linkedlist和压缩列表ziplist。当list元素个数少且元素内容长度不大时，使用ziplist实现，否则使用linkedlist。

 - 链表

Redis使用的链表是双向链表。为了方便操作，使用了一个list结构来持有这个链表。如图所示：
![此处输入图片的描述][3]
其数据结构如下所示：
```c++
typedef struct list{
 //表头节点
 listNode *head;
 //表尾节点
 listNode *tail;
 //链表所包含的节点数量
 unsigned long len;
 //节点值复制函数
 void *(*dup)(void *ptr);
 //节点值释放函数
 void *(*free)(void *ptr);
 //节点值对比函数
 int (*match)(void *ptr,void *key);
}list;
```

 - 压缩列表

与上面的链表相对应，压缩列表有点儿类似数组，通过一片连续的内存空间，来存储数据。不过，它跟数组不同的一点是，它允许存储的数据大小不同。每个节点上增加一个length属性来记录这个节点的长度，这样比较方便地得到下一个节点的位置。
![此处输入图片的描述][4]

上图的各字段含义为：

 - zlbytes：列表的总长度
 - zltail：指向最末元素
 - zllen：元素的个数
 - entry：元素的内容，里面记录了前一个Entry的长度，用于方便双向遍历
 - zlend：恒为0xFF，作为ziplist的定界符

 压缩列表不只是list的底层实现，也是hash的底层实现之一。当hash的元素个数少且内容长度不大时，使用压缩列表来实现。
 
 

hash
----
hash底层有两种实现：压缩列表和字典（dict）。压缩列表刚刚上面已经介绍过了，下面主要介绍一下字典的数据结构。

 
 


  [1]: https://github.com/Audi-A7/learn/blob/master/image/redis/redis_object.jpeg?raw=true
  [2]: https://github.com/Audi-A7/learn/blob/master/image/redis/sds.jpeg?raw=true
  [3]: https://github.com/Audi-A7/learn/blob/master/image/redis/list.jpeg?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/redis/ziplist.jpeg?raw=true