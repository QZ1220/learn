#Atomix分享

Atomix 是一个帮助我们搭建分布式容错系统的工具。它能应对非拜占庭问题，即丢包，延迟，乱序等非人为问题。

拜占庭问题有解：  
一致性：所有正值的节点都会执行一致的命令。  
正确性：Leader也正值，所有正值的节点会执行leader的命令。

##Distributed primitives（分布式基元）
分布式基元是Atomix能够在分布式系统中进行**状态复制**和**状态协调**的核心。由于raft算法被atmoix封得很深，作为使用者，我们其实看不到和状态机相关的内容，仅仅能通过atomix提供的对**分布式数据结构**、**分布式通信**和**分布式协调**的操作，管理分布式系统。而对比copycat，我们在搭建copycat服务器的时候，需要自定状态机和command，然后配进copycat服务器。

分布式基元的例子：  
>Distributed data structures (maps, sets, trees, etc)  
Distributed communication (direct, publish-subscribe, etc)  
Distributed coordination (locks, leader elections, semaphores, etc)

##两个协议
###Raft
Raft协议是Atomix中用于强一致性、分区容错的一致协议。Atomix为基于共识的分布式基元提供了成熟的raft议实现。在Atomix核心，raft能将分布式基元的更改转化成持久化的复制日志。
它通过一个leader复制状态改变给follower来实现。由于只有一个leader，集群能保证一致性。
然而，raft协议的一个重要特性是，只有当大多数集群可用时，它才有效。（大于一半的节点正确）

所以，raft协议能分区容错并保证强一致性，但它所带来的性能负担着实不小。

###Primary-backup（主备份协议）

主备份协议是一种更简单的内存复制协议，具有较弱的一致性保证。与Raft不同的是，Atomix主备份协议可以容忍除一个节点以外的所有节点丢失，状态改变可以同步或异步复制到任意数量的节点。这使得主备份协议更适合于高性能和低一致性要求的场景。

主备份协议的工作方式是选择主节点，将状态改变复制到备份节点。

##Partition Groups（分区组）
在Atomix中，最先配置的cluster并不是复制分布式基元的集群，它需要我们配置一组或多组**分区组**，在各个分区组中复制分布式基元。各个分区组可配置为raft协议或主备份协议来支撑对分布式基元的复制。

![partition group](C:\Users\Zhu_Yuanxiang\Desktop\atomix分享\partition-group.png)

####两种分区组类型
1.管理分区组  
用与存储分布式基元的metadata，复制到管理分区组的所有节点上。

```
	        builder.withManagementGroup(
                RaftPartitionGroup.builder("system")
                        .withStorageLevel(StorageLevel.DISK)
                        .withMembers(serverID)
                        .withNumPartitions(1)
                        .withDataDirectory(new File(dataPath + "system"))
```                     .build())


2.分布式基元分区组
用于存储和复制分布式基元到分区组的所有节点上。

```
				builder.withPartitionGroups(
                        RaftPartitionGroup.builder("data")
                                .withStorageLevel(StorageLevel.DISK)
                                .withMembers(serverID)
                                .withNumPartitions(1)
                                .withDataDirectory(new File(dataPath + "data"))
```                             .build());

####多组分区组
我们在创建分布式基元的时候需要指定此分布式基元在哪一个分区组进行共享，所以atomix支持创建多组分区组，各个分区组数据隔离。

####每个分区组多个分区
每个分区组可以支持多个分区，当一个分布式基元被创建，它将被复制到此分区组的每一个分区。
如果一个分区组配置为raft算法，这个分区组的每一个分区都会有一个leader负责状态复制。


>This allows multiple Raft leaders to concurrently replicate changes for the primitive, and it’s how Atomix achieves greater scalability than similar systems. 

##Atomix代码结构
atomix-core  
1.负责atomix的主要逻辑。  
2.各种分布式数据的管理：  
ConsistentMap  
ConsistentMultimap  
DistributedSet  
AtomicValue  
3.各种分布式协调的管理：  
LeaderElection  
DistributedLock  
WorkQueue

atomix-primitives  
1.负责partition管理，数据在partition中共享  
2.负责atomix在各个partition中复制状态和协调状态改变  
3.管理协议

atomix-protocal-raft
实现raft协议

atomix-protocal-primary-backup  
实现primary-backup协议

atomix-cluster  
1.负责atomix集群中所有成员管理，cluster和partition是有区别的  
2.负责消息收发

atomix-storage  
负责数据存储管理，三个等级：内存，内存map，磁盘

![简要流程](C:\Users\Zhu_Yuanxiang\Desktop\atomix分享\简要流程.png)