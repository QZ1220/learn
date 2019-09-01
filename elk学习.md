# elk学习

标签（空格分隔）： rabbitMq

---

参考链接：https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/

https://www.jianshu.com/p/0962fb50ecff

我们本次的目的是搭建一个单机版的elk日志收集分析可视化组建。如下图所示：
![此处输入图片的描述][1]

os：macOs 10.14.5 

elasticsearch-5.6.4.tar.gz       

kibana-5.2.0-darwin-x86_64.tar.gz

logstash-5.6.3.tar.gz

整体来说还是比较简单的，基本都是解压以后，直接启动就可以了。

解压启动elasticsearch
-----------------

```linux
tar -zxvf elasticsearch-5.6.4.tar.gz

./elasticsearch-5.6.4/bin/elasticsearch
```

![此处输入图片的描述][2]

解压启动logstash-5.6.3
------------------
```linux
cd logstash-5.6.3/config/
vim logstash.conf

输入如下内容：
input {
     file {
        type => "log"
        path => "/Users/wangquanzhou/IdeaProjects/micro-service/log/*.log"
        start_position => "beginning"
    }
}
output {
  stdout {
   codec => rubydebug { }
  }
  elasticsearch {
    hosts => "127.0.0.1"
    index => "log-%{+YYYY.MM.dd}"
  }
}
```

然后启动应用，同样是bin下  ./logstash  启动

启动完成以后，logstash会去配置里指定的目录下收集所有的日志，如下图所示：
![此处输入图片的描述][3]

最后，启动kibana服务
-------------
```linux
tar -zxvf kibana-5.2.0-darwin-x86_64.tar.gz

./kibana-5.2.0-darwin-x86_64/bin/kibana
```
![此处输入图片的描述][4]

访问kibana的页面，http://localhost:5601，做如下配置：
![此处输入图片的描述][5]

然后就可以查看日志了：
![此处输入图片的描述][6]

Elasticsearch原理学习
-----------------

参考链接：

 - https://www.jianshu.com/p/eaa59e966ec4
 - https://www.cnblogs.com/hzzhero/p/7000672.html
 

学习Elasticsearch原理，我觉得有以下几个方面需要注意：

 1. 倒排索引
 2. 数据存储
 3. 选主逻辑


首先我们说是**倒排索引**：

它首先是一个索引，为了加快数据的查询效率。当我们在进行文档关键字搜索的时候，**常规**的做法是：从文档的开始直接检索该文档，效率较为低下。

**倒排索引**是这么做的：首先将文档进行分段存储，并且利用分词器将文档内容分成一个个的词条（Term：对英文来说，就是一个个的单词，对中文来说，一般是分词后的一个词）。分词的同时，需要记录该term出现在哪个文档，哪个位置，出现的频次等信息。最终，会将这些信息组成一个倒排索引，存在内存中。实际的文本文件是存储在磁盘上的。              当我们需要查询某个词（word）的时候，es回去索引文件中查询该词对应索引记录的信息，进而定位到word对应的文档位置的信息，从而得到结果。如果查询的是词组（sentence），同样的，es会去索引文件中查询sentence对应的索引记录，理论上会得到多个索引信息，求**交集**，定位sentence的最终位置，从而查询得到结果信息。

![此处输入图片的描述][7]
![此处输入图片的描述][8]

接下来我么说一下es的**数据存储**：

 - 分段存储
 所谓分段存储，其实就是刚刚说的，需要被检索的文档，会被拆分成多个部分存储在磁盘上。在底层采用了分段的存储模式，使它在读写时几乎完全避免了锁的出现，大大提升了读写性能。段被写入到磁盘后会生成一个**提交点**，提交点是一个用来记录所有提交后段信息的文件。一个段一旦拥有了提交点，就说明这个段**只有读**的权限，**失去了写**的权限。相反，当段在内存中时，就只有写的权限，而不具备读数据的权限，意味着不能被检索。
 - 延迟刷新
所谓延迟刷新，就是新增的文本文档，不会立即存储到磁盘上，而是经过一定时间或者达到一定大小时才会刷新到磁盘上。我们也可以手动触发 Refresh，POST /_refresh 刷新所有索引，POST /nba/_refresh 刷新指定的索引。

为了避免丢失数据，Elasticsearch添加了**事务日志**(Translog，那Translog如何保证断电信息不丢失呢？我觉得其实它也是类似于固态硬盘这种的disk，写入速度要远高于机械disk)，事务日志记录了所有还没有持久化到磁盘的数据。
![此处输入图片的描述][9]

 - 段合并
由于自动刷新流程每秒会创建一个新的段，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。

每一个段都会消耗文件句柄、内存和 CPU 运行周期。更重要的是，每个搜索请求都必须轮流检查每个段然后合并查询结果，所以段越多，搜索也就越慢。
Elasticsearch 通过在后台定期进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。

段合并的时候会将那些旧的已删除文档从文件系统中清除。被删除的文档不会被拷贝到新的大段中。合并的过程中不会中断索引和搜索。

段合并在进行索引和搜索时会自动进行，合并进程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中，这些段既可以是未提交的也可以是已提交的。

合并结束后老的段会被删除，新的段被 Flush到磁盘，同时写入一个包含新段且排除旧的和较小的段的新提交点，新的段被打开可以用来搜索。
段合并的计算量庞大， 而且还要吃掉大量磁盘 I/O，段合并会拖累写入速率，如果任其发展会影响搜索性能。
Elasticsearch 在默认情况下会对合并流程进行资源限制，所以搜索仍然有足够的资源很好地执行。

然后我们在讨论一下es的**选主逻辑**：

es内部一般有两种角色，**master**节点和**data**节点和**coordinator**节点。master节点负责创建索引、删除索引、跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点、追踪集群中节点的状态等，稳定的主节点对集群的健康是非常重要的。data节点负责数据的存储和相关的操作，例如对数据进行增、删、改、查和聚合等操作，所以数据节点(Data 节点)对机器配置要求比较高，对 CPU、内存和 I/O 的消耗很大。因此一般我们不宜将master节点和data节点部署在一起。

虽然对节点做了角色区分，但是用户的请求可以发往任何一个节点，并由该节点负责分发请求、收集结果等操作，而不需要主节点转发。这种节点可称之为coordinator节点，协调节点是不需要指定和配置的，集群中的任何节点都可以充当协调节点的角色。

ES 内部是如何通过一个相同的设置 cluster.name 就能将不同的节点连接到同一个集群的?答案是 **Zen Discovery**。它提供单播和基于文件的发现，并且可以扩展为通过插件支持云环境和其他形式的发现。Zen Discovery 与其他模块集成，例如，节点之间的所有通信都使用 Transport 模块完成。节点使用发现机制通过 Ping 的方式查找其他节点。

Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上运行的节点才会自动组成集群。如果集群的节点运行在不同的机器上，使用单播，你可以为 Elasticsearch提供一些它应该去尝试连接的节点列表。
当一个节点联系到单播列表中的成员时，它就会得到整个集群所有节点的状态，然后它会联系 Master 节点，并加入集群。这意味着单播列表不需要包含集群中的所有节点， 它只是需要足够的节点，当一个新节点联系上其中一个并且说上话就可以了。

如果你使用 Master 候选节点作为单播列表，你只要列出三个就可以了。这个配置在 elasticsearch.yml 文件中：
```java
discovery.zen.ping.unicast.hosts: ["host1", "host2:port"] 
```
节点启动后先 Ping ，如果 discovery.zen.ping.unicast.hosts 有设置，则 Ping 设置中的 Host ，否则尝试 ping localhost 的几个端口。

Elasticsearch 支持同一个主机启动多个节点，Ping 的 Response 会包含该节点的基本信息以及该节点认为的 Master 节点。

选举开始，先从各节点认为的 Master 中选，规则很简单，按照 ID 的字典序排序，取第一个。如果各节点都没有认为的 Master ，则从所有节点中选择，规则同上。

这里有个限制条件就是 discovery.zen.minimum_master_nodes ，如果节点数达不到最小值的限制，则循环上述过程，直到节点数足够可以开始选举。
最后选举结果是肯定能选举出一个 Master ，如果只有一个 Local 节点那就选出的是自己。
如果当前节点是 Master ，则开始等待节点数达到 discovery.zen.minimum_master_nodes，然后提供服务。
如果当前节点不是 Master ，则尝试加入 Master 。Elasticsearch 将以上服务发现以及选主的流程叫做 Zen Discovery 。

由于它支持任意数目的集群( 1- N )，所以不能像 Zookeeper 那样限制节点必须是奇数，也就无法用投票的机制来选主，而是通过一个规则。

为了预防集群**脑裂**的问题发生，一般需要设置当集群一边拿以上的节点选出的master一致时，才能对外提供服务。





 


  [1]: https://github.com/Audi-A7/learn/blob/master/image/elk/elk.png?raw=true
  [2]: https://github.com/Audi-A7/learn/blob/master/image/elk/elastic.png?raw=true
  [3]: https://github.com/Audi-A7/learn/blob/master/image/elk/logstash.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana_config.png?raw=true
  [6]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana_front_page.png?raw=true
  [7]: https://github.com/Audi-A7/learn/blob/master/image/elk/es1.png?raw=true
  [8]: https://github.com/Audi-A7/learn/blob/master/image/elk/es2.png?raw=true
  [9]: https://github.com/Audi-A7/learn/blob/master/image/elk/es_translog.png?raw=true