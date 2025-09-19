# disv5

## kademlia


Discv5 用的路由表和Kademlia 很接近。而 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) 是一个分布式的迭代查询，用平均 log N 次能找到到一个节点的路由，而每个节点记录的路由表的个数却可以很少，总共最多 160（哈希的位数）个桶，每个桶大概装最多20个节点的路由。


## discv4

在以太网上，用的更多的反而是上一个版本。它的相关教程要多一些。




## 4 layer
根据 [rust-discv5手册](https://docs.rs/discv5/latest/discv5/) 。
- socket 使用UDP socket 来实现 transportation layer
- handler  因为 UDP 是无连接的，在这层进行 session 的管理和会话加密。使用 AES-GCM 加密
- service  路由表控制，peer discovery
- Discv5  应用层。具体的API 和 任务。




## 作用
当一个新的 ethereum 节点开始运行时，它需要找到网络上的其他矿工/validator,.. 此时，它就需要这个discovery 协议了。这个“discover 其他矿工” 的过程，通过 DHT, distributed hash table 来组织。像 kademlia 里的 K-Bucket 就是其中一种。总之，路由表就是一个以distination node 为键值的 hash table，但是peer 太多了，所以我们需要一种把它distribute 的方法。

另外一个关键是，节点在网络上，其实可以发现非常大量的peer，不过它们不一定运行着我的application，于是要有服务的协商。


## 资料
[# Discovery Overview](https://github.com/ethereum/devp2p/wiki/Discovery-Overview) 比较入门的读物，讲讲 eth 的p2p 协议。
 [kademlia 的路由表](https://zhuanlan.zhihu.com/p/40286711) kademlia 算是一个前置知识。
这个[youtube 教程](https://www.youtube.com/watch?v=o17ly2hej9w ) ，似乎是一个学者详细的介绍 discv5。






