# **Quorum 的 Raft-based Consensus**
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
4. 区块被 RLP 编码后传递给对应的 Raft leader。
5. leader 收到区块后通过 Raft 算法将区块传递给 follower。这包括如下步骤：
    1. leader 发送 `AppendEntries` 指令给 follower。
    1. follower 收到这个包含区块信息的指令后，返回确认回执给 leader。
    1. leader 收到不少于指定数量的确认回执后，发送确认 `append` 的指令给 follower。
    1. follower 收到确认 `append` 的指令后将区块信息记录到本地的 Raft log 上。
6. Raft 节点将区块传递给对应的 Quorum 节点。Quorum 节点校验区块的合法性，如果合法则记录到本地链上。


## **Block Race**
通常情况下，每一个被传至 Raft 的区块最终都会被添加到链上。但是也会有意外出现。比如因为一些网络的原因，某个 leader 无法与大部分的 follower 进行交互了。这时其他 follower 就会推选出新的 leader。在这期间，旧的 leader 还会产生新的区块。但是因为没有收到足量的 follower 回执，所以它产生的区块都不会最终写到链上。与之相对的，新的 leader 这边则会正常进行区块同步。一旦旧 leader 这边恢复通信，它会将自己产生的 `AppendEntries` 指令广播出去。由于其发出的指令已经过时了，所以大部分的 follower 不会给予这些指令正确的回执。

具体流程如下：
1. `Node1` 作为 leader 产生一个新的区块：`[0x002, parent: 0x001]`。这个区块的父块是编号为 0x001 的区块。`Node1` 通过 Raft 将这个区块进行共识。
2. 0X002 区块共识成功后网络出现了问题，`Node1` 无法与大部分的 follower 进行通信。
3. 网络问题并没有影响 `Node1` 的产块。一个新的区块被产出：`[0x003, parent: 0x002]`。为了共识这个新的区块，`Node1` 向 Raft 网络发送 `AppendEntries` 指令（指令中包含新区块的信息），并等待 follower 的确认回执。因为网络问题，`Node1` 一直没有收到足够数量的 follower 回执。
4. 于此同时，那些无法与 `Node1` 通信的 follower 因为长时间没收到 leader 的心跳，所以推选出了新的 leader：`Node2`。
5. `Node2` 产生区块`[0x004, parent: 0x002]` 后将含此区块信息的 `AppendEntries` 指令发送给 follower。follower 确认这个指令后返回确认回执。最终这个指令被执行并记录在 Raft log中。
6. 0x004 区块共识完成后网络状态得到恢复。此时第三步中的来自 `Node1` 的 `AppendEntries` 指令终于被传递给大部分的 follower。但是此时 follower 的链上的最终块已经是第五步中的 0x004，所以区块 `[0x003, parent: 0x002]` 无法被执行，因为其 parent 是 0x002 不满足当前链的状态。这条不执行的动作也会被记录到 Raft log 中去。
7. `Node2` 继续生成区块 `[0x005, parent: 0x004]`。
8. 最后整个流程下来，follower 的 Raft log 内容大致会长这样：
```
得到区块[0x002, parent: 0x001, sender: Node1] - 执行     
得到区块[0x004, parent: 0x002, sender: Node2] - 执行     
得到区块[0x003, parent: 0x002, sender: Node1] - 不执行     
得到区块[0x005, parent: 0x004, sender: Node2] - 执行   
```

需要注意的是，整个共识过程中，Raft 层面只负责记录自己节点的 Raft log。真正执行 log 内容的是 Quorum 节点。Quorum 节点根据其节点对应的 Raft log 来做具体的操作。


## **Speculative Minting**
一个区块从被创建，到经过 Raft 同步，到最后记录到链上多多少少会经历一段时间（尽管非常短）。如果等上一个区块写入到链上以后下一个区块才能生成，那么就会使得 TX 的确认时间增长。为了解决这个问题，同时为了能更有效率的处理区块生成，Quorum 推出了 `speculative minting` 机制。在这种机制下，新区块可以在其父区块没有完全上链的情况下被创建。如果这个场景重复出现，那么就会出现一串未被上链的区块，这些区块都会有指向其父区块的索引，我们将这类区块串称为 `speculative chain`。

在维护 speculative chain 的同时，系统还会维护一份被称作 `proposedTxes` 的数组。这份数组包含了所有 speculative chain 中的 TX。主要是为了记录已经被传输到 Raft 中但是还没被正式上链的交易。防止同一条交易被重复打包。

## **Conclusion**
与以太坊的共识机制相比较，Raft-based Consensus 除了效率上的不同外，最大的差异是前者保证的是最终一致性，而 Raft 保证的是强一致性。既每次产生的区块都要求全网保持共识。在这里我们只是粗粗的了解 Raft-based，还需要更多对源码的研究来拨开 Quorum 的面纱。

# END



