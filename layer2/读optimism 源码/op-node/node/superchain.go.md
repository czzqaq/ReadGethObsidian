# 什么是super chain

这是一个挺大的概念，先阅读 [官网手册的# Superchain ](https://docs.optimism.io/superchain/superchain-explainer)

Superchain 是由多个 L2 **链**（OP Chains）组成的网络，这些链基于统一的 OP Stack 构建。
这个概念的提出是为了彻底解决 scalability的问题。

它们之间共享 L1 区块链，共同的 bridge contract，消息通过跨链消息互通。

可以把superchain 当成是L2 链的集群，一条链可以通过 registry 加入superchain。使用 op-geth 的用户就可以自己创建一条链。注册、链通讯和链的管理均通过特定的 smart contract 完成，有些是L2的，不过主要都是L1 的协议。

## 在代码中

这段代码用来完成版本检查，如果版本不兼容，就返回错误。主体是 engine_signalSuperchainV1 的RPC。

这个signal 的作用是通知L2 chain 做版本更新，走的L2 协议。见 [Superchain upgrades](https://docs.optimism.io/superchain/superchain-upgrades)


