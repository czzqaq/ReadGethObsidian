# disv5

## kademlia


Discv5 用的路由表和Kademlia 很接近。而 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) 是一个分布式的迭代查询，用平均 log N 次能找到到一个节点的路由，而每个节点记录的路由表的个数却可以很少，总共最多 160（哈希的位数）个桶，每个桶大概装最多20个节点的路由。


## discv4

在以太网上，用的更多的反而是上一个版本。它的相关教程要多一些。
这几乎就是 kademlia 套壳，一样的节点发现过程，一样的路由表。增加了一些对报文格式的规定，还有ping 方式的 bonding(用来防止放大攻击)。详见： [discv4](https://raw.githubusercontent.com/ethereum/devp2p/master/discv4.md) 

## bootstrap node
以太网等官网提供的固定 ENR 信息的节点，用来做初始化，一个新加入的节点会从bootstrap node 开始，通过 discv4(5) 的find node 类协议构建 k-bucket 路由表。


## 作用
**为什么要有kademlia**：当一个新的 ethereum 节点开始运行时，它需要找到网络上的其他矿工/validator,.. 此时，它就需要这个discovery 协议了。这个“discover 其他矿工” 的过程，通过 DHT, distributed hash table 来组织。像 kademlia 里的 K-Bucket 就是其中一种。总之，路由表就是一个以distination node 为键值的 hash table，但是peer 太多了，所以我们需要一种把它distribute 的方法。

**为什么有 discv5:** 
我们知道 discv4 基本就是 kademlia 套皮。
有一个难点是，节点在网络上，其实可以发现非常大量的peer，它们的确都支持discv4 协议，不过它们不一定运行着我的application（比如eth 主网和测试网就是两个完全不同的application）。不运行同样应用的节点，在discv 4 上的组网是冗余的，要去除它，找到真正的在 app layer 上有效的peer，需要依次find node,逐个询问。
这显然不够高效，discv5 引入了topic 的概念，做更高效的 service discovery






## 实现细节
### 4 layer
根据 [rust-discv5手册](https://docs.rs/discv5/latest/discv5/) 。
- socket 使用UDP socket 来实现 transportation layer
- handler  因为 UDP 是无连接的，在这层进行 session 的管理和会话加密。使用 AES-GCM 加密
- service  路由表控制，service discovery
- Discv5  应用层。具体的dht 和 ENR 的 API


### Actors
- Advertiser：the nodes running the specific application
- registrar: The nodes holding the registry 
- Searcher: The nodes who ask registrar what's the advertisers are.

三者都是节点。

### registrar 的选择
#### built in STORE and GET

根据topic的哈希确定一个离该hash 最近的node id，这个对应的 node 便拥有了registrar 的角色。总之规定了一个topic hash 后，任何一个 searcher 和 advertiser 都能定位到这个节点。通过 STORE 和 GET 命令，做registry 表项的添加和获取。

这个过程很简单，整个还是 kademlia 式的(所谓built-in),效率高。问题在于load balance 很差，一个topic，无论多大，都只有一个node。
 
#### random placement
随机选几个node，每个node 都可能记录不止一种 topic，每个registrar 都是平行的。
这种方案的 load balance 肯定很好，因为分布式，所以灾备的安全性更好。缺点是速度慢，因为你并不知道储存合适topic 的node id 是什么，于是无法做log n 复杂度的查找。另外，对unpopular topic 的支持也不好，unpopular topic 的search 时间会更长。

#### Discv5的方案
简单来说，就是 topic hash 确定一个位置，对同一个 topic 可能有若干 registrar，选择方式为，离topic hash 越近，越容易被选上。
advertiser 通过逻辑去在 topic hash 相关的bucket table 中，随机选择 peer 作为topic 的registrar。

#### 防止攻击者，waittime
register 发送的内容有 topic 和 node id。另外还包括了 ip, port, 
为了防止恶意攻击者占用 registrar 资源，同一个 node id 最多向同一个 registar 注册 n 个 topic。
每项注册都有 expired time。
每个 advertiser 都有申请的waitime，指一个register提交到生效的等待时间。这个时间不固定，和 advertiser 已经占用的资源、register 内的资源使用量、以及 提供的 topic 的diversity 有关（是registrar 没见过的topic，就给短的waittime)
![[Pasted image 20250922152131.png]]

![[Pasted image 20250922152459.png]]

![[Pasted image 20250922152902.png]]



### lookup
类似于 bootstrap 的过程，一个 searcher 去registar 中获得足够多的 peer under the topic 
![[Pasted image 20250922150450.png]]



## 资料
 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) kademlia 算是一个前置知识。
 [discv4](https://raw.githubusercontent.com/ethereum/devp2p/master/discv4.md) 前一个版本
这个[youtube 教程](https://www.youtube.com/watch?v=o17ly2hej9w ) ，似乎是一个学者详细的介绍 discv5。
[Discovery Overview](https://github.com/ethereum/devp2p/wiki/Discovery-Overview) 开发者笔记，讲讲 eth 的p2p 协议现在的问题和概念。


# 在代码中

discovery.go 中最主要的函数是 DiscoveryProcess。该函数负责把 Ethereum DiscV5 (UDP-based) 的节点发现结果转换成 LibP2P 的连接信息，其中 DiscV5 就是上面写的。

下面是一些重点代码，总之仅仅是 dv5Udp 和 LibP2P.peerstore 两个库的使用。peer store 的说明见：[[host.go#peerstore]]

```go
    filter := FilterEnodes(log, cfg)
    // We pull nodes from discv5 DHT in random order to find new peers.
    // Eventually we'll find a peer record that matches our filter.
    randomNodeIter := n.dv5Udp.RandomNodes()
    randomNodeIter = enode.Filter(randomNodeIter, filter)
    
    
    pstore := n.Host().Peerstore()
    
    randomNodeIter.Next()
    found := randomNodeIter.Node()
    eps.SetPeerMetadata(info.ID, store.PeerMetadata{
                    ENR:       found.String(),
                    OPStackID: dat.chainID,
                })
   
   info, pub, err := enrToAddrInfo(found)
   pstore.AddAddrs(info.ID, info.Addrs, discoveredAddrTTL)
```

