# 说明

这里是全部 p2p 功能的集合，它把 **LibP2P**、**discv5**、**GossipSub**、**Req-Resp 同步**、**对等节点评分与封禁** 等杂糅在一起，对外再包一层易用接口，供上层逻辑（L2 区块同步、EL 协议消息传递等）使用。

## 全部功能
这里只列出主要功能的部分
### host
```go
n.host, err = setup.Host(log, bwc, metrics)
```
见 [[host.go]]

### Req-Resp 同步
```go
if setup.ReqRespSyncEnabled() && !elSyncEnabled {...}
```
见 [[sync.go]]

### gossip topic
```go
n.gs, err = NewGossipSub(...) 
n.gsOut, err = JoinGossip(...)
```

见 [[gossip.go]]

### discv5

```go
tcpPort, err := FindActiveTCPPort(n.host)
n.dv5Local, n.dv5Udp, err = setup.Discovery(...)
```
discv5（Discovery v5，也叫 “Ethereum Node Discovery Protocol v5”）是由以太坊社区设计并实现的 **基于 UDP 的分布式节点发现协议**。
见 [[discovery.go]]

