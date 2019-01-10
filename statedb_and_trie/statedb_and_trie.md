# **以太坊的StateDB和Trie**

在以太坊中，所有和账户相关的状态信息都是通过 StateDB 来存储和获取的。StateDB 作为表层和其他逻辑模块交互，在 StateDB 之后使用 Merkle Patricia Trie (MPT) 结构来构建编码后的 state 关系，用于快速索引以及回滚等操作。MPT 中的所有节点最后都会以 `key - value` 的形式存入磁盘数据库。

这篇文章将重点介绍 StateDB 和 Trie 在以太坊中的实现以及两者在账户状态存储流程中所扮演的角色。

## **Merkle Patricia Trie**

MPT 是结合了 Merkle Tree 和 Patricia Tree 的特点后创建的树形数据结构。其包含了如下的一些特点：
- 能存储任意长度的键值对数据。
- 支持 Merkle Proof，用于节点的快速校验。
- 能快速的查询 key 所对应的 value 数据。

<!-- ![01](https://github.com/heeeeeng/my_docs/blob/master/statedb_and_trie/mpt.png?raw=true) -->

在以太坊中，MPT被定义为四种不同类型的节点：`fullNode, shortNode, valueNode, hashNode`：
```go
type (
    fullNode struct {
        Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
        lags    nodeFlag
    }
    shortNode struct {
        Key   []byte
        Val   node
        flags nodeFlag
    }
    hashNode  []byte
    valueNode []byte
)
```
`valueNode` 存储具体的 value 数据，它的 key 是从 root 到此节点的路径上所有 key 的总和。

`hashNode` 存储一个数据库中其他节点的哈希用作索引。

`shortNode` 是 MPT 的枝干节点之一。Key 字段存储当前 shortNode 之后所有 node 共同的一段前缀 key。比如有两个数据它们的 Key 分别是

一棵 MPT 树由 fullNode 和 shortNode 来构成主体枝干。

先来看 shortNode。`Key` 字段表示


## **StateDB**

StateDB 作为账户状态的更新以及查询入口，基于具体逻辑的方法调用。比如账户余额的更新，nonce 的查询等。同时，它还肩负着所有合约数据的存储查询。为了支持数据的快速查询以及区块的回滚操作，StateDB 使用 MPT 结构作为其下层的存储方式。

先来看看 StateDB 的数据结构：
```go
type StateDB struct {
    db   Database
    trie Trie

    // This map holds 'live' objects, which will get modified while processing a state transition.
    stateObjects      map[common.Address]*stateObject
    stateObjectsDirty map[common.Address]struct{}

    // 其余字段暂时省去
    ...
}
```

- `db` - 用于连接下层 trie 数据库的字段。本身不存储数据，为了调取 TrieDB 存在。
- `trie` - 当前所有账户信息构建的 trie 结构。
- `stateObjects` - 


