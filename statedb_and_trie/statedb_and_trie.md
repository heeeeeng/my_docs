# **以太坊的StateDB和Trie**

在以太坊中，所有和账户相关的状态信息都是通过 StateDB 来存储和获取的。StateDB 作为表层和其他逻辑模块交互，在 StateDB 之后使用 Merkle Patricia Trie (MPT) 结构来构建编码后的 state 关系，用于快速索引以及回滚等操作。MPT 中的所有节点最后都会以 `key - value` 的形式存入磁盘数据库。

这篇文章将重点介绍 StateDB 和 Trie 在以太坊中的实现以及两者在账户状态存储流程中所扮演的角色。

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

`db` - 用于连接下层 trie 数据库的字段。本身不存储数据，只是为了调取 TrieDB 存在



