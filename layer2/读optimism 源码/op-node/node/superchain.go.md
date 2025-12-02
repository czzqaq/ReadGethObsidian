# 什么是super chain

## 概括

这是一个挺大的概念，先阅读 [官网手册的# Superchain ](https://docs.optimism.io/superchain/superchain-explainer)

Superchain 是由多个 L2 **链**（OP Chains）组成的网络，这些链基于统一的 OP Stack 构建。
这个概念的提出是为了彻底解决 scalability的问题。

它们之间共享 L1 区块链，共同的 bridge contract，消息通过跨链消息互通。

可以把superchain 当成是L2 链的集群，一条链可以通过 registry 加入superchain。使用 op-geth 的用户就可以自己创建一条链。注册、链通讯和链的管理均通过特定的 smart contract 完成，有些是L2的，不过主要都是L1 的协议。


## Superchain interoperability

见： https://docs.optimism.io/op-stack/interop/explainer

未来，它不再是简单的星型结构（所有人都只跟 L1 说话），而是网状结构（Mesh Network）

> Each blockchain in the Superchain interop cluster would have direct connections to every other blockchain... creating a fully connected mesh network

安全验证当然还是通过 L1，具体是由目标的L2链调用 **CrossL2Inbox** 合约。
# 在代码中

这段代码主体是 engine_signalSuperchainV1 的RPC。这个signal 的作用是向网络发出“请准备好升级”的信号，其过程 见 [Superchain upgrades](https://docs.optimism.io/superchain/superchain-upgrades)

1.  **[ProtocolVersion](https://specs.optimism.io/protocol/superchain-upgrades.html?utm_source=op-docs&utm_medium=docs#protocolversions-l1-contract)** L1 smart contract 改变了配置，CL 层感知到链版本升级。
2. 调用engine_signalSuperchainV1，通知网络做好升级准备。这是一个 L2 Execution layer 的RPC 服务，见 [engine_signalsuperchainv1](https://specs.optimism.io/protocol/exec-engine.html#engine_signalsuperchainv1)
3. 在本代码中，根据配置，做 halt，即暂时停止CL 层链的增长，防止链分叉。


## Superchain 本身
这里的 superchain.go 只是一个小服务，实际的功能大块儿在另一个模块，见：[[op-supervisor]]，它实现了 Superchain interoperability






