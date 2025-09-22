# gossip Overview
 
```ccard
type: folder_brief_live
```
 

# 阅读建议
[[gossip.go]] 和 [[discovery.go]] 中分别是两个协议的实现说明，基础知识。
[[layer2/读optimism 源码/op-node/p2p/node.go]] 阅读源码的入口。

这章的核心其实是下面两个库：
libp2p：peerstore, gossip，链接建立，p2p链接管理 .[libp2p](https://libp2p.io/)
ethereum devp2p: discover.UDPv5  [go-ethereum/p2p](https://pkg.go.dev/github.com/ethereum/go-ethereum/p2p)



