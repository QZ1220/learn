# prime工作总结

标签（空格分隔）： 区块链

---

* [prime工作总结](#prime工作总结)
   * [raft算法介绍](#raft算法介绍)
   * [区块链简介](#区块链简介)
   * [RS总结](#rs总结)
   * [VN总结](#vn总结)
   * [synchronizer总结](#synchronizer总结)
   * [rs_failover总结](#rs_failover总结)
   * [prime slave总结](#prime-slave总结)
   * [代码技巧](#代码技巧)
   * [prime slave多币种支持改造](#prime-slave多币种支持改造)

## raft算法介绍

记录在另外一篇[笔记](https://github.com/AudiVehicle/learn/blob/master/source/atomix%E5%88%86%E4%BA%AB/raft%E5%88%86%E4%BA%AB.md)。

## 区块链简介

简单来讲，区块链是一个分布式的记账本，核心在于去中心化以及可信任链。
区块链技术的典型应用场景就是支付和清算，可以解决不同主体之间的信任问题。

区块链采用密码学的方法来保证已有数据的不可篡改性，主要包括密码学**哈希**函数和**非对称加密**（有公钥和私钥）。

## RS总结

RS主要负责和用户Api进行交互，接受Api的请求，以及将处理请求的结果反馈给AI。

RS可以有多个，相互之间独立运行。

RS内部会存储本地分账，其中可能包括交易信息、账户信息、交易方信息等。

RS请求VN可分为两种方式，同步请求和异步请求。

同步请求VN主要发生在交易信息的初始阶段，如商户业务系统首次发起调用。同步请求要求实时返回处理结果，且会设置超时时间30秒。30秒请求VN并成功返回，则订单状态为已受理，否则返回系统繁忙的信息。

异步请求VN与同步请求VN的去呗在于RS请求VN以后，将将VNRequestRecord发送到请求VN待处理队列以后就会返回，而不会有一个30秒的等待时长。此时返回的结果是处理中。

RS除了主动请求VN以外，还可以接受VN的回调。这里分为回调受理流程（异步）和回调处理流程两部分。
 
回调受理流程主要负责将回调区块id、高度发送到VN区块回调待处理队列，然后返回成功标志，如果不成功，会一直重试，重试达到指定次数，将会产生告警信息。

回调处理流程主要处理回调受理过程存储的VNBlockCallbackRecord，通过查询VNBlockCallbackRecord，获得区块处理进度（处理完成的最后一个区块的高度，假设为N），通过数据库查询是否存在高度为N+1的区块，不存在就从受理流程产生的待处理队列中取记录，从而获得N+1区块。存在就直接处理，处理流程如下。一个VNBlockCallbackRecord就对应着一个BlockItem。首先blockitem中的数据需要进行一次校验与本地数据的校验，校验通过且rsid相等才会进行更新，如果更新失败，会不断重试更新。处理完成以后接着处理下一个block中的blockitem数据。


VN投票结果回调接受流程和VN投票结果回调处理流程（类似于上面的两步的流程）

通用数据模型一般数据结构相对固定，用户定制模型一般是在通用模型的基础上进行一些定制化的处理。

刚刚的代码在哪里看到的？

## VN总结

VN可以分为三层。分别是VN SERVICE、VN BIZ、VN INTEGRATION。三层各自对应一些pojo。

VN SERVICE负责给外部系统调用，包括RS投票和SLAVE的回调。

VN BIZ负责VN模块内部业务逻辑的处理。

VN INTEGRATION负责主动调用外部系统，比如调用RS或者Slave。


findVoteRule


在bizFSM(BizVoting bizVoting)方法里面有一个while循环，一直处理订单，直到订单的状态变成Done才可以。
投票的几种状态：Init、Voted、VoteDecided、ChainDecided、InChain、Receipting、Receipted、Done。


syncVote(RSVoteParam rsVoteParam)这个方法不太懂
为什么需要并发和串行两种方式来处理回调？
非发起方回调处理和发起方回调处理有什么区别？
什么是回调隐藏规则？什么是回调数据隐藏？
LOGGER.error("回调处理第[{}]个子任务时发生异常", i);这里看不懂


## synchronizer总结
synchronizer主要完成入链数据的打包分发，将instructions打包成数组的形式，然后再转发给下层的prime slave。


## rs_failover总结

这个模块主要完成rs节点的上线以及下线处理，伴随着这个处理过程会有数据的同步处理过程在其中。
下线过程分为正常下线和异常下线。

## prime slave总结

该模块主要完成往区块链上写入数据，完成AI发出的交易指令，完成用户账户之间的交易。

首先监听package的变换，如果package有变化就放入本地队列，并尝试进行校验。校验前，需要从数据库取出校验当前package所需的数据。校验时，针对package中的指令单起一个线程进行处理。校验需要多个节点（>3）同时参与，各个节点汇总校验结果到leader进行汇总投票。然后各个节点再进行后续的处理，比如具体指令的执行，然后就是持久化数据，持久化的结果也需要进行汇总投票，当票数超过某一个值N时，开始进行回调处理，也就是通知VN，RS指令执行成功。



##slave模块本地调试流程
 1. 启动zookeeper
 2. 启动primeslave的三个节点
 3. 除第一个启动的节点外，后两个节点进行join操作
 4. 除第一个启动的节点外，后两个节点分别进行上线操作
 6. 启动org进行数据构造



## 代码技巧

 1. 对于入参的校验，可以使用google的guava包里的Preconditions类提供的checkArgument方法，例如下面的代码：
```java
Preconditions.checkArgument(timeoutMS > 0, "timeoutMS must > 0");
```
 2. 对于多个参数的校验，或者是对实体类的参数校验
```java
BeanValidator.validate(vnRequestParam).failThrow();

 
 
MNG运行下，VNCallbackProcessService的运行状态必须为=PAUSED or STOPPED

// 注意不能直接使用JSON.toJSON, 会丢失一些序列化特性(例如跳过transient字段)
                raw.setRaw(JSON.parseObject(JSON.toJSONString(callback)));
                
                
                asyncRequestVNToTerminate
```
                
                

## prime slave多币种支持改造

问题：
1. 冻结金额部分
2. 冻结金额与总金额的显示（即冻结的金额是否会从总金额中扣除）
3. 开户、交易时对于accountId的验证
4. 开户的时候对于冻结金额的支持
5. accountSnapshotMap是一直存在内存中的（不查询数据库）

需要修改的地方：
同一个用户具有一个唯一的address，这个唯一的address可以对应多个accountId，一个accountId+currency组合标明当前address下的一个币种。

开户：
1. account表增加currency（币种）字段
2. 开户时，除了设置account_id、amount（如果有的话）以外，还要设置currency字段

交易：
1. 交易隔离（不同币种之间不能直接转币）
2. 余额有效性校验


```java
>eth.getTransaction（"0xf197f83993742a4e422244fe91a1175513ad11da2bd1209906443dcc391279eb")
{
  blockHash: "0xf3ac98d6db31f72a586b0239eab43d40fb568668561769c2e60d4dd7082050d8",
  blockNumber: 40164,
  from: "0x95a171d45c7551474f3479bf006e2a9a3852bbd8",
  gas: 90000,
  gasPrice: 100000000000,
  hash: "0xf197f83993742a4e422244fe91a1175513ad11da2bd1209906443dcc391279eb",
  input: "0x90b98a11000000000000000000000000964f31c1c95eac140e9f1ef1e1a22a5f3cc7a90600000000000000000000000000000000000000000000000000000000000f4240",
  nonce: 73,
  r: "0x3480b8405e47e1b402039f813b966cfb894ff61f60d2c5301f85c889638dce1d",
  s: "0x6cc87ac31c610f78f258f36551e7a74833cb361afe596fc43bc9ca3e8a6e3ceb",
  to: "0x7673af653c4ea7dee50a10c50d1d62a20c63cd5c",
  transactionIndex: 0,
  v: "0x42",
  value: 0
}
```