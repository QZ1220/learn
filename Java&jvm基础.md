---

# Java&jvm基础

标签（空格分隔）： 笔记
---

content
-------

 1. java基本类型
 2. Java中的NIO
 3. 一致性 Hash 算法

java基本类型占几个字节？
==============

| 整数类型 | byte    | 1B         | -128~127               |
| -------- | ------- | ---------- | ---------------------- |
|          | short   | 2B         | -32768~32767           |
|          | int     | 4B         | -2147483648~2147483647 |
|          | long    | 8B         | -2^63~2^63-1           |
| 字符类型 | char    | 2B         |                        |
| 浮点型   | float   | 4B         |                        |
|          | double  | 8B         |                        |
| 布尔型   | boolean | 一般占用1B |                        |

需要指出的是：所有的正无穷大都是相等的，所有的负无穷大也是相等的。
NaN不与任何数值相等，与NaN也不相等。浮点数除以0会得到正/负无穷大，整数除以0会抛出异常。

Java中的NIO
=========

NIO也就是New Input/Output，新IO。
NIO和IO目的相同，都是用于进行输入/输出。但是新IO采用内存映射文件的方式来处理IO。新IO将文件或文件的一段区域映射到内存中，这样就可以像访问内存一样来访问文件了（这种方式模拟了操作系统上的虚拟内存的概念）。
Java中与NIO相关的包如下：

 - java.nio：主要包含各种和Buffer相关的类
 - java.nio.channels：主要包含与Channel和Selector相关的类

java.nio.charset：主要包含与字符集相关的类
java.nio.channels.spi
java.nio.charset.spi

一致性 Hash 算法
===========

一致性Hash算法主要是为了解决分布式环境下,数据均匀映射散列分布的问题.具体可以参考一下[这里][1].


  但是如上的链接对于虚拟节点的讲解不是很清除,我又搜索整理了一下资料.
  
  https://blog.csdn.net/baidu_30000217/article/details/53671716
  
  发现这个链接对于虚拟节点部分讲解挺不错的.引入虚拟节点以后:
  ![此处输入图片的描述][2]
  
  此时，对象到“虚拟节点”的映射关系为：

objec1->cache C2 ； objec2->cache A1 ； objec3->cache C1 ； objec4->cache A2 ；
注意,objec2,objec3,objec4并未在上图中标出,但是位置我们应该知道他们的位置.例如objec2应该在cache C2和cache A1之间.

此时,又因为cache C2和cache C1都会映射到cache C上.同理,cache A1和cache A2也会映射到cache A上.因此,平衡性有了很大提高。
  


  [1]: https://github.com/crossoverJie/JCSprout/blob/master/MD/Consistent-Hash.md
  [2]: https://github.com/WQZ321123/learn/blob/master/image/consistentHash/virtualNode.png?raw=true