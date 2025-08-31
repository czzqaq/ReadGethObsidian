这部分来讲L2 architecture，对应[这部分](https://specs.optimism.io/protocol/overview.html#optimism-overview)

# L2 node 职责
![[Pasted image 20250423081253.png]]
## Rollup Node
定义 Derivation：
```
derive_rollup_chain(l1_blockchain) -> rollup_blockchain
```
执行这个操作的叫 Rollup node。

### deposits 处理
用户直接和Op 交互的deposit 会通过 `OptimismPortal` L1 contract，之后被Rollup Node 收集。
L1 上的存款自然也会自动被Rollup Node 收集。

### Execution Engine
实际区块的产生调用了 Execution Engine，这是geth 执行程序的包装，用来执行交易，影响区块状态。

## 交互对象
### L1 合约
包括特定的L 1 合约，比如Optimism Portal，这是 withdraw 的入口，同时也用来读取L1 上其他合约提供的配置信息。
看起来，L2Node 不会直接改写L1 合约的状态，它只是读取L1 上的交易信息（从Batch Inbox Address）和OptimismPortal 上提供的状态。

### L2 合约
包括L2 合约：
![[Pasted image 20250423082149.png]]


# L2 合约
很难对这些合约的功能概括。它们提供了非常广泛的功能，比如对L1 状态反应，比如和 L1 之间做消息传递。
这些合约都用solidity 写成，和 L2 的用户交互，被L2 node 运行。
## bridge
用来和 L1 提供交互。比如 [Messengers](https://specs.optimism.io/protocol/messengers.html#cross-domain-messengers) 为L2 开发者提供了从L2 向L1 直接发消息的能力。比如在L2 上 ERC20 withdraw 的模拟，就是先在L2 上burn 掉，然后把消息再给L1. 见：[On L2](https://specs.optimism.io/protocol/withdrawals.html#on-l2)

