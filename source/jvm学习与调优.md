# jvm学习与调优

标签（空格分隔）： 未分类

---


* [jvm学习与调优](#jvm学习与调优)
   * [常用命令](#常用命令)
      * [jps](#jps)
      * [jinfo](#jinfo)
      * [jstat](#jstat)
      * [jmap](#jmap)
      * [jstack](#jstack)
   * [调优](#调优)
      * [打印gc日志](#打印gc日志)
      * [分析gc日志](#分析gc日志)

hint：以下如无特殊说明都是针对jdk8而言

## 常用命令

- https://docs.oracle.com/javase/8/docs/technotes/tools/

从上面的[官方](https://docs.oracle.com/javase/8/docs/technotes/tools/)文档，我们知道jdk提供了比较丰富的命令行工具。这里着重介绍一些较为常用的。

### jps
[这个命令](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)主要是列出当前存活的java虚拟机进程信息。

```java
root@1c30b864a390:/# jps -l
1 /app/app.jar
13816 sun.tools.jps.Jps
```

### jinfo

[这个命令](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html)可以获取一些jvm的相关配置参数信息。

```linux
### 查看最大堆内存
# jinfo -flag MaxHeapSize 39137
-XX:MaxHeapSize=268435456

## 查看使用的垃圾回收器
# jinfo -flag UseG1GC 39137
-XX:-UseG1GC

## 可以看出这里使用的并行垃圾回收器
# jinfo -flag UseParallelGC 39137
-XX:+UseParallelGC

# jinfo -flag UseConcMarkSweepGC 39137
-XX:-UseConcMarkSweepGC
```

### jstat
[这个命令](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)主要用于统计一些jvm的性能相关的信息，功能非常强大，可选的配置参数也很多，我一般用来查看gc的相关信息及内存占用等。

如 -gc 参数，可以显示young、old大小、gc次数及时间等信息。
```linux
root@1c30b864a390:/# jstat -gc 1
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
6656.0 6656.0 696.2   0.0   140288.0 102410.2  153600.0   73818.1   104576.0 98932.9 13184.0 12125.8     62    1.598   0      0.000    1.598

## 也可以使用如下语句每隔固定时间进行输出：
# jstat -gc 39137  5s 5
```
相关参数含义如下：
```java
S0C: Current survivor space 0 capacity (kB).

S1C: Current survivor space 1 capacity (kB).

S0U: Survivor space 0 utilization (kB).

S1U: Survivor space 1 utilization (kB).

EC: Current eden space capacity (kB).

EU: Eden space utilization (kB).

OC: Current old space capacity (kB).

OU: Old space utilization (kB).

MC: Metaspace capacity (kB).

MU: Metacspace utilization (kB).

CCSC: Compressed class space capacity (kB).

CCSU: Compressed class space used (kB).

YGC: Number of young generation garbage collection events.

YGCT: Young generation garbage collection time.

FGC: Number of full GC events.

FGCT: Full garbage collection time.

GCT: Total garbage collection time.
```
### jmap

[这个命令](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html)主要用于输出jvm的堆信息，例如可以使用如下命令保存堆信息为二进制文件：
```java
# jmap -dump:format=b,file=build.hprof 49450
Dumping heap to /Users/wangquanzhou/ideaProj/learn/build.hprof ...
Heap dump file created
```
然后配合mat等工具可进行内存溢出分析

### jstack

[这个命令](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)主要输出jvm的线程栈信息以及线程状态。`jstack pid`即可查看该pid下的线程栈信息及线程状态。一般cpu占用较高的话，可以使用这个命令进行排查。

## 调优

### 打印gc日志

- https://blog.csdn.net/u012988901/article/details/100708349?spm=1001.2014.3001.5501

要打印gc日志，需要设置一些jvm启动参数，这里我们正在开发的一个系统为基础，在idea里进行设置相关vm参数进行调优测试。

java版本：
```java
➜  Documents java -version
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
```

jvm参数：
```java
## 这里我们直接在项目的根目录输出gc日志
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:./gc.log 
```

注意：这里我除了上面的三个-xx参数外，并没有设置其他的jvm参数了。

![gc](./image/jvm/gc_print_options.jpg)

### 分析gc日志

- https://blog.csdn.net/u012988901/article/details/102791020

参考这篇博客，我选用了GCViewer来帮助我分析gc情况。

```java
git  clone https://github.com/chewiebug/GCViewer.git

cd GCViewer

mvn clean package -Dmaven.test.skip=true
```

然后在target下有一个jar包，gcviewer-1.37-SNAPSHOT.jar，直接双击打开。然后加载刚刚生成的gc日志文件。
![gcViewer](./image/jvm/gcViewer.jpg)







- https://blog.csdn.net/u012988901
- https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/toc.html
- https://blog.csdn.net/u012988901/article/details/100708349?spm=1001.2014.3001.5501
