# **以太坊的 P2P 网络**

以太坊的p2p网络主要有两部分构成：节点之间互相连接用于传输数据的tcp网络和节点之间互相广播用于节点发现的udp网络。本篇文章将重点介绍用于节点发现的udp网络部分。

p.s. 主要针对以太坊源码的对应实现，相关的算法如kademlia DHT算法可以参考其他文章。

## **P2P整体结构**

在以太坊中，节点之间数据的传输是通过tcp来完成的。但是节点如何找到可以进行数据传输的节点？这就涉及到P2P的核心部分**节点发现**了。每个节点会维护一个table，table中会存储可供连接的其他节点的地址。这个table通过基于udp的节点发现机制来保持更新和可用。当一个节点需要更多的TCP连接用于数据传输时，它就会从这个table中获取待连接的地址并建立TCP连接。

与节点建立连接的流程大致如下图：

<!-- ![01](https://github.com/heeeeeng/my_docs/blob/master/ethereum%20P2P/img/01%20p2p%20main%20struct.png?raw=true) -->

上图中，第一和第二步每隔一段时间就会重复一次。目的是保持当前节点的table所存储的url的可用性。当节点需要寻找新的节点用于tcp数据传输时，就从一直在更新维护的table中取出一定数量的url进行连接即可。

## **UDP部分主要概念**

前文提到过，以太坊会维护一个table结构用于存储发现的节点url，当需要连接新的节点时会从table中获取一定数量的url进行tcp连接。在这个过程中，table的更新和维护将会在独立的goroutine中进行。这部分涉及的代码主要在 `p2p/discvV5` 目录下的 `udp.go`, `net.go`, `table.go` 这三个文件中。下面我们看看udp这部分的主要逻辑结构：

<!-- ![02](https://github.com/heeeeeng/my_docs/blob/master/ethereum%20P2P/img/02%20udp%20part%20struct.png?raw=true) -->

### **readloop()**

`readloop()`方法在 udp.go 文件中，用来监听所有收到的udp消息。通过 `ReadFromUDP()` 接收收到的消息，然后在 `handlePacket()` 方法内将消息序列化后传递给后续的消息处理程序。

### **sendPacket()**

`sendPacket()` 用于将消息通过 udp 发送给目标地址。

### **UDP msg type**

所有通过 udp 发出的消息都分为两种类型：ping 和 pong。ping 作为通信的发起类型，pong 作为通信的应答类型。一个完整的通讯应该是：

1. Node A 向 Node B 发送 ping 类型的 udp 消息。
2. Node B 收到此消息后进行消息处理。
3. Node B 向 Node A 发送 pong 类型的消息作为之前收到的 ping 消息的应答。

### **loop()**

`loop()` 方法在 net.go 文件中，是整个 p2p 部分的核心方法。它控制了节点发现机制的大部分逻辑内容，由一个大的 select 进行各种 case 的监控。主体代码大致如下：

```go
func (net *Network) loop() {
    ...
    for {
        select {
        case pkt := <-net.read:
            // 处理收到的 udp 消息
        case timeout := <-net.timeout:
            // 发送的 udp ping 消息超时未收到 pong 应答
        case q := <-net.queryReq:
            // 从table中查找距离某个target最近的n个节点
        case f := <-net.tableOpReq:
            // 操作table, 这个case主要是为了防止table被异步操作
        case <-refreshTimer.C:
            // 计时器到点，刷新table里的url
        ...
        }
    }
    ...
}
```

### **nodeState**

`nodeState` 表示了一个url对应的状态。在 `net.go` 的 init() 方法中定义了各种 nodeState：

```go
type nodeState struct {
    name     string
    handle   func( ... ) ( ... )
    enter    func( ... )
    canQuery bool
}

var (
    unknown          *nodeState     // 新收到的未知节点
    verifyinit       *nodeState     // 处在初次验证的节点
    verifywait       *nodeState     // 正在等待应答的节点
    remoteverifywait *nodeState
    known            *nodeState     // 已经完成应答的节点，可以加入到table中
    contested        *nodeState
    unresponsive     *nodeState
)

func init() {
    unknown = &nodeState{
        ...
    }
    verifyinit = &nodeState{
        ...
    }
    ...
}
```

nodeState主要是为了记录新地址的可用状态。当节点接触新的 url 时，此 url 会首先被定义为 unknown 状态，然后进入后续 ping pong 验证阶段。验证流程过完以后 就会被定义为 known 状态然后保存到 table 中。

### **enter() 和 handle()**

nodeState 结构下定义了两个方法 `enter()` 和 `handle()`。

- enter 是 nodeState 的入口方法，当一个 url 从 nodeState1 转到 nodeState2 时，它就会调用 nodeState2 的 enter() 方法。
- handle 是 nodeState 的处理方法，当节点收到某个 url 的 udp 消息时，会调用此 url 当前 nodeState 的 handle() 方法。一般 handle() 方法会针对不同的 udp 消息类型进行不同的逻辑处理：

```go
// verifywait 的 handle 方法
func (net *Network, n *Node, ev nodeEvent, pkt *ingressPacket) (*nodeState, error) {
    switch ev {
        case pingPacket:
            net.handlePing(n, pkt)
            return verifywait, nil
        case pongPacket:
            err := net.handleKnownPong(n, pkt)
            return known, err
        case pongTimeout:
            return unknown, nil
        default:
            return verifywait, errInvalidEvent
    }
}
```

handle() 方法在处理后会返回一个 nodeState，意思是我当前的 nodeState 在处理了这个 udp 消息后下一步将要转换的新的 state。比如在上面的代码中，verifywait 状态下的 url 在收到 pong 类型的 udp 消息后就会转换成 known 状态。

## **具体的新节点加入流程**



## **lookup节点发现**



## **寻找closest节点**

