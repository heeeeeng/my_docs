# **Raft-based Consensus**
在2017年11月30日，Quorum 推出了 2.0 的 release 版本。这一版最大的改动就是剔除了 QuorumChain的共识方式，只支持 Raft-based Consensus。相比较以太坊的POW，Raft-based 提供了更快更高效的区块生成方式。相比 QuorumChain，Raft-based 不会产生空的区块，而且在区块的生成上比前者更有效率。

在这一章节，我们重点关注 Raft 是如何被运用到 Quorum 上的。想要了解 Raft 算法的技术细节，可以参考 [raft.github.io](https://raft.github.io/) 或者 [Raft 动态演示](http://thesecretlivesofdata.com/raft/)。

## **Raft on Quorum**
Quorum 的 Raft 代码主要沿用了 [etcd's Raft implementation](https://github.com/coreos/etcd/tree/master/raft)。

以太坊和 Raft 都有节点这一概念。在 Raft 中，节点可以有 leader, follower, condidate 三种身份。leader 负责生产区块。follower 监听 leader 的心跳并收取 leader 传递过来的区块。follower 一段时间没有收到来自 leader 的心跳后，就转变身份为 candidate，同时整个 Raft 进入下一个周期。然后 candidate 向所有节点发送投票的请求，其他节点（leader 收到消息后 leader 身份就被收回）收到请求后给予应答。如果 candidate 收到超过半数的应答，则晋升为 leader。一个节点一个周期只能投一次票。

在以太坊中，任意节点都可以作为区块的打包者，只要其在一轮 pow 中胜出。我们知道 Quorum 的节点沿用了以太坊的设计和代码。所以为了连接以太坊节点和 Raft 共识，Quorum 采用了网络节点和 Raft 节点一对一的方式来实现 Raft-based 共识。当一笔 TX 诞生后，TX 会在以太坊的 P2P 网络中传输。同时，Raft 的 leader 竞选一直在同步进行。当前 leader 节点对应的以太坊节点收到 TX 时，以太坊节点就会将 TX 打包成区块并将区块通过 Raft 节点发送给 Raft 网络上的 follower。follower 节点收到区块后将区块交给对应的以太坊节点。然后以太坊节点将区块记录到链上。

与以太坊不同的是，当一个节点收到区块后并不会马上记录到链上。而是等 Raft 网络中的 leader 收到所有 follower 的确认回执后，广播一个执行的消息。然后所有收到执行消息的 follower 才会将区块记录在本地的链上。


## **Lifecycle of a Transaction**
让我们来看看一个完整的 TX 流是怎样的：
1. 客户端发起一笔 TX 并通过 RPC 来呼叫节点。
2. 节点通过以太坊的 P2P 协议将节点广播给网络。
3. 当前的 Raft leader 对应的以太坊节点收到了 TX 后将 TX 打包成区块。
4. 区块被 RLP 编码后传递给对应的 Raft leader 节点。
5. leader 收到区块后通过 Raft 算法将区块传递给 follower。这包括如下步骤：
    1. leader 发送 `AppendEntries` 指令给 follower。
    1. follower 收到这个包含区块信息的指令后，返回确认回执给 leader。
    1. leader 收到不少于指定数量的确认回执后，发送确认 append 的指令给 follower。
    1. follower 收到确认 append 的指令后将区块信息记录到本地的 Raft log 上。
6. Raft 节点将区块传递给对应的 Quorum 节点。Quorum 节点校验区块的合法性，如果合法则记录到本地链上。

## **Block Race**




## **Speculative Minting**
一个区块从被创建，到经过 Raft 同步，到最后记录到链上会经历一段时间（尽管非常短）。如果等上一个区块写入到链上以后下一个区块才能生成，那么就会使得 TX 的确认时间增长。为了解决这个问题，同时为了能更有效率的处理区块生成，Quorum 推出了 `speculative minting` 机制。在这种机制下，新区块可以在其父区块没有完全上链的情况下被创建。如果这个场景重复出现，那么就会出现一串未被上链的区块，这些区块都会有指向其父区块的索引，我们成这类区块串为 `speculative chain`。