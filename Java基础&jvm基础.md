---

# Java基础&jvm基础

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
https://blog.csdn.net/baidu_30000217/article/details/53671716

一致性Hash算法主要是为了解决分布式环境下,数据均匀映射散列分布的问题.具体可以参考一下[这里][1].


  [1]: https://github.com/crossoverJie/JCSprout/blob/master/MD/Consistent-Hash.md
  
  但是如上的链接对于虚拟节点的讲解不是很清除,我又搜索整理了一下资料.