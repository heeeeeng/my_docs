# **Quorum是什么？**
Quorum 是由 J.P.Morgan 推出的企业级分布式账本平台。在以太坊的基础上，Quorum额外提供了联盟链的服务。在公有链方面，Quorum继承了以太坊的协议及其客户端Geth。

Quorum 和以太坊的主要区别：
- 提供了Transaction和Contract的私有化功能。
- 多种基于投票机制的共识方式。
- 网络与节点的权限管理。
- 更高的性能。

Quorum 的主要组件：
- Quorum Node (节点)
- Constellation - Transaction Manager (用于私有Transaction的管理)
- Constellation - Enclave (用于加解密私有Transaction的信息)

# **Quorum的结构**
![quorum_architecture](https://github.com/jpmorganchase/quorum-docs/raw/master/images/Quorum%20Architecture.JPG)

## **Quorum Node**
Quorum 节点的设计主要沿袭以太坊的 geth。面对日益壮大的以太坊社区，Quorum 希望能借助以太坊越来越完善的节点设计。因此，未来每次 geth 的 release 版本 Quorum 都会针对性的做升级调整。

为了适配其企业级的联盟链功能，Quorum 同时还对 geth 做了部分调整：
1. 用其自己实现的基于投票机制的共识方式 "QuorumChain" 来代替原来的 "Proof of work" 。
1. 在原来无限制的P2P传输方式上增加了权限功能。使得P2P传输只能在互相允许的节点间传输。
1. 原来区块中的 "global state root" 被替换成了 "global public state root"。
1. 原来的 state 存储被分成了两部分，分别存储 public state 和 private state。
1. 修改区块校验逻辑使其能支持 private transaction。
1. Transaction 生成时支持 transaction 内容的替换。这个调整是为了能支持联盟中的私有交易。（后面的 [Transaction Processing](#transaction_processing) 章节会提到）


## **Constellation**
Constellation 模块的主要职责是支持 private transaction。Constellation 由两部分组成：Transaction Manager 和 Enclave。Transaction Manager 用来管理和传递私有消息，Enclave 用来对私有消息的加解密。

### Transaction Manager 
在一次私有交易中，Transaction Manager 会存储私有交易的内容，并且会将这条私有交易内容与其他相关的 Transaction Manager 进行交互。同时它也会利用 Enclave 来加密或解密其收到的私有交易。

### Enclave
在分布式账本中，密码学被广泛的运用在交易真实性校验，成员校验，历史信息追溯等方面。为了能更有效率的处理消息的加密与解密，Quorum 将这个功能单独拉出并命名为 Enclave 模块。Enclave 和 Transaction Manager 是一对一的关系。

# **Transaction**
在 Quorum 中有两种交易类型，"Public Transaction" 和 "Privat Transaction"。在实际的交易中，这两种类型都采用了以太坊的 Transaction 模型，但是又做了部分修改。Quorum 在原有的以太坊 tx 模型基础上添加了一个新的 "privateFor" 字段。同时，针对一个 tx 类型的对象添加了一个新的方法 "IsPrivate"。用 "IsPrivate" 方法来判断 tx 是 public 还是 private，用 "privateFor" 来记录 tx 只有谁能查看。


### Public Transaction
Public Transaction 的机理和以太坊一致。TX 中的交易内容能被链上的所有人访问到。

### Private Transaction
Private Transaction 虽然被叫做 "Private"，但是在全网上也会出现与其相关的交易。只不过交易的明细只有与此交易有关系的成员才能访问到。在全网上看到的交易内容是一段hash值，当你是交易的相关人员时，你就能利用这个hash值，然后通过 Transaction Manager 和 Enclave 来获得这比交易的正确内容。这在 [Transaction Processing](#transaction_processing) 章节中会详细介绍。

<a name="transaction_processing" ></a>

## **Transaction Processing**
Public Transaction的处理流程和以太坊的 TX 流程一致。TX 广播全网后，被矿工打包到区块中。节点收到区块并校验区块中的 TX 信息。然后根据 TX 信息更新本地的 State。

Private Transaction也会将 TX 广播至全网。但是它的 TX payload已经从原来的真实内容替换为一个hash值。这个hash值是由`Transaction Manager`提供的。

<a name="trasaction_processing_img01"></a>

两者的区别可以参考下图：
![transactions](https://github.com/heeeeeng/my_img/blob/master/quorum/Tx.png?raw=true)

在以太坊中，每个节点都会维护一份本地的 StateDB 来快速的查询一些信息。所有节点中的 StateDB 都会进行共识从而保证大家的数据一致。在Quorum中，因为Private TX的存在，这种设计就会出现问题。由于只有部分节点能接触真实的数据，那就导致了这部分节点的StateDB内容和其他节点会存在差异，最终无法达成共识。为了解决这个问题，Quorum将原来的StateDB分成两部分，**Public State** 和 **Private State**。Public State会进行全网的共识，Private State则只记录 Private Transaction 相关的信息。

### **Private Transaction Process Flow**
Quorum中一个Private Transaction的详细流程可以参考下图：
![private_tx_flow](https://github.com/jpmorganchase/quorum-docs/raw/master/images/QuorumTransactionProcessing.JPG)

在这个例子中，有一笔`Transaction AB`跟联盟A和联盟B相关，与联盟C的节点无关。

> 注：每个Party都有自己的公钥和私钥，Party内所有成员都有这对公私钥。

1. DAPP 将 TX 发送给PartyA的节点。节点收到 TX 后将上文提到的 `privateFor` 字段的值设置为包含PartyA和PartyB的public key的数组：`["public_key_A", "public_key_B"]`。
2. 节点将 TX 发送给其对应的 Transaction Manager。
3. Transaction Manager 呼叫与其关联的 Enclave，并要求 Enclave 加密这笔 TX。
4. PartyA 的 Enclave 校验获取到的PartyA私钥，如果确认通过则进行如下动作：\
    i. 生成一个密钥（symmetric key）。\
    ii. 用上一步生成的symmetric key来加密 TX 的内容。\
    iii. 用SHA3-512来获取加密后的TX内容的hash值。\
    iv. 将 `i` 生成的symmetric key用**第一步**中的public key数组的所有值加密，然后生成一个新的数组。新的数组的每个元素都是由 `i` 中的symmetric key用原来数组的public key加密生成：`["key_encrypted_by_publickey_A", "key_encrypted_by_publickey_B"]` \
    v. 将 `ii` 生成的加密TX，`iii` 生成的hash值，`iv` 生成的加密后的数组返回给Transaction Manager。

5. PartA的Transaction Manager会把加密后的TX以及加密后的symmetric key保存到本地，并用从 Enclave 中获取的 hash 值作为索引。另外Transaction Manager会把hash值，加密后的TX，public_key_B加密的symmetric key这三项通过HTTPS发送给PartyB的Transaction Manager。PartyB的Tx Manager收到数据后，同样将加密后的TX和symmetric key保存到本地，并用收到的hash值作为索引。处理完后，PartyB的TX manager发送一个成功的回执给PartyA的TX manager。

6. PartyA的TX Manager收到成功回执后，将hash值返回给其对应的Quorum节点。节点收到hash值后，用这个hash值来替换原来TX的交易内容。（参考 [Transaction Processing 章节的第一张图](#trasaction_processing_img01) ）同时，将TX的 `V` 值设置为 37 或者 38。37或38就是Private Transaction的标识。其他节点查询后发现 `V` 的值为37或38时，就会认定其为Private Transaction。 

7. TX内容被替换后，TX就和Pbulic Transaction一样被节点通过P2P方式广播给整个网络。

8. 这条TX被某个区块收录到区块信息中。

9. 节点收到带这个TX的区块后，发现这个TX的 `V` 值为37或38。然后这个TX就被认定为Private Transaction，并将此TX的内容（也就是替换后的hash值）传给节点对应的Transaction Manager。

10. 因为PartyC的节点不在这个Private TX的范围内，所以其TX Manager无法在本地通过这个hash值找到对应的TX内容和symmetric key。然后TX Manager就会返回其节点一个 `NotARecipient` 回执。PartyC的节点收到这个回执后就不会更新其本地的Private State。对于PartyB的节点，其TX Manager通过这个hash值找到了本地存储的TX内容和symmetric key，但是由于这两个东西是被加密存储的，所以TX Manager将TX内容和symmetric key发送给其对应的 Enclave 进行解密。

11. PartyB的Enclave收到TX Manger发来的数据后，用PartyB的私钥Private Key来解密symmetric key。然后用解密后的symmetric key来解密TX的内容。解密完成后将正确的TX内容返回给TX Manager。

12. TX Manager收到解密的TX后通过EVM执行TX里面的内容。执行完成后将执行结果返回给Quorum节点，并更新Quorum节点的Private State。


# **QuorumChain Consensus**

## **Voting Smart Contract**

## **Maker**

## **Voter**

## **Observer**

## **Block Creation**

## **Block Voting**

## **Consensus Process Flow**

# **Raft Consensus**
// TODO

# **Security & Network Permissioning**
// TODO


