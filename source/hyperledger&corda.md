# hyperledger&corda

标签（空格分隔）： 区块链

---

hyperledger
===========
hyperledger的智能合约(chaincode)使用docker存储和运行.

Paxos
-----
https://yeasy.gitbooks.io/blockchain_guide/content/distribute_system/paxos.html

Paxos 问题是指分布式的系统中存在故障（fault），但不存在**恶意**（corrupt）节点场景（即可能消息丢失或重复，但无错误消息）下的共识达成（Consensus）问题。因为最早是 Leslie Lamport 用 Paxon 岛的故事模型来进行描述而命名。

作为现在共识算法设计的鼻祖，以最初论文的难懂（算法本身并不复杂）出名。算法中将节点分为三种类型：

 - proposer：提出一个提案，等待大家批准为结案。往往是客户端担任该角色；
 - acceptor：负责对提案进行投票。往往是服务端担任该角色；
 - learner：被告知结案结果，并与之统一，不参与投票过程。可能为客户端或服务端。

Paxos 能保证在超过  的正常节点存在时，系统能达成共识。

Raft
----
Raft 算法是Paxos 算法的一种简化实现。

包括三种角色：leader、candidate 和 follower，其基本过程为：

 - Leader 选举：每个 candidate 随机经过一定时间都会提出选举方案，最近阶段中得票最多者被选为 leader；
 - 同步 log：leader 会找到系统中 log 最新的记录，并强制所有的 follower 来刷新到这个记录；

注：此处 log 并非是指日志消息，而是各种事件的发生记录。

拜占庭问题(BFT)
----------
https://yeasy.gitbooks.io/blockchain_guide/content/distribute_system/bft.html

拜占庭问题更为广泛，讨论的是允许存在少数节点作恶（消息可能被伪造）场景下的一致性达成问题。拜占庭算法讨论的是最坏情况下的保障。
对于拜占庭问题来说，假如节点总数为 N，叛变将军数为 F，则当 $ N>=3F+1$ 时，问题才有解，即 Byzantine Fault Tolerant (BFT) 算法。

默克尔树
----

默克尔树（又叫哈希树）是一种二叉树，由一个根节点、一组中间节点和一组叶节点组成。最下面的叶节点包含存储数据或其哈希值，每个中间节点是它的两个孩子节点内容的哈希值，根节点也是由它的两个子节点内容的哈希值组成。






