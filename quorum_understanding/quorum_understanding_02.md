# **Quorum的共识机制**
截止当前（2017年12月5日），Quorum前后用过两套共识算法：**QuorumChain Consensus** 和 **Raft-Based Consensus**。QuorumChain 是一种基于投票机制的共识算法，其利用以太坊的智能合约来实现投票和共识。Raft-based 则是在 Raft Consensus 算法基础上做了针对 Quorum 的修正。在 Quorum 1.2 之前的所有 Release 版本都采用了 QuorumChain。从 2.0 版本开始，Quorum 废弃了 QuorumChain 转而支持 Raft-based Consensus。在这里我们任然会介绍 QuorumChain 的共识方式，以方便大家了解和比较。


# **QuorumChain Consensus**
QuorumChain Consensus 是一个基于投票的共识算法。其主要特点有： 
 - 通过智能合约来实现和管理共识。
 - 用以太坊Transaction的形式来完成网络上投票的动作。
 - 用以太坊的签名校验机制来校验来自`Maker`和`Voter`节点的签名。

在 QuorumChain 中，有三种身份 `Maker`, `Voter`, `Observer`。身份有 **Maker** 的节点有权利打包交易并生成区块。其他节点收到区块后会查看区块头里的 Maker 签名，校验生成此区块的节点是否拥有 Maker 身份。拥有 **Voter** 身份的节点可以为收到的区块投票。一个区块只有收到一定数量的投票后才能被所有节点校验通过。**Observer** 身份没有任何特殊的权限，只能做一个记录区块的节点。

一个节点可以同时拥有 **Maker** 和 **Voter** 身份。

## **Voting Smart Contract**
QuorumChain 是由一个 Solidity 实现的 `BlockVoting` 智能合约实现的（[合约源码](https://github.com/jpmorganchase/quorum/blob/3a9ef674b76acf981da26739c7795e6e8accd489/core/quorum/block_voting.sol)）。在 Quorum 客户端被创建的时候，这个合约会被编译并存储到创世区块中。如果想修改投票的机制，则需要在 Quorum 客户端启动前重新编译新的 Solidity 代码。在[BlockVoting](https://github.com/jpmorganchase/quorum/blob/3a9ef674b76acf981da26739c7795e6e8accd489/core/quorum/block_voting.sol)中，合约提供了 `addVoter`, `removeVoter`, `addBlockMaker`, `removeBlockMaker` 等方法来增减 Voter 和 Maker。

当一个节点收到新的区块时，节点就会呼叫 Voting 合约，来确认上一轮区块的投票结果。这个结果将决定这个新区块将会连接到哪个区块上。

### **Maker Nodes**
Maker 节点主要用来打包交易并生成区块。所有 Maker 节点的地址信息都会被注册在 BlockVoting 合约中。在 BlockVoting 合约中，必须至少存在一个注册在案的 Maker。在 Quorum 刚搭建的时候，Maker 节点的信息是被预设置在 `genesis.json` 中的。但是在网络运行过程中，可以通过 BlockVoting 合约中的特定方法来增删 Maker 节点。

Maker 节点同时也可以作为 Voter 节点存在。

### **Voter Nodes**
Voter 节点的主要职责是给新生成的合法区块投票。和 Maker 一样，Voter 也会在网络启动的时候根据 `genesis.json` 中的预设配置进行初始化。同样，在网络运行过程中 Voter 可以通过 BlockVoting 合约来增删。Voter 节点可以给同一区块链深度的区块投票。在某一深度上，得票最多的区块就会成为链上该位置的区块并被整个网络共识。（关于这点会在[Block Voting](#topic_consensus_blockvoting)章节中提到）。

## **Block Creation**
在 QuorumChain 中，可以同时存在多个 Maker 节点。每个节点都会维护一份自己的随机时间。当这个随机时间到了以后，且这个 Maker 没有收到其他 Maker 生成区块的消息，那么它就会打包交易并生成区块。一旦生产区块这个动作开始，Maker 就会向网络上广播，告诉别的节点自己已经开始产块了。同时，刷新自己的随机时间，等待下一轮区块生成。对于别的节点，一旦收到这个有节点开始产块的消息，就立刻刷新自己的随机时间，等待时间读秒结束后开始产块。

当一个 Maker 节点准备生成新的区块时，它会校验本地的链上的最后一个区块的合法性。通过调用 BlockVoting 合约中的方法得到上一轮投票中获得投票最多的区块的HASH，然后对比这个 HASH 和本地最后一个区块的 HASH 是否有区别。如果两者一致，则表示本地的链没有问题，然后就可以把新的区块连接到本地的链上并广播出去。如果两者不一致，则本地的链不对。本地的最后一个区块并不是上一轮获得 Voter 投票最多的区块。生成区块的步骤将被终止。步骤可以参考下图：

![block_creation](https://github.com/heeeeeng/my_docs/blob/master/quorum_understanding/img/02_01.png?raw=true)


<a name="topic_consensus_blockvoting" ></a>

## **Block Voting**
在 QuorumChain 中，每一个周期区块的共识都包括：`区块生成 -> 区块广播 -> 区块校验 -> 区块投票` 这样一个流程。根据上面提到的区块生成机制，虽然可能性很小，但是还是会有一定几率出现两个 Maker 同时生成区块。为了解决这种情况，QuorumChain 推出了投票的机制。假设当前是第 n 轮区块共识过程。身份为 Voter 的节点收到新的区块后会首先校验区块的内容。如果校验成功，就会通过 BlockVoting 合约来投票给这个区块。如果本轮又来了其他区块，则不会对后续区块进行投票。当收到第 n+1 轮的区块时，节点就会通过 BlockVoting 合约来获得第 n 轮的区块投票结果，从而确定第 n 轮真正被全网大多数 Voter 接受的区块。

### **一个周期的区块共识过程**
![block_voting](https://github.com/heeeeeng/my_docs/blob/master/quorum_understanding/img/02_02.png?raw=true)

在一个周期中：
1. Maker 节点的随机时间读秒结束并没有收到别的 Maker 生成区块的消息，这个节点开始生产区块。同时，这个节点会将上一个周期的投票信息TX也打包至区块中。
2. 新生成的区块通过以太坊的 P2P 协议被广播到全网。所有的任意身份的节点都能收到这个区块。
3. Voter 节点收到这个区块后会校验区块的内容。这个校验步骤包括：
    1. 通过 `BlockVoting` 合约查看新区块的生产者是否在 Maker 列表中。
    1. 执行新区块中的所有的 TX。
    1. 通过 `BlockVoting` 合约查看新区块指向的父区块是否得到了足够的 voter 投票。
    1. 计算新区块中的所有TX生成的 Hash 值，对比和新区块中的 Hash 值是否一致。
4. 校验通过后，Voter 节点通过 `BlockVoting` 合约将其投票投给这个新区块，并将这次投票的 TX 广播给全网。这个 TX 只有在下次区块生成时才会被打包进区块。
5. 某个 Maker 节点随机时间读秒结束，准备开始生成新的区块。进入下一个周期。


# **Raft-based Consensus**
在2017年11月30日，Quorum 推出了 2.0 的 release 版本。这一版最大的改动就是剔除了 QuorumChain的共识方式，只支持 Raft-based Consensus。相比较以太坊的POW，Raft-based 提供了更快更高效的区块生成方式。相比 QuorumChain，Raft-based 不会产生空的区块，而且在区块的生成上比前者更有效率。

在这一章节，我们重点关注 Raft 是如何被运用到 Quorum 上的。想要了解 Raft 算法的技术细节，可以参考 [raft.github.io](https://raft.github.io/) 或者 [Raft 动态演示](http://thesecretlivesofdata.com/raft/)。

## **Raft on Quorum**
Quorum 的 Raft 代码主要沿用了 [etcd's Raft implementation](https://github.com/coreos/etcd/tree/master/raft)。

以太坊和 Raft 都有节点这一概念。在 Raft 中，节点可以有 leader, follower, condidate 三种身份。leader 负责生产区块。follower 监听 leader 的心跳并收取 leader 传递过来的区块。follower 一段时间没有收到来自 leader 的心跳后，就转变身份为 candidate，同时整个 Raft 进入下一个周期。然后 candidate 向所有节点发送投票的请求，其他节点（leader 收到消息后 leader 身份就被收回）收到请求后给予应答。如果 candidate 收到超过半数的应答，则晋升为 leader。一个节点一个周期只能投一次票。

在以太坊中，任意节点都可以作为区块的打包者，只要其在一轮 pow 中胜出。我们知道 Quorum 的节点沿用了以太坊的设计和代码。所以为了连接以太坊节点和 Raft 共识，Quorum 采用了网络节点和 Raft 节点一对一的方式来实现 Raft-based 共识。当一笔 TX 诞生后，TX 会在以太坊的 P2P 网络中传输。同时，Raft 的 leader 竞选一直在同步进行。当当前 leader 节点对应的以太坊节点收到 TX 时，以太坊节点就会将 TX 打包成区块并将区块通过 Raft 节点发送给 Raft 网络上的 follower。follower 节点收到区块后将区块交给对应的以太坊节点。然后以太坊节点将区块记录到链上。

与以太坊不同的是，当一个节点收到区块后并不会马上记录到链上。而是等 Raft 网络中的 leader 收到所有 follower 的确认收到区块的回执后，广播一个执行的消息。然后所有收到执行消息的 follower 才会将区块记录在本地的链上。


## **Lifecycle of a Transaction**

## **Block Race**

## **Speculative Minting**




