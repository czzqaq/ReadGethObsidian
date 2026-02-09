# disv5

## 概念
### kademlia


Discv5 用的路由表和Kademlia 很接近。而 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) 是一个分布式的迭代查询，用平均 log N 次能找到到一个节点的路由，而每个节点记录的路由表的个数却可以很少，总共最多 160（哈希的位数）个桶，每个桶大概装最多20个节点的路由。

**为什么要有kademlia**：当一个新的 ethereum 节点开始运行时，它需要找到网络上的其他矿工/validator,.. 此时，它就需要这个discovery 协议了。这个“discover 其他矿工” 的过程，通过 DHT, distributed hash table 来组织。像 kademlia 里的 K-Bucket 就是其中一种。总之，路由表就是一个以distination node 为键值的 hash table，但是peer 太多了，所以我们需要一种把它distribute 的方法。

### discv4

在以太网上，用的更多的反而是上一个版本。它的相关教程要多一些。
这几乎就是 kademlia 套壳，一样的节点发现过程，一样的路由表。增加了一些对报文格式的规定，还有ping 方式的 bonding(用来防止放大攻击)。详见： [discv4](https://raw.githubusercontent.com/ethereum/devp2p/master/discv4.md) 

discv4 基本就是 kademlia 套皮。它有缺陷。
节点在网络上，其实可以发现非常大量的peer，它们的确都支持discv4 协议，不过它们不一定运行着我的application（比如eth 主网和测试网就是两个完全不同的application）。不运行同样应用的节点，在discv 4 上的组网是冗余的，要去除它，找到真正的在 app layer 上有效的peer，需要依次find node,逐个询问。
这显然不够高效，discv5 引入了topic 的概念，做更高效的 service discovery

### bootstrap node
以太网等官网提供的固定 ENR 信息的节点，用来做初始化，一个新加入的节点会从bootstrap node 开始，通过 discv4(5) 的find node 类协议构建 k-bucket 路由表。
node 就是一个p2p 的网络单元。discv 就是一个**节点**的发现协议。

### ENR 
它是从dicv4 中就有的概念，但在discv5中做了改进。discv5的版本见： https://eips.ethereum.org/EIPS/eip-778

ENR是一个包含节点信息的结构化数据记录，用KV 的格式组织起来必要的节点信息描述。

可以包含以下信息：
- 节点的公钥和签名，用来验证节点的身份和消息完整性。
- 节点的IP地址和端口号，支持选择 ipv6
- 序列号，每当节点更改其记录（例如 IP 变了），它必须增加这个数字并重新发布记录。其他节点通过比较 `seq` 的大小来确定谁是最新的。
- 节点的版本号，例如 "v4" 或 "v5"。
- 其他附加信息。

ENR 编码为 RLP，也可以用文字编码。这里不深入到编码细节。 

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


## 根据官方文档
### Rationale - 作用

#### Replace the Discovery v4 Endpoint Proof
Discovery v4 Endpoint Proof 的 Endpoint Proof 为了防止放大攻击，作用在 find node 和 ENRRequest 类型的packet 上。Endpoint Proof 的方法是验证其Sender 是否在12小时内发送过 Pong 包做 PING 的回应。
这个地方，如果 receiver 没有收到Pong，那么 Sender 其实是没有什么好方法来让它注意到自己的，毕竟得先有Ping，Sender 没有主动权。（其实我感觉Discv4这个真心反人类，凭直觉就觉得有问题）

#### Require knowledge of destination node ID for communication

Discv4 中，我只要知道你的 IP 地址，就可以给你发一个 `PING` 包。你收到后，**必须**回复一个包含你NodeID 的 `PONG` 包。这使得可以不通过 FINDNODE 就获得 node ID。

#### Support more than one node ID cryptosystem
这将允许除 `secp256k1/keccak256` 以外的身份加密系统。

####  Replace node information tuples with ENRs
过去的 ENR 里只有 【IP, UDP端口, TCP端口, NodeID】。Discv 5里有更多功能，必须支持字段拓展。

####  Guard against Kademlia implementation flaws
Discv4 过于信任对方，默认认为 FINDNODE 的回应正确。

#### Secondary topic-based node index
Discv4 只能根据 ID 找节点，假设网络里只有 1% 的节点运行特定的服务，在 Discv4 里，我只能随机乱撞，连上一个问一个，直到碰运气找到为止。

#### Change replay prevention
为了防止重放攻击，Discv4 使用了 expiration 来控制，如果 `接收时间 - 包内时间戳 > 20秒`，包就过期了。
这假设全世界电脑的时间都是准的。但很多用户的电脑时钟可能慢了几分钟，明明网络是好的，但就是连不上任何节点。
此外，这个防止重放攻击的方式也不完全，如果攻击者截获了一个有效的 `PING` 包，在过期时间（通常 20 秒）内可以多次发送给接收者。

#### Message obfuscation
Discv4 的数据包格式非常固定，头部是哈希，特征明显。这让以太坊协议很容易被针对性封锁。

#### 缺乏对 PONG 的过滤

节点可能会处理并未主动请求的 `NEIGHBORS` 或 `PONG` 包，或者轻信恶意节点提供的“邻居”信息，导致将恶意节点加入路由表。进而受到 NODES 重放攻击等影响。

## session
### 握手 (电子信封模型)

**核心目的**  
在不直接传输密钥的前提下，双方达成**Session Key (对称密钥)** 的共识，从“公开喊话”切换到高效安全的“私密频道”。

**握手四步走**

1. **试探 (Request)**：A 发起请求，试图建立连接。
2. **质疑 (WHOAREYOU)**：B 无法解密或不认识 A，回传一个 **Nonce** (Challenge)，潜台词：“证明你有私钥，且这不是录播”。
3. **投递信封 (Auth Response)**：**关键步骤**。
    - A 用私钥对 Nonce **签名**（证明身份）。
    - A 利用 **ECDH** 协商出的临时密钥，将签名打包**加密**（制作电子信封）。
4. **默契达成 (Ack)**：B 拆信验证成功。双方利用 **HKDF** 算法，在本地各自算出完全一致的 Session Keys。

**未来搜索关键词 (Keywords)**

- **WHOAREYOU Packet** (握手的触发点)
- **Challenge-Response** (Nonce 的作用：防重放)
- **ECDH** (Elliptic Curve Diffie-Hellman，不联网传输的密钥交换魔法)
- **HKDF** (Key Derivation Function，把共享秘密搅拌成 Session Key 的算法)
- **AES-GCM** (最终生成的 Session Key 所用的加密模式)

### 额外补充
- nonce -> session key 的生成一定不能重样，通过 nonce+salt（一个64位随机数）
- 每有一个 **Endpoint** (IP + Port)，确定一个session_key。
- LRU Cache 来存 session key。这称为 session 的缓存。
- WHOAREYOU 一定针对某个特定的请求（FINDNODE,PING, 等），它隐含条件：我不认识你，所以拒绝了你的请求。只要收到一个 WHOAREYOU，请求者需要忘记所有做过的请求，先从握手开始。

## Node Table
本质 K-bucket。下面只说修改点。
### 距离

距离：定义logdistance = log_2(XOR(id1,id2))，取值范围是 0~255，本质数第几个开始不一样，每个距离一个桶，所以是256个bucket。

### add

添加时，如果桶已满，PING 最久没见的，如果它挂了则踢出，否则扔掉新节点。这是 discv4 的逻辑，也是discv5 的。但是：
注意，这个过程不是每次都要做，否则无数的新节点会构成ddos 攻击，每个添加请求都带来一次PING。
实际的做法是：
- Asynchronous check
	 新增节点放入candidates，candidate 数量有限，且仅在空闲时异步地处理。
- 随机抽查
	 不是一直查最老的那个，而是没事干的时候随机PING，维护一个没有联系的 node 的列表。彻底把 PING 来维持表和新节点加入请求解绑。


另外，还有一个点是，不要因为一次PING 不通，就干掉某个已经很老的节点，可能只是网络波动。

### Lookup
就是标准的过程，和 Discv4 完全一致。这里补充一些文档上我不知道的细节：
#### 多路并发 (Disjoint Paths)
把初始的节点（就是从我知道的节点里选出来的那 \alpha 个最近的）分成几组（比如一个节点一组），走不同的路径去查。也就是记录每个节点迭代查询的过程。
注意，这个过程中，要时刻刻意保证 disjoint，也就是每个path 中的所有节点互不相同。

所有诚实的节点都会提供相似的路径，而路径完全不同的是恶意节点。

#### table 维护
另外的，诚实的节点也应该尽量保证提供的nodes 的真实性，定期PING ，保证回复的节点都是有效的。

#### 协议
```
A -> B: FINDNODE [d]

`A <- B: NODES [N₁, N₂, N₃]`  
`A <- B: NODES [N₄, N₅]`  
_(B 可能分多个包发回来)_
```

其中 d 是特定的一个 logdistance 的值，也即一个桶的标号。

如果 B 的回应里，node 数量不够，则发送：
```
A -> B: FINDNODE [d+1]
```

#### FINDNODE 

```
message-data = [request-id, [distance₁, distance₂, ..., distanceₙ]]
message-type = 0x03
distanceₙ    = requested log2 distance, a positive integer
```

- When distance `0` is requested, the result set should contain the recipient's current record.
#### NODE（协议）
```
message-data = [request-id, total, [ENR, ...]]
message-type = 0x04
total        = total number of responses to the request
```

- The recommended result limit for FINDNODE queries is 16 nodes. I.E. total should no more than 16.
- Multiple NODES messages may be sent as responses to a single query. 
- Once make enough(=total) node response, additional responses may be ignored.
- The recipient(sender of NODE) should verify that the nodes match the requested distances.

## Topic
### Topic Advertisement
Topic 指一类服务。一个节点可以关联多个或者0个Topic。Topic 是另一种节点索引的方式，把它想象成一个大广告牌。
当一个节点想要提供某种服务时，它就 Place an ad.

### Topic Table
针对某一个特定主题的广告列表，被称为 **Topic Queue**，它是一个 Queue。
每个节点都会存储多个 Topic Queue，这些组成一个 Topic table。

- 每个Topic queue 都要有一个上限，Topic table 也有一个总上限。
- 在同一个主题队列里，同一个节点只能出现一次。如果一个已经在排队的节点想再次注册同一个主题，会直接失败。

#### 寿命控制和 ticket
每个 queue 中的节点信息，都会有一个寿命 target-ad-lifetime。 target-ad-lifetime = 15 分钟。
显然，如果 Table 满了，等全表腾出一个空位，如果当前Topic queue 满了，等queue 空出来。如果不满，直接进。

为了做到这一点，使用 Ticket 控制注册速度。如果满了，就给你一个 ticket，告诉你需要等待多久。ticket 并没有严格的结构规范，但一般：Ticket = 加密(密钥, 随机数, \[谁领的, IP是多少, 啥主题, 领券时间, 需要等多久, 累计等待时间\])。等到时间到了，客户端拿着 ticket 重新向服务端申请 place ad。这个申请有一个窗口期，是在\[领券时间+等待时间~领券时间+等待时间+10s\] 的区间里


这个机制保证了无状态的等待控制，让服务端无需维护一个等待时间的队列。

#### 协议
下面仅简单介绍：

REGTOPIC
用来做一个 place an ad 请求。提供了topic、ENR。可以带 ticket 或者不带ticket 请求。总是被 TICKET 回应，还可能跟着一条 REGCONFIRMATION。

TICKET
REGTOPIC 的回应。提供一个ticket，详见 [[#寿命控制和 ticket]]。

[REGCONFIRMATION](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#regconfirmation-response-0x09)
表示 server 已经发布了REGTOPIC 请求的ad。

TOPICQUERY
指定一个topic，读取其 topic queue

## 资料
 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) kademlia 算是一个前置知识。
 [discv4](https://raw.githubusercontent.com/ethereum/devp2p/master/discv4.md) 前一个版本
这个[youtube 教程](https://www.youtube.com/watch?v=o17ly2hej9w ) ，似乎是一个学者详细的介绍 discv5。
[Discovery Overview](https://github.com/ethereum/devp2p/wiki/Discovery-Overview) 开发者笔记，讲讲 eth 的p2p 协议现在的问题和概念。
[devp2p的 discv5 规范](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-rationale.md)
https://github.com/sigp/discv5 这里是一个 rust 写出来的库。我将详细阅读它，见：[[rust-discv5]]
[官方disv5规范](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) 一个最好的discv5 规范文档和教程。


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

